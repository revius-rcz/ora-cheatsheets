# Oracle ExaWatcher Cheatsheet

## Contents

1. [What ExaWatcher is](#what-exawatcher-is)
2. [First 60 seconds](#first-60-seconds)
3. [Lifecycle](#lifecycle)
4. [Configuration and collectors](#configuration-and-collectors)
5. [Extract an incident window](#extract-an-incident-window)
6. [Storage management](#storage-management)
7. [What to inspect](#what-to-inspect)
8. [Common symptoms](#common-symptoms)
9. [Maintenance checklists](#maintenance-checklists)

## What ExaWatcher is

ExaWatcher is a high-frequency diagnostic collector installed on Exadata database and storage servers. It gathers operating-system, process, network, and—on storage servers—`cellsrvstat` evidence. Most operating-system statistics are normally sampled every five seconds.

Use it for detailed debugging and baseline comparison. It is not a database workload repository and does not replace AWR/ASH, SQL Monitor, CellCLI metrics, or alert logs.

| Item | Typical value |
| --- | --- |
| Installation | `/opt/oracle.ExaWatcher` |
| Service | `ExaWatcher` |
| Collector | `/opt/oracle.ExaWatcher/ExaWatcher.sh` |
| Default config on Oracle Linux | `/opt/oracle.ExaWatcher/ExaWatcher.conf` |
| Extractor | `/opt/oracle.ExaWatcher/GetExaWatcherResults.sh` |
| Default interval | 5 seconds, subject to per-collector minimums |
| Documented default space limit | DB server: 6 GB; storage server: 600 MB |
| Documented default compression | `bzip2` |

Paths and defaults are release-sensitive. Confirm them on the target node.

## First 60 seconds

Run as `root` on each relevant server:

```bash
hostname -f
date -Is
timedatectl status
imageinfo -ver 2>/dev/null || true

systemctl status ExaWatcher --no-pager
systemctl is-enabled ExaWatcher
systemctl show ExaWatcher \
  -p ActiveState -p SubState -p MainPID -p ExecMainStartTimestamp

ls -ld /opt/oracle.ExaWatcher
find /opt/oracle.ExaWatcher -maxdepth 2 -type f -mmin -15 -printf '%TY-%Tm-%Td %TH:%TM:%TS %10s %p\n' \
  2>/dev/null | sort | tail -30

df -h /opt /var/tmp
df -i /opt /var/tmp
du -xsh /opt/oracle.ExaWatcher 2>/dev/null

/opt/oracle.ExaWatcher/ExaWatcher.sh --lastconf
/opt/oracle.ExaWatcher/ExaWatcher.sh --listcmd Full
```

Capture the output before restarting or changing configuration.

## Lifecycle

```bash
# Read-only
systemctl status ExaWatcher --no-pager
systemctl is-active ExaWatcher
systemctl is-enabled ExaWatcher
systemctl cat ExaWatcher
journalctl -u ExaWatcher --since '30 minutes ago' --no-pager

# Changes collection state
systemctl start ExaWatcher
systemctl stop ExaWatcher
systemctl restart ExaWatcher

# Collector-level stop; stops collector processes and archives current data
/opt/oracle.ExaWatcher/ExaWatcher.sh --stop
```

Prefer the systemd service for normal lifecycle control. Use `ExaWatcher.sh --stop` only when the installed release's procedure or a controlled collector invocation requires it. Do not use both stop mechanisms blindly; inspect the unit definition first.

After start or restart:

```bash
systemctl is-active ExaWatcher
journalctl -u ExaWatcher --since '5 minutes ago' --no-pager
find /opt/oracle.ExaWatcher -maxdepth 2 -type f -mmin -10 -ls 2>/dev/null
```

## Configuration and collectors

```bash
cd /opt/oracle.ExaWatcher

# Installed syntax and effective configuration
./ExaWatcher.sh --help
./ExaWatcher.sh --lastconf
./ExaWatcher.sh --listcmd Nameonly
./ExaWatcher.sh --listcmd Core
./ExaWatcher.sh --listcmd CMD
./ExaWatcher.sh --listcmd Full

# Create a candidate config; always provide a separate path
candidate=/var/tmp/ExaWatcher.conf.candidate
./ExaWatcher.sh --interval 10 --spacelimit 6144 --createconf "$candidate"

# Review before use
diff -u /opt/oracle.ExaWatcher/ExaWatcher.conf "$candidate"
./ExaWatcher.sh --fromconf "$candidate"
```

`--fromconf` starts a collector using that file; it is not a syntax-only test. Run it only in an approved test/change window and stop the corresponding invocation cleanly.

Frequently useful options:

| Option | Purpose |
| --- | --- |
| `--help` | Show installed syntax. |
| `--lastconf` | Display the most recently used configuration without collecting. |
| `--listcmd Full` | Show collectors, enabled state, intervals, and commands. |
| `--createconf FILE` | Build a configuration from validated inputs. |
| `--fromconf FILE` | Run using a chosen configuration. |
| `--interval SECONDS` | Set a group sampling interval. |
| `--spacelimit MB` | Limit collector storage use. |
| `--count NUMBER` | Set archive count for a group. |
| `--disable NAME` | Disable one collector in a group. |
| `--commandmode ALL|CORE|SELECTED` | Select the collection-module set. |
| `--resultdir DIR` | Set collection result location. |
| `--zip bzip2|gzip` | Select compression. |
| `--stop` | Stop collector processes and archive current data. |

## Extract an incident window

The result extractor uses underscore-separated date and time:

```bash
cd /opt/oracle.ExaWatcher

result_dir=/var/tmp/exawatcher-results
install -d -m 0700 "$result_dir"

./GetExaWatcherResults.sh \
  --from 08/26/2026_10:00:00 \
  --to   08/26/2026_10:30:00 \
  --resultdir "$result_dir"
```

Confirm supported options first:

```bash
./GetExaWatcherResults.sh --help
```

Package the same wall-clock interval from every relevant database and storage server. Record each node's timezone and clock state.

After extraction:

```bash
find "$result_dir" -maxdepth 2 -type f -printf '%TY-%Tm-%Td %TH:%TM %12s %p\n' | sort
sha256sum "$result_dir"/* 2>/dev/null
```

The archive can include raw data and HTML charts. After extracting it on a workstation, look for:

```text
Charts.ExaWatcher.<hostname>/<timestamp>_<duration>/index.html
```

Oracle documents `-c`/`--scp` for copying the result archive. Prefer the organization's approved transfer method, and never put a password in the command line or shell history.

## Storage management

### Inspect

```bash
df -h /opt /var/tmp
df -i /opt /var/tmp
du -xsh /opt/oracle.ExaWatcher
du -xh --max-depth=2 /opt/oracle.ExaWatcher 2>/dev/null | sort -h | tail -30
find /opt/oracle.ExaWatcher -xdev -type f -printf '%s %TY-%Tm-%Td %TH:%TM %p\n' \
  2>/dev/null | sort -nr | head -30
```

### Control growth

- Use `--spacelimit MB` through a reviewed configuration.
- Increase `--interval` only after evaluating the loss of diagnostic resolution.
- Disable a high-volume collector only when the incident and support requirements permit it.
- Keep extraction output outside a constrained `/opt` filesystem.
- Remove old result bundles copied back into the installation tree before touching collector-managed archives.
- Preserve the incident window and a representative baseline before reducing retention.

Do not assume `--count` means days. It is an archive count, and coverage depends on interval, rotation, compression, collector output, and release.

## What to inspect

| Question | Primary evidence | Correlate with |
| --- | --- | --- |
| Host CPU saturation? | `Iostat`, `Mpstat`, `Top`, `Ps` | AWR host CPU, run queues, application demand |
| Memory pressure? | `Vmstat`, `Meminfo`, `Slabinfo`, `Top`, `Ps` | database memory changes, OOM/kernel logs |
| Device latency or queueing? | `Iostat`, `Diskinfo` | CellCLI metrics, AWR I/O waits, storage alerts |
| Network loss or congestion? | `Netstat`, interface/RDMA collectors | switch counters, CellCLI alerts, client errors |
| Storage-cell behavior? | `CellSrvStat`, `FlashSpace`, `RDSinfo` | CellCLI metrics/alerts, database cell waits |
| One process caused a spike? | `Top`, `Ps` | process logs, job schedules, SQL/ASM activity |

Interpret deltas and sustained patterns. A single isolated sample is usually insufficient.

## Common symptoms

| Symptom | First checks | Avoid |
| --- | --- | --- |
| Service inactive | `systemctl status`, `journalctl`, unit definition, free space | Restarting before preserving logs |
| Active but no new files | recent `find`, process list, config, permissions, bytes/inodes | Assuming `active` proves collection |
| Extractor returns no data | node/timezone, archive coverage, exact syntax, disk space | Expanding immediately to multiple days |
| Charts missing | local extraction, chart directory/index, source collectors | Treating missing charts as missing raw data |
| `/opt` nearly full | `df`, `du`, result bundles, effective space limit | `rm -rf` or deleting incident evidence |
| High collector overhead | `top`, enabled collectors, intervals, custom commands | Killing arbitrary child processes |
| Failure after patch | image version, unit, scripts, config diff, journal | Copying scripts from another release |

## Maintenance checklists

### Before reboot or patch

```bash
date -Is
imageinfo -ver 2>/dev/null || true
systemctl status ExaWatcher --no-pager
/opt/oracle.ExaWatcher/ExaWatcher.sh --lastconf
/opt/oracle.ExaWatcher/ExaWatcher.sh --listcmd Full
cp -a /opt/oracle.ExaWatcher/ExaWatcher.conf \
  "/var/tmp/ExaWatcher.conf.$(date +%Y%m%dT%H%M%S%z).bak"
```

- Extract any required pre-maintenance baseline.
- Record service enablement, active state, software version, configuration checksum, and archive coverage.
- Confirm the approved Exadata update procedure rather than patching ExaWatcher independently.

### After reboot or patch

```bash
date -Is
imageinfo -ver 2>/dev/null || true
systemctl status ExaWatcher --no-pager
systemctl is-enabled ExaWatcher
journalctl -u ExaWatcher -b --no-pager
/opt/oracle.ExaWatcher/ExaWatcher.sh --lastconf
/opt/oracle.ExaWatcher/ExaWatcher.sh --listcmd Full
find /opt/oracle.ExaWatcher -maxdepth 2 -type f -mmin -10 -ls 2>/dev/null
```

Verify actual file growth and perform a short test extraction; service status alone is not sufficient.

