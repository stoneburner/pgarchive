# pgarchive

A PostgreSQL Archive Server providing fast and incremental Backup and Restore Services

## Release Status

This is a very early alpha software. You will need to change parameters and possibly code in a shell script to make this work for you.

It is used in production at emarsys.com to back up large (up to 1TB, 100GB WAL per day) databases and you may find it useful or at least inspirational.

## Features and Design Goals

* pgarchive runs an archive container service using local storage
    * more than just a one-shot backup and restore tool
    * can backup local and remote databases
    * works for whole DB clusters (like pg_basebackup, not like pg_dump)
    * incremental data storage
    * can run multiple containers for several distinct upstream databases on a single server
    * a database server be backed up to multiple pgarchive containers (e.g. at different sites)
    * CLI interface to create and manage containers, and assist in recovery
* avoinding complexity and reinventing the wheel
    * it's a single shell script, using common shell utilities
    * uses many of the built-in PostgreSQL tools, sometimes in novel ways
    * uses [btrfs] to store only the incremental difference between snapshots of a database cluster directory
    * uses cron for time based jobs
* user friendliness
    * it is intended to be usable by people who are not PostgreSQL experts for routine tasks
    * one command initialization and teardown of whole backup containers
    * supports restore with point in time recovery
    * includes maintenance tasks for time based expiry, compression, etc. via cron
    * keeps helpful logs
    * has a simple dashboard
    * comes with custom bash completion
* little impact on upstream database, continuous operation
    * after initial setup containers pull down data using only streaming replication
    * no load spikes or delays due to full dumps or rsyncs
    * does not require instrumentation of upstream DB except for a replication slot, which it creates and removes itself
    * the upstream DB does not need to (and should not!) run on btrfs
* fast (local) restore
    * as fast as activating a btrfs snapshot and doing postgres recovery to consistency
    * allow point in time recovery forward from any snapshot
    * locally restored and started database clusters run on btrfs, and will be slow
    * full disaster recovery is currently not supported, but easy to do and add in the future (rsync or [pg_basebackup] a local copy somewhere else)
* resilience
    * to outages of the upstream DB
    * network interruptions
    * continues where it stopped if interrupted
    * it is easy to monitor


## Caveats

* opinionated
    * uses standard tools in a unique way
    * presumes some upstream configuration parameters, like CSV logging, and an available streaming replication connection and slot
    * at the moment contains many hard coded configuration parameters in the script instead of in a configuratioin file
* requires PostgreSQL 9.4
    * due to its architecture and use of replication slots earlier versions are not easily supported
    * changes for 9.5 should be trivial and will be done soon
    * does not support cross-version backup&restore, containers must match upstream databases exactly
* requires btrfs (for containers), despite well known problems with PostgreSQL
    * btrfs is slow for random updates in large files
    * huge jitter in TPS and large CPU spikes
    * btrfs fragmentation issues cause even more performance loss over time, needs defragmentation
    * deemed acceptable for an archive system, as the throughput is good enough and snapshots are a huge benefit
* use at your own risk!


## Architecture

The basic components and data flow for a single container looks like this. Some details like automatic expiry and btrfs filesystem maintenance are not shown for clarity.

```
   +--------------------------+
   |  Upstream PostgreSQL DB  | <-------------------------------------------+
   |      may be remote       |                                             |
   |  not part of container   | -------------+                              |
   +--------------------------+              |                              |
        |                                    | pg_basebackup                |
        | streaming repl.                    | on container init only       |
        |                                    |                              |
        V                                    V                              |
+-------------------+                  +--------------------------+         |
| pg_receivexlog    | restore_command  | warm standby PG process  |         |
| /wal_archive      | ---------------> | /standby                 |         |
+-------------------+                  +--------------------------+         |
            |                                   |                           |
            |                                   | btrfs snapshot            |
            | restore_command                   | <cron>                    |
            |          |                        V                           |
            |  <user requests clone>   +--------------------------+         |
            |          |               | snapshot archive         |         |
            |     btrfs snapshot       | /snapshots               |         |
            |  +---------------------- | /snapshots/<timestamp>   |         |
            |  |                       | /snapshots/...           |         |
            |  |                       +--------------------------+         |
            |  |                                                            |
            V  V                                                            |
        +------------------------------------+        user does full or     |
        | user created clone                 |-+    partial DR on his own   |
        | (name, base snapshot, target time) | | ---------------------------+
        | /clones/<name>                     | |   rsync, streaming repl.,
        +------------------------------------+ |    custom dump&restore, ...
          +------------------------------------+

```

There are two demon subsystems for every container. One is a [pg_receivexlog] process pulling down WAL data from the upstream DB to $PGARCHIVE/wal_archive using the streaming replication protocol. There's also an optional cron job which compresses old WAL segments.

The second demon subsystem is a PostgreSQL "warm" standby instance in $PGARCHIVE/standby which is initially created with [pg_basebackup] from the upstream DB. Afterwards it polls the local wal_archive and applies completed WAL segments to its cluster directory, so it mirrors the upstream DB with some lag.

The snapshot subsystem is purely cron-based, it uses btrfs to create snapshots of the standby cluster directory. This is fast and space efficient, although there's some redundancy with the wal_archive. Creating more snapshots will decrease time to recovery for some storage increase. Automatic time based expiry is included in the cron job.

Finally snapshots may be "cloned" to separate directories by users. They are provisioned with a suitable [recovery.conf] automatically, and can be run instantly without manual modification. Once started they will perform recovery according to parameters in [recovery.conf], and present a usable PostgreSQL server instance some time afterwards. "Cloning" is just doing another btrfs snapshot under the hood and so is very fast, but the time needed for WAL playback will vary according to the amount needed and available CPU and IO resources. Faster restore times can be traded for storage space required by more frequent snapshots. Clones are created and deleted by users only.

Note that there is no full disaster recovery support yet, at the moment it's left to the user to rsync a clone back to a production server, or do fancy stuff with streaming replication, or do whatever works best in his environment. A knowledgeable user could also copy a snapshot to some replacement server dierctly, and connect remotely to the container's wal_archive, thereby skip creating a local clone. It's not recommended to run a local clone as full production replacement (at least not for a longer than strictly necessary). Support for more scripted DR options are planned for the future.


## Installation

1. You must have PostgreSQL 9.4 installed, including its client and server applications.
1. You must have `btrfs` installed, from btrfs-progs.
1. Copy `pgarchive` from this repository to /usr/local/bin or somewhere else in your path. Make sure it has +x permissions.
1. Edit `pgarchive's` `export PATH=` line near the top so it can find the PostgreSQL applications and `btrfs`.

If you want fancy bash completion to work you should execute the following, or better yet put it in a .bashrc or equivalent file:

```
eval "$(pgarchive bash-completion)"
```

All invocations of pgarchive require $PGARCHIVE to be the path to a backup container, which must be located on a btrfs filesystem. You might want to add something like this to your .bashrc too:

```
export PGARCHIVE=/path/to/container
```

For high-load DBs at emarsys.com the following btrfs mount options work best. You do need `user_subvol_rm_allowed` for snapshot expiry to work, the rest merely improve performance for us. See [btrfs mount options] for more details.

```
user_subvol_rm_allowed,noatime,nobarrier,enospc_debug,nodatacow,metadata_ratio=6
```

Now you need to use `pgarchive container init`, possibly with more configuration, to create container(s). Please see the MANUAL.md (TODO) for more information. Re-starting containers on server re-boots is left up to you.


## Monitoring

Backup services need monitoring. How exactly this is done is out of scope of pgarchive and determined by local policy, but here is what we found useful at emarsys.com (for each container):

The wal_archive directory should have frequent activity (mtime). If your upstream DB has long idle periods setting [archive_timeout] upstream will put a limit on this (like 1 hour). Typically the wal_archive subsystem is not running if this alert is raised, or otherwise cannot write to the wal_archive directory.

The snapshots directory should have frequent activity too, depending on the cron job for snapshot creation. Note snapshot creation only succeeds if the standby process is active (running and has a recent restartpoint), so this typically flags standby process problems.

You absolutely need to monitor the size of your upstream DB's $PGDATA/pg_xlog directory. But you already do this, don't you? If pgarchive containers cannot stream replication data, old WAL segments cannot be cleaned up there because it's held by the container's [replication slot][replication slots]. You may eventually need to decide to drop the replication slot, which will break synchronization with your container.

Of course you should monitor free disk space for your containers. Monitoring IO and CPU performance on your backup server is a given too.

It's probably less useful to check for the presence of [pg_receivexlog] and PostgreSQL standby processes, because it's hard to associate them with multiple containers easily, and this may miss certain problems if subsystems become stuck without dying fully.


## Command Reference (output of pgarchive --help)

TODO: create a MANUAL.md

```
PGARCHIVE=<dir> pgarchive <cmd> <arg...>

Commands and sub-commands:

help                Show this help.

container init      Create a new container and upstream replication slot, both of which
                    must not exist yet. This requires additional environment variables to be set,
                    see below. The new backup container is fully started and operational at the end,
                    except you need to verify and enable the cron jobs (crontab -e).
container init-cron Add a new default cron stanza to the calling user's crontab, in case word in
                    case you messed up the original.
container purge [--force]
                    Completely remove the full backup location and the replication slot.
                    Tries to read PGARCHIVE's config, and falls back to the same
                    extra environment parameters init uses. Make sure nothing at all is needed
                    any more, and everything is stopped.
container start     Start backup container, which consists of the wal_archive and standby processes.
container stop      Stop backup container.
container status    Show status of wal_archive and standby subsystems, exit 1 if any is not running.
container dashboard Show detailed status and tails of log files.

wal_archive start   Start the wal_archive process. This process connects to the upstream DB
                    using the PostgreSQL's replication protocol, with the configured
                    replication slot, and stores retrieved WAL segments in the wal_archive
                    directory. Implemented by using pg_receivexlog.
wal_archive stop    Stop the wal_archive process.
wal_archive status  Check if the wal_archive process is running.
wal_archive list    List wal_archive directory and size.
wal_archive log [-f]
                    Write wal_archive.log to stdout or tail -f it.
                    If stdout is a terminal $PAGER is used to show it.

standby (start|stop|status|reload|restart) [arg] ...
                    The standby process is a warm standby PostgreSQL instance which pulls
                    completed segments from the wal_archive directory. This is a simple frontend
                    to control it using pg_ctl.
standby log [-f]    Write today's standby log to stdout or tail -f it.
                    If stdout is a terminal $PAGER is used to show it.
                    Assumes the standard weekly log rotation and csv logging.

snapshot create [--force]
                    Create a new btrfs snapshot of the standby directory under snapshots/.
                    With --force do this even when the standby process is not running,
                    and or if there's no recent restartpoint.
snapshot delete <pattern> ...
                    Directly delete one or more snapshots, may use a simple globbing pattern.
snapshot expire [--show]
                    Delete snapshots older than $expire_date, which should be defined in the
                    config file. Passing $EXPIRE_DATE in the environment may be used to
                    override the config file. This also removes WAL segments which are not
                    needed any more from the wal_archive.
                    The format of the date must be understood by date (1) --date=STRING,
                    and may be a relative term like "1 week ago".
                    With --show only show the actions to be taken, but don't delete anything.
snapshot list [<pattern>]
                    List snapshots, optionally restricted to a simple globbing pattern.

clone create <snapshot> <name> [<target-time>]
                    Create a new named (PostgreSQL) clone instance from a snapshot.
                    A new port is assigned and appended to postgresql.conf.
                    The clone will have a recovery.conf suitable to do recovery to
                    either the next consistent point in time, or an arbitrary target time
                    (which should be after snapshot creation time). The clone is not
                    started automatically, so you may edit *.conf files before
                    initiating recovery. Recovery success depends on the wal_archive
                    still containing all required WAL segments.
clone duplicate <source> <target>
                    Duplicates an existing clone instance. Note that if source is running
                    target will have to do crash recovery once started. A new port is
                    assigned and appended to postgresql.conf automatically.
clone delete <name> Delete a (stopped) clone.
clone (start|stop|status|reload|restart) <name> [arg] ...
                    This is a simple frontend for pg_ctl to start and stop named clones.
                    The stop command also accepts --all instead of <name>.
clone list [--status] [<pattern>]
                    List clones, optionally restricted to a simple globbing pattern.
                    With --status the assigned port and pg_ctl status is given for each.
clone log <name> [-f]
                    Write the named clone's log to stdout or tail -f it.
                    If stdout is a terminal $PAGER is used to show it.
                    Assumes the standard weekly log rotation and csv logging.
clone psql <name> [arg] ...
                    Open a psql shell connected to a running named clone instance.
                    Additional arguments are passed through.

cron compress-wal-archive <n>
                    Compress completed segments in wal_archive, which have already been
                    processed by the standby process. Run n processes in parallel (default 1).
                    The log is written to log/cron_wal_archive.log.
cron expire-and-create-snapshot
                    Expire old snapshots and wal segments, and create a new snapshot.
                    The log is written to log/cron_snapshot.log.
cron defrag-btrfs   Defragment the standby directory, since PostgreSQL tends to cause a critical
                    amount of fragmentation fast. Requires password-less sudo privileges to call
                    "/sbin/btrfs filesystem defrag *". The log is written to log/cron_btrfs.log.

bash-completion     Output a bash completion snippet for this utility.
                    This may be saved to global or personal profiles, or eval'ed directly:
                        eval "$(pgarchive bash-completion)"

Required Environment:

PGARCHIVE           Base path to a backup container, which must be on btrfs.

Extra environment parameters only used by 'container init' and 'purge' commands (settings are
persisted to $PGARCHIVE/pgarchive.conf):

SLOT                Create and use this replication slot at upstream. Defaults to
                    "pgarchive_<basename of $PGARCHIVE>".
UPSTREAM            A connection info string used when accessing the upstream DB in
                    command mode, for example when creating a SLOT or checking status. See
                    http://www.postgresql.org/docs/9.4/static/libpq-connect.html#LIBPQ-CONNSTRING.
                    The default value is empty (local connection, default port, user, DB etc.)
                    This is persisted to the config file and may be edited there later.
                    Note that the usual libpq environment settings (PG*) which are active in the
                    caller's shell are explicitly unset, so this needs to provide enough
                    parameters to work in isolation (e.g. by cron, or a different user
                    shell) later.
UPSTREAM_REPL       The connection info string the WAL archive process uses to fetch
                    new WAL segments. Defaults to UPSTREAM + " user=replication".
                    All of UPSTREAM's caveats apply.

Container configuration file:

$PGARCHIVE/pgarchive.conf
                    Contains various settings, primarily persisted upstream connection parameters,
                    and a running counter for clone port numbers.
                    This is sourced as a shell script, and could theoretically be used to override
                    most internal variables and functions, at your own risk.
```

### Examples

TODO


## Security

There are no internal security mechanisms at all. Note especially:

* This is expected to be run in a trusted net, or over a VPN. You could try to secure the upstream DB connections using SSL certs, but this is up to you.
* By default backup containers can log into their upstream DB server using a privileged user, although you could reduce this to replication only after container initialization.
* Access to containers and their data is purely host based. Data is not encrypted.


## About

Created by Jürgen Strobel <juergen.strobel@emarsys.com> while working at [emarsys.com].


[archive_timeout]: http://www.postgresql.org/docs/9.4/static/runtime-config-wal.html#GUC-ARCHIVE-TIMEOUT
[btrfs mount options]: https://btrfs.wiki.kernel.org/index.php/Mount_options
[btrfs]: https://btrfs.wiki.kernel.org/index.php/Main_Page
[emarsys.com]: http://emarsys.com/
[pg_basebackup]: http://www.postgresql.org/docs/9.4/static/app-pgbasebackup.html
[pg_receivexlog]: http://www.postgresql.org/docs/9.4/static/app-pgreceivexlog.html
[recovery.conf]: http://www.postgresql.org/docs/9.4/static/recovery-config.html
[replication slots]: http://www.postgresql.org/docs/9.4/static/warm-standby.html#STREAMING-REPLICATION-SLOTS
