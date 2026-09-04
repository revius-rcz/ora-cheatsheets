# Oracle ExaWatcher Tutorial

## Contents

1. [Scope](#scope)
2. [Mental model](#mental-model)
3. [Discover the local installation](#discover-the-local-installation)
4. [Verify continuous collection](#verify-continuous-collection)
5. [Understand the collected evidence](#understand-the-collected-evidence)
6. [Extract a first incident window](#extract-a-first-incident-window)
7. [Review the result](#review-the-result)
8. [Inspect raw data](#inspect-raw-data)
9. [Correlate an incident](#correlate-an-incident)
10. [Build a useful baseline](#build-a-useful-baseline)
11. [Make a controlled configuration change](#make-a-controlled-configuration-change)
12. [Practice recovery](#practice-recovery)
13. [Production acceptance checklist](#production-acceptance-checklist)

## Scope

This tutorial is for an administrator taking over an existing Exadata environment. It covers the host-level ExaWatcher collector already supplied with Exadata System Software. It does not install ExaWatcher, replace Oracle Support procedures, or teach general Linux/Exadata performance tuning.

Run the commands on a lab or approved production node as `root`. Replace all example timestamps and destinations. ExaWatcher releases differ, so local help and effective configuration are authoritative.

## Mental model

ExaWatcher answers: **what was the host and network doing at a precise time?**

It runs separately on every database server and storage server. A database-node archive cannot show the storage-cell process activity, and one cell's archive cannot represent all cells. Collect from each node that participated in the failing path.

The evidence layers complement one another:

| Layer | Best source | Example question |
| --- | --- | --- |
| Application | application logs/traces | Which request failed and when? |
| Database session/SQL | ASH, AWR, SQL Monitor, alert log | Which SQL or wait event dominated? |
| Database host | ExaWatcher | Was CPU, memory, process, or network pressure present? |
| Storage cell | ExaWatcher plus CellCLI metrics/alerts | Was cell CPU, flash, disk, or fabric behavior abnormal? |
| Fabric/switch | switch and fabric telemetry | Was the path dropping or delaying traffic? |

Use a common timezone and retain original timestamps. Never align evidence by file modification time alone.

## Discover the local installation

Start read-only:

```bash
hostname -f
date -Is
timedatectl status
imageinfo -ver 2>/dev/null || true

systemctl status ExaWatcher --no-pager
systemctl is-enabled ExaWatcher
systemctl cat ExaWatcher

ls -la /opt/oracle.ExaWatcher
```

Then inspect the installed interface:

```bash
cd /opt/oracle.ExaWatcher
./ExaWatcher.sh --help
./ExaWatcher.sh --lastconf
./ExaWatcher.sh --listcmd Nameonly
./GetExaWatcherResults.sh --help
```

Record:

- Exadata System Software release.
- Node role: database server or storage server.
- Service active/enabled state.
- Unit `ExecStart` and any environment files.
- Effective result directory, space limit, interval, compression, enabled collectors, and custom commands.
- Local timezone and clock synchronization state.

Do not assume another node has the same configuration. Compare nodes of the same role explicitly.

## Verify continuous collection

An active systemd unit is necessary but not sufficient. Prove that output is advancing:

```bash
systemctl is-active ExaWatcher
journalctl -u ExaWatcher --since '15 minutes ago' --no-pager

find /opt/oracle.ExaWatcher -maxdepth 2 -type f -mmin -15 \
  -printf '%TY-%Tm-%Td %TH:%TM:%TS %10s %p\n' 2>/dev/null | sort
```

If the configured result directory is outside the installation tree, repeat `find` there.

Check processes without relying on one exact child-process name:

```bash
systemctl show ExaWatcher -p MainPID -p ControlGroup
pgrep -af 'ExaWatcher|iostat|vmstat|mpstat|cellsrvstat' || true
```

Expected evidence:

1. Service is active.
2. The collector or sampler processes exist.
3. Files in the configured result/archive location are new or growing.
4. Filesystem bytes and inodes are available.
5. The journal does not show repeated collector, permission, compression, or space errors.

## Understand the collected evidence

The exact collector set depends on node role and release. Common collectors include:

| Collector | Use |
| --- | --- |
| `Iostat` | Host CPU summary and device/partition I/O rates, utilization, queueing, and latency. |
| `Mpstat` | Per-CPU utilization and imbalance. |
| `Vmstat` | Run queue, memory, paging, block I/O, interrupts, context switches, and CPU. |
| `Meminfo` | Kernel memory counters. |
| `Slabinfo` | Kernel slab-cache use. |
| `Top` / `Ps` | Process activity and resource consumers. |
| `Netstat` | Network connection and protocol statistics. |
| `Diskinfo` | Low-level disk I/O statistics. |
| `CellSrvStat` | Exadata storage server statistics. |
| `FlashSpace` | Raw flash-card space information. |
| `RDSinfo` | Cell availability from the fabric/RDS perspective. |

Sampling intervals matter. A five-second collector can expose short bursts that a one-minute average hides. Conversely, one high sample is not proof of a sustained bottleneck. Compare several samples before, during, and after the event.

## Extract a first incident window

Assume the application reported a slowdown at `2026-08-26 10:15:00 +02:00`. Use a 30-minute window from 10:00 through 10:30 on every involved node.

First verify timezone and archive coverage:

```bash
date -Is
timedatectl status
find /opt/oracle.ExaWatcher -maxdepth 3 -type f \
  -printf '%TY-%Tm-%Td %TH:%TM %p\n' 2>/dev/null | sort | tail -50
```

Choose a result directory with adequate capacity:

```bash
df -h /var/tmp
df -i /var/tmp
install -d -m 0700 /var/tmp/exawatcher-results
```

Run the extractor:

```bash
cd /opt/oracle.ExaWatcher

./GetExaWatcherResults.sh \
  --from 08/26/2026_10:00:00 \
  --to   08/26/2026_10:30:00 \
  --resultdir /var/tmp/exawatcher-results
```

The documented result syntax is `MM/DD/YYYY_HH24:MI:SS`. Confirm the installed help, especially on older images.

Repeat the same time window on:

- Every database node that ran the workload.
- Every storage cell serving the affected grid disks.
- Any additional node implicated by network or cluster evidence.

Do not generate a rack-wide multi-day bundle as the first response. Expand only if evidence requires it.

## Review the result

Inventory and checksum the result before moving or unpacking it:

```bash
find /var/tmp/exawatcher-results -maxdepth 2 -type f \
  -printf '%TY-%Tm-%Td %TH:%TM:%TS %12s %p\n' | sort

sha256sum /var/tmp/exawatcher-results/* 2>/dev/null
```

Record a manifest outside the archive:

```text
Host: db01.example.com
Role: database server
Node timezone: Europe/Warsaw (+02:00 at incident)
Incident time: 2026-08-26T10:15:00+02:00
Extracted range: 2026-08-26 10:00:00 through 10:30:00 local time
Exadata image: <imageinfo output>
Result archive: <file name>
SHA-256: <checksum>
```

Copy the archive through an approved secure channel. Oracle documents the extractor's `-c`/`--scp` option, but an organization-managed transfer service may provide stronger audit and credential handling.

After extracting on a workstation, open the chart index if present:

```text
Charts.ExaWatcher.<hostname>/<timestamp>_<duration>/index.html
```

The generated charts cover a subset of data, commonly:

- I/O and CPU summary from `iostat`.
- Per-CPU detail from `mpstat`.
- Storage-cell statistics from `cellsrvstat`.
- Alert history for the selected window.

If charts are absent or cannot render, continue with the raw collector files. Missing charts do not prove missing raw evidence.

## Inspect raw data

Keep the original result archive unchanged. Work from a separate extraction directory on an analysis host:

```bash
file ExaWatcher-result-archive
tar -tf ExaWatcher-result-archive | sed -n '1,120p'

install -d -m 0700 ./exawatcher-analysis
tar -xaf ExaWatcher-result-archive -C ./exawatcher-analysis

find ./exawatcher-analysis -type f \
  -printf '%12s %TY-%Tm-%Td %TH:%TM %p\n' | sort -k4,4 -k5,5
```

Replace `ExaWatcher-result-archive` with the actual file. If `file` shows a format unsupported by `tar -a`, use the matching decompressor without overwriting the original.

Start with the chart time series, then open the corresponding raw collector file around the same timestamp. Preserve headers because collector versions can change column order or units.

Useful orientation commands:

```bash
# Find likely collector directories/files.
find ./exawatcher-analysis -type f \
  | grep -Ei '/(Iostat|Mpstat|Vmstat|Meminfo|Top|Ps|Netstat|CellSrvStat|RDSinfo)'

# Locate an exact incident time in text output.
grep -RFn -- '10:15:' ./exawatcher-analysis | head -100

# Inspect a selected text file without changing it.
sed -n '1,160p' ./exawatcher-analysis/path/to/selected-file
```

Do not apply a fixed `awk` column parser until the file's header and units have been verified. For counters such as network errors, calculate differences between timestamps; for gauges such as CPU utilization or queue depth, compare sustained levels and baselines.

## Correlate an incident

Use a disciplined timeline.

### 1. Anchor the event

Capture the application's exact error, request ID, client host, and timestamp with timezone. If only an approximate time is known, state the uncertainty.

### 2. Check database evidence

Use ASH/AWR/SQL Monitor and alert logs to identify:

- Dominant waits and SQL.
- Host CPU demand.
- Cell I/O latency or interconnect waits.
- Cluster and instance events.
- Database or ASM process identifiers worth finding in `Top`/`Ps`.

### 3. Check database-node ExaWatcher

Look across multiple samples for:

- Run queue exceeding available CPU capacity.
- System CPU, I/O wait, or steal time where applicable.
- Per-CPU imbalance.
- Paging, swap activity, memory reclaim, or OOM symptoms.
- A process whose CPU or memory rise matches the database evidence.
- Network errors, retransmits, drops, or sudden connection changes.

### 4. Check storage-cell ExaWatcher

Look for:

- Cell process CPU saturation.
- Device latency, utilization, or queueing changes.
- Flash-space or Smart I/O changes.
- RDS/fabric availability changes.
- Evidence isolated to one cell versus common across all cells.

### 5. Correlate, do not conflate

Examples:

- High DB time plus low host CPU can indicate waits rather than CPU starvation.
- High `iostat` device utilization alone does not prove a problem; compare latency, throughput, workload type, and baseline.
- A CPU spike after the application slowdown may be recovery work, not the trigger.
- Network counter values may be cumulative. Calculate deltas between samples rather than treating the total as an event count.

Document the evidence chain and label conclusions as confirmed, likely, or unproven.

## Build a useful baseline

Oracle recommends retaining ExaWatcher baselines. Capture representative periods, not arbitrary quiet time:

- Normal peak workload.
- Month-end or year-end processing.
- Batch and backup windows.
- Before a patch, upgrade, migration, or configuration change.
- After the change, using a comparable workload window.

For each baseline, preserve:

- Node and role.
- Exact start/end and timezone.
- Exadata image and configuration checksum.
- Workload description.
- ExaWatcher result checksum.
- Corresponding AWR baseline/export, SQL Monitor reports, and CellCLI evidence where relevant.

Store baselines outside the live collector retention area. The collector's space limit is for rolling diagnostics, not durable capacity planning.

## Make a controlled configuration change

Suppose an approved test requires a 10-second interval and a 6 GB space limit on a database server.

### 1. Capture current state

```bash
cd /opt/oracle.ExaWatcher
date -Is
./ExaWatcher.sh --lastconf
./ExaWatcher.sh --listcmd Full
systemctl status ExaWatcher --no-pager
cp -a ExaWatcher.conf "/var/tmp/ExaWatcher.conf.$(date +%Y%m%dT%H%M%S%z).bak"
sha256sum ExaWatcher.conf /var/tmp/ExaWatcher.conf.*.bak
```

### 2. Generate a candidate

```bash
candidate=/var/tmp/ExaWatcher.conf.candidate
./ExaWatcher.sh \
  --interval 10 \
  --spacelimit 6144 \
  --createconf "$candidate"

sed -n '1,240p' "$candidate"
diff -u ExaWatcher.conf "$candidate"
```

Always provide a candidate path. With no path, `--createconf` may overwrite the default configuration.

### 3. Review impact

Confirm:

- All expected collectors remain enabled.
- Per-collector minimum intervals are respected.
- No custom command changed unexpectedly.
- The result directory and compression remain correct.
- The space limit fits both retention and filesystem headroom.
- Oracle Support evidence requirements are still met.

### 4. Apply through the approved procedure

The exact service/config activation sequence depends on the installed image. Follow its unit definition, local help, and current Exadata maintenance documentation. Plan a rollback to the timestamped backup.

### 5. Verify

After activation:

```bash
systemctl is-active ExaWatcher
journalctl -u ExaWatcher --since '10 minutes ago' --no-pager
./ExaWatcher.sh --lastconf
./ExaWatcher.sh --listcmd Full
find /opt/oracle.ExaWatcher -maxdepth 2 -type f -mmin -10 -ls 2>/dev/null
```

Perform a short test extraction. Verify coverage before closing the change.

## Practice recovery

In a lab or approved maintenance window, practice these non-destructive scenarios:

1. Identify the active unit and effective configuration.
2. Detect stale output despite an active service.
3. Generate a 15-minute result bundle.
4. Verify its checksum and chart index.
5. Compare current configuration with a saved copy.
6. Identify which directories consume ExaWatcher space without deleting anything.
7. Restore a reviewed prior configuration through the release-specific procedure.

The goal is to make incident collection routine before a real outage.

## Production acceptance checklist

- [ ] ExaWatcher is present and healthy on every database and storage server.
- [ ] Service enablement matches the approved Exadata image standard.
- [ ] Output files are current, not merely the service status.
- [ ] Time synchronization and timezone handling are documented.
- [ ] Effective collectors and intervals are recorded per node role.
- [ ] Space limits preserve sufficient incident coverage and filesystem headroom.
- [ ] Result extraction has been tested on each node role.
- [ ] Secure transfer and checksum procedures are documented.
- [ ] Representative baselines are stored outside live retention.
- [ ] Pre/post-patch validation is included in the maintenance runbook.
- [ ] Manual deletion and arbitrary custom collectors require explicit approval.
