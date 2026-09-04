# Oracle ExaWatcher Command Reference

## Contents

1. [Conventions](#conventions)
2. [Systemd commands](#systemd-commands)
3. [ExaWatcher.sh syntax](#exawatchersh-syntax)
4. [ExaWatcher.sh options](#exawatchersh-options)
5. [GetExaWatcherResults.sh](#getexawatcherresultssh)
6. [Collector reference](#collector-reference)
7. [Inspection commands](#inspection-commands)
8. [Time formats](#time-formats)
9. [Exit and verification discipline](#exit-and-verification-discipline)

## Conventions

Run ExaWatcher as `root`. Start in the installed directory:

```bash
cd /opt/oracle.ExaWatcher
```

Always read local help first:

```bash
./ExaWatcher.sh --help
./GetExaWatcherResults.sh --help
```

The tables below reflect current Oracle Exadata documentation reviewed on 2026-08-26. Installed options and behavior can differ by Exadata System Software release.

## Systemd commands

| Command | Changes state | Purpose |
| --- | --- | --- |
| `systemctl status ExaWatcher --no-pager` | No | Show unit and recent status. |
| `systemctl is-active ExaWatcher` | No | Return active state. |
| `systemctl is-enabled ExaWatcher` | No | Show boot enablement. |
| `systemctl cat ExaWatcher` | No | Show unit and drop-ins. |
| `systemctl show ExaWatcher` | No | Show effective unit properties. |
| `journalctl -u ExaWatcher --since '1 hour ago' --no-pager` | No | Show service journal. |
| `systemctl start ExaWatcher` | Yes | Start continuous collection. |
| `systemctl stop ExaWatcher` | Yes | Stop collection and create a gap. |
| `systemctl restart ExaWatcher` | Yes | Stop/start collection and create a gap. |

Oracle's current documentation explicitly shows:

```bash
systemctl start ExaWatcher
```

Inspect the unit before combining systemd with direct `ExaWatcher.sh` lifecycle commands.

## ExaWatcher.sh syntax

```bash
/opt/oracle.ExaWatcher/ExaWatcher.sh [options]
```

With no options, the utility runs using default options.

Several options apply to the **current group**. Use `--group` to begin another group and place group-specific options after it, following the installed help. Avoid constructing a complex multi-group production configuration without generating it to a candidate file and reviewing the entire result.

## ExaWatcher.sh options

### `-c`, `--command`

Change the core command executed for a supported collector in the current group.

```bash
./ExaWatcher.sh \
  --command 'Vmstat;; "vmstat -a"' \
  --createconf /var/tmp/ExaWatcher.vmstat-a.conf
```

Oracle documents command replacement for these core collectors:

- `CellSrvStat`
- `Iostat`
- `Mpstat`
- `Netstat`
- `Ps`
- `Top`
- `Vmstat`

Changing a command can break parsers/charts or increase overhead. Review it as code.

### `--createconf [FILE]`

Parse and validate supplied command-line inputs and create a configuration file.

```bash
./ExaWatcher.sh \
  --interval 10 \
  --spacelimit 6144 \
  --createconf /var/tmp/ExaWatcher.conf.candidate
```

If the path/name is omitted, Oracle documents that the default configuration can be overwritten. Always use an explicit candidate file unless replacement is deliberately approved.

### `-d`, `--disable NAME`

Disable one collector in the current group.

```bash
./ExaWatcher.sh \
  --disable 'Vmstat' \
  --createconf /var/tmp/ExaWatcher.no-vmstat.conf
```

Use the exact installed collector name from `--listcmd Nameonly`.

### `-e`, `--end "END_TIME"`

Set the current group's end time. The documented default is 10 years from the current time.

```bash
./ExaWatcher.sh --end '08/26/2026 18:00:00'
```

This collector-control syntax uses a space between date and time. It differs from the result extractor's underscore format.

### `--fromconf [FILE]`

Run using a chosen configuration file. The documented default on Oracle Linux is `/opt/oracle.ExaWatcher/ExaWatcher.conf`.

```bash
./ExaWatcher.sh --fromconf /var/tmp/ExaWatcher.conf.candidate
```

This starts collection; it is not a read-only validation command.

### `-g`, `--group`

Start a new data-gathering group. Subsequent group options apply to that group.

```bash
./ExaWatcher.sh --group --interval 10 --commandmode CORE
```

Use local help for multi-group ordering and selection syntax.

### `-h`, `--help`

Display help:

```bash
./ExaWatcher.sh --help
```

Capture it during inventory and after patching.

### `-i`, `--interval SECONDS`

Set the sampling interval for the current group. The documented default is five seconds.

```bash
./ExaWatcher.sh --interval 10
```

Some collectors have minimum intervals because they consume more resources. The group interval must respect those minimums.

### `-l`, `--spacelimit MB`

Set the storage space limit used by the utility, in MB.

```bash
./ExaWatcher.sh --spacelimit 900
```

Oracle-documented defaults:

- Database server: 6 GB.
- Storage server: 600 MB.

Confirm the effective value with `--lastconf`; do not infer it from node role alone.

### `--lastconf`

Display the most recently used configuration. No data is collected.

```bash
./ExaWatcher.sh --lastconf
```

Use it before changing the default file.

### `--listcmd [MODE]`

Display collector/command information. Documented modes:

| Mode | Output |
| --- | --- |
| `Full` | All command and sampler information. |
| `Nameonly` | Names and enabled state. |
| `Core` | Core sampler information. |
| `CMD` | Names, enabled state, and default commands. |
| `Enabled` | Enabled collector information where supported by the installed release. |
| no mode | Release-defined default listing. |

Examples:

```bash
./ExaWatcher.sh --listcmd Full
./ExaWatcher.sh --listcmd Nameonly
./ExaWatcher.sh --listcmd Core
./ExaWatcher.sh --listcmd CMD
```

Confirm exact capitalization/mode support locally.

### `-m`, `--commandmode MODE`

Select module scope for the current group:

| Value | Meaning |
| --- | --- |
| `ALL` | Run all collection modules; documented default. |
| `CORE` | Run core collection modules only. |
| `SELECTED` | Run only selected collection modules. |

```bash
./ExaWatcher.sh --commandmode CORE
```

Use installed help to specify the selected list with `SELECTED`.

### `-o`, `--count ARCHIVE_COUNT`

Set the current group's archive count. The documented default is 720.

```bash
./ExaWatcher.sh --count 500
```

This is a count, not a duration. Do not describe it as days or hours without measuring actual rotation.

### `-r`, `--resultdir DIRECTORY`

Set the collector result directory.

```bash
./ExaWatcher.sh --resultdir /opt/oracle.ExaWatcher/archive
```

Before changing it, verify ownership, mode, mount availability at boot, capacity, inode headroom, security, and patch/reboot behavior.

### `--stop`

Stop the utility and all its processes, then compress current data files.

```bash
./ExaWatcher.sh --stop
```

This creates a collection gap. Inspect the systemd unit and local help before using it with a service-managed instance.

### `-t`, `--start "START_TIME"`

Set the current group's start time. The documented default is 20 seconds from the current time.

```bash
./ExaWatcher.sh --start '08/26/2026 12:00:00'
```

### `-u`, `--customcmd`

Add a custom collection module to the current group.

```bash
./ExaWatcher.sh \
  --customcmd 'Lsl;; "/bin/ls -l"' \
  --createconf /var/tmp/ExaWatcher.custom.conf
```

Validate exact delimiter syntax with local help. Require security, runtime, volume, and sensitive-data review. Custom commands must be read-only and non-interactive.

### `-z`, `--zip PROGRAM`

Select compression. Oracle documents `bzip2` and `gzip`, with `bzip2` as default.

```bash
./ExaWatcher.sh --zip gzip
```

Verify the program exists and account for the CPU-versus-space tradeoff.

## GetExaWatcherResults.sh

Use the extractor to package raw data and supported charts for a time range.

### Help

```bash
./GetExaWatcherResults.sh --help
```

### Documented extraction example

```bash
./GetExaWatcherResults.sh \
  --from 08/24/2023_17:00:00 \
  --to   08/25/2023_17:00:00 \
  --resultdir /var/log/oracle/ExaWatcherResults
```

Prefer a smaller initial window in a real incident.

### Common documented options

| Option | Purpose |
| --- | --- |
| `--from MM/DD/YYYY_HH24:MI:SS` | Beginning of extraction window. |
| `--to MM/DD/YYYY_HH24:MI:SS` | End of extraction window. |
| `--resultdir DIRECTORY` | Destination for the compressed result. |
| `-c`, `--scp` | Copy the resulting archive to another location. |

The extractor's complete option set is release-sensitive. Do not infer undocumented SCP argument syntax; use local `--help` and approved credential handling.

### Chart result

After unpacking the archive, look for:

```text
Charts.ExaWatcher.<hostname>/<timestamp>_<duration>/index.html
```

The charts may include I/O, CPU, CPU detail, cell server statistics, and alert history. They represent only part of the collected data.

## Collector reference

Availability varies by node role, hardware generation, and release.

| Collector | Purpose | Documented interval constraint |
| --- | --- | --- |
| `CellSrvStat` | Cell server status/statistics. | Check local configuration. |
| `Diskinfo` | Disk I/O counters such as reads, merges, and time spent reading. | Check local configuration. |
| `FlashSpace` | Raw flash-card space value. | Minimum 300 seconds. |
| `IBCardInfo` | RDMA/InfiniBand card and port status. | Minimum 300 seconds; Oracle notes it is not available on X8M systems. |
| `IBprocs` | Commands checking RDMA/InfiniBand card status. | Minimum 600 seconds. |
| `Iostat` | CPU and device/partition I/O statistics. | Check local configuration. |
| `Lsof` | Files opened by processes. | Minimum 120 seconds. |
| `MegaRaidFW` | MegaRAID firmware/battery information. | Minimum 86400 seconds. |
| `Meminfo` | Kernel memory-management counters. | Check local configuration. |
| `Mpstat` | Per-processor statistics. | Check local configuration. |
| `Netstat` | Network connection/protocol statistics. | Check local configuration. |
| `Ps` | Active process statistics. | Check local configuration. |
| `RDSinfo` | Cell-server availability. | Limit/minimum 30 seconds. |
| `Slabinfo` | Kernel slab-cache information. | Check local configuration. |
| `Top` | Dynamic process/resource view. | Check local configuration. |
| `Vmstat` | Virtual memory and system activity. | Check local configuration. |

Do not force a group interval below a collector's minimum.

## Inspection commands

### Identity and time

```bash
hostname -f
date -Is
date -u -Is
timedatectl status
imageinfo -ver 2>/dev/null || true
```

### Space

```bash
df -h /opt /var/tmp
df -i /opt /var/tmp
du -xsh /opt/oracle.ExaWatcher
du -xh --max-depth=2 /opt/oracle.ExaWatcher 2>/dev/null | sort -h | tail -30
```

### Freshness

```bash
find /opt/oracle.ExaWatcher -maxdepth 3 -type f -mmin -15 \
  -printf '%TY-%Tm-%Td %TH:%TM:%TS %12s %p\n' 2>/dev/null | sort
```

### Processes

```bash
systemctl show ExaWatcher -p MainPID -p ControlGroup
pgrep -af 'ExaWatcher|iostat|vmstat|mpstat|cellsrvstat' || true
ps -eo pid,ppid,etimes,pcpu,pmem,stat,args --sort=-pcpu | head -40
```

### Checksums

```bash
sha256sum /opt/oracle.ExaWatcher/ExaWatcher.conf
sha256sum /var/tmp/exawatcher-results/* 2>/dev/null
```

## Time formats

Do not mix these formats:

| Tool | Example |
| --- | --- |
| `ExaWatcher.sh --start/--end` | `08/26/2026 12:00:00` |
| `GetExaWatcherResults.sh --from/--to` | `08/26/2026_12:00:00` |
| Evidence manifest | `2026-08-26T12:00:00+02:00` |
| UTC manifest | `2026-08-26T10:00:00Z` |

Record the node's timezone and UTC equivalent for every incident window.

## Exit and verification discipline

Do not judge success only from a zero exit code or `active` service state. Verify postconditions:

| Action | Verification |
| --- | --- |
| Start/restart | Active state, clean journal, collector processes, recent/growing files. |
| Config activation | `--lastconf`, `--listcmd Full`, expected result path, short extraction. |
| Space change | Effective limit, `df`, directory growth trend, required lookback retained. |
| Extraction | Correct node/time range, non-empty archive, file inventory, SHA-256, raw files/charts. |
| Patch/reboot | Image release, service enablement/state, config diff, fresh output, test extraction. |

