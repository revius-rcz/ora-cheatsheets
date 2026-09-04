# Oracle ExaWatcher Troubleshooting Runbook

## Contents

1. [Triage principles](#triage-principles)
2. [Capture current state](#capture-current-state)
3. [Service is missing](#service-is-missing)
4. [Service does not start](#service-does-not-start)
5. [Service is active but data is stale](#service-is-active-but-data-is-stale)
6. [A collector repeatedly fails](#a-collector-repeatedly-fails)
7. [ExaWatcher uses excessive CPU or I/O](#exawatcher-uses-excessive-cpu-or-io)
8. [Storage is full or growing unexpectedly](#storage-is-full-or-growing-unexpectedly)
9. [Result extraction fails or returns no data](#result-extraction-fails-or-returns-no-data)
10. [Charts are absent or do not render](#charts-are-absent-or-do-not-render)
11. [Timestamps do not align across nodes](#timestamps-do-not-align-across-nodes)
12. [Data differs between nodes](#data-differs-between-nodes)
13. [Failure after patching or reboot](#failure-after-patching-or-reboot)
14. [Escalation package](#escalation-package)
15. [Recovery and validation](#recovery-and-validation)

## Triage principles

Diagnose in this order:

1. Node identity, role, image release, timezone, and incident time.
2. Service unit existence and effective command.
3. Service state and journal.
4. Effective ExaWatcher configuration.
5. Collector processes.
6. Actual output freshness.
7. Filesystem bytes, inodes, permissions, and compression tools.
8. Result extraction inputs and destination.

Do not begin with restart, reinstallation, or deletion. These actions can remove evidence or create a diagnostic gap without proving the cause.

## Capture current state

Run as `root` and preserve the output:

```bash
case_dir="/var/tmp/exawatcher-triage.$(date +%Y%m%dT%H%M%S%z)"
install -d -m 0700 "$case_dir"

{
  hostname -f
  date -Is
  timedatectl status
  imageinfo -ver 2>/dev/null || true
} > "$case_dir/identity.txt" 2>&1

systemctl status ExaWatcher --no-pager \
  > "$case_dir/systemctl-status.txt" 2>&1 || true
systemctl cat ExaWatcher \
  > "$case_dir/systemctl-unit.txt" 2>&1 || true
systemctl show ExaWatcher \
  > "$case_dir/systemctl-show.txt" 2>&1 || true
journalctl -u ExaWatcher --since '4 hours ago' --no-pager \
  > "$case_dir/journal.txt" 2>&1 || true

/opt/oracle.ExaWatcher/ExaWatcher.sh --lastconf \
  > "$case_dir/lastconf.txt" 2>&1 || true
/opt/oracle.ExaWatcher/ExaWatcher.sh --listcmd Full \
  > "$case_dir/listcmd-full.txt" 2>&1 || true

ps -efww > "$case_dir/ps.txt"
df -h /opt /var/tmp > "$case_dir/df-bytes.txt" 2>&1
df -i /opt /var/tmp > "$case_dir/df-inodes.txt" 2>&1
du -xsh /opt/oracle.ExaWatcher > "$case_dir/du-total.txt" 2>&1 || true

find /opt/oracle.ExaWatcher -maxdepth 3 -type f \
  -printf '%TY-%Tm-%Td %TH:%TM:%TS %12s %u:%g %m %p\n' 2>/dev/null \
  | sort > "$case_dir/files.txt"
```

This creates files but does not modify ExaWatcher. Review them for credentials or sensitive process information before sharing.

## Service is missing

### Symptoms

- `Unit ExaWatcher.service could not be found`.
- `/opt/oracle.ExaWatcher` is absent.
- The service exists on peers but not one node.

### Checks

```bash
systemctl list-unit-files | grep -i exawatcher || true
find /etc/systemd /usr/lib/systemd -type f -iname '*ExaWatcher*' -print 2>/dev/null
ls -ld /opt/oracle.ExaWatcher 2>&1
imageinfo -ver 2>/dev/null || true
```

Compare only with a peer of the same node role and exact image release:

```bash
systemctl cat ExaWatcher
sha256sum /opt/oracle.ExaWatcher/ExaWatcher.sh \
  /opt/oracle.ExaWatcher/GetExaWatcherResults.sh
```

Use checksums to establish a mismatch, not to justify copying files.

### Corrective direction

- Confirm whether the image/release should include ExaWatcher.
- Review the Exadata update log and rollback state.
- Restore the release-consistent component using the current Exadata patching or Oracle Support procedure.
- Do not create a hand-written unit or copy another server's directory.

## Service does not start

### Checks

```bash
systemctl status ExaWatcher --no-pager -l
journalctl -u ExaWatcher -b --no-pager
systemctl cat ExaWatcher
systemctl show ExaWatcher -p ExecStart -p User -p Group -p EnvironmentFiles

namei -l /opt/oracle.ExaWatcher/ExaWatcher.sh
ls -l /opt/oracle.ExaWatcher/ExaWatcher.conf
df -h /opt
df -i /opt
```

Run version/help inspection only; do not manually launch a second collector while systemd is repeatedly trying to start it:

```bash
/opt/oracle.ExaWatcher/ExaWatcher.sh --help
/opt/oracle.ExaWatcher/ExaWatcher.sh --lastconf
/opt/oracle.ExaWatcher/ExaWatcher.sh --listcmd Full
```

### Common causes

| Evidence | Likely area | Corrective direction |
| --- | --- | --- |
| Script or config not found | incomplete/mismatched patch | Repair through image-consistent procedure. |
| Permission denied | ownership, mode, mount options, SELinux | Compare with same-release peer and correct through approved OS procedure. |
| No space left | bytes or inodes exhausted | Follow storage-pressure runbook; preserve evidence first. |
| Unknown option/collector | old config on new release or wrong script | Diff pre/post-patch config and local help. |
| Invalid result directory | missing mount/path or wrong permissions | Restore the intended destination safely. |
| Rapid restart loop | child exits on configuration/tool failure | Stop the loop if needed, preserve journal, fix root cause, then start once. |

Do not repeatedly restart. Each attempt can rotate logs and consume more space.

## Service is active but data is stale

### Prove staleness

Find the configured result location from `--lastconf`, `--listcmd Full`, and the unit. Then:

```bash
systemctl show ExaWatcher -p ActiveState -p SubState -p MainPID -p ControlGroup
pgrep -af 'ExaWatcher|iostat|vmstat|mpstat|cellsrvstat' || true

find /opt/oracle.ExaWatcher -maxdepth 3 -type f -mmin -20 \
  -printf '%TY-%Tm-%Td %TH:%TM:%TS %12s %p\n' 2>/dev/null | sort
```

If no file is recent, compare current time with newest file time and confirm timezone.

### Diagnose

```bash
journalctl -u ExaWatcher --since '2 hours ago' --no-pager
df -h /opt
df -i /opt
mount | grep ' /opt ' || true

namei -l /opt/oracle.ExaWatcher
./ExaWatcher.sh --lastconf
./ExaWatcher.sh --listcmd Full
```

Possible causes:

- The systemd wrapper remains active after collector children exited.
- The configured result directory differs from the directory inspected.
- Filesystem is full, read-only, unmounted, or out of inodes.
- A compression or rotation subprocess is stuck.
- All expected collectors are disabled or outside their configured start/end window.
- A custom command hangs and prevents expected progress.
- Clock/timezone changed, making files look stale.

Capture blocked process state and journal before an approved restart.

## A collector repeatedly fails

Identify the collector and exact command:

```bash
cd /opt/oracle.ExaWatcher
./ExaWatcher.sh --listcmd Full
journalctl -u ExaWatcher --since '2 hours ago' --no-pager
```

Check:

- Executable exists and matches the installed image.
- Command options are supported by the local OS tool.
- Required privilege and device/interface exist on this node role.
- Collector respects its minimum interval.
- Output directory is writable and has space/inodes.
- Compression utility selected by `--zip` exists.
- Custom command terminates within its interval and does not wait for input.

Do not silently disable a failing collector. First preserve the error, assess diagnostic loss, and determine whether the configuration was copied from a different release or node role.

For a temporary approved configuration, generate a candidate rather than editing live:

```bash
./ExaWatcher.sh \
  --disable 'CollectorName' \
  --createconf /var/tmp/ExaWatcher.without-collector.conf
```

Restore the collector once the underlying issue is corrected.

## ExaWatcher uses excessive CPU or I/O

### Establish attribution

```bash
systemctl show ExaWatcher -p MainPID -p ControlGroup
systemd-cgtop --depth=3 --iterations=3 2>/dev/null || true
ps -eo pid,ppid,etimes,pcpu,pmem,stat,args --sort=-pcpu | head -40
```

Compare process start times and commands with `--listcmd Full`. Determine whether load comes from:

- A supported collector with an interval too short for the host.
- A collector that hangs or overlaps with its next run.
- A custom command.
- Compression or a broad `GetExaWatcherResults.sh` extraction.
- Unrelated work merely containing `ExaWatcher` in its path.

### Corrective direction

1. Stop or narrow an in-progress broad extraction if it is the cause and the user authorizes interruption.
2. Remove or correct unsafe custom collectors through a reviewed candidate configuration.
3. Increase the affected group interval while respecting the diagnostic-resolution requirement.
4. Disable a heavy optional collector only after impact review.
5. Escalate repeatable high overhead in default configuration with release, logs, process data, and configuration.

Do not renice, kill random children, or change collector commands in place. The service may respawn them and obscure the cause.

## Storage is full or growing unexpectedly

### Inspect bytes and inodes

```bash
df -h /opt /var/tmp
df -i /opt /var/tmp
du -xsh /opt/oracle.ExaWatcher
du -xh --max-depth=2 /opt/oracle.ExaWatcher 2>/dev/null | sort -h | tail -40
```

### Find large/rapidly changing files

```bash
find /opt/oracle.ExaWatcher -xdev -type f \
  -printf '%s %TY-%Tm-%Td %TH:%TM:%TS %p\n' 2>/dev/null \
  | sort -nr | head -40

find /opt/oracle.ExaWatcher -xdev -type f -mmin -30 \
  -printf '%TY-%Tm-%Td %TH:%TM:%TS %12s %p\n' 2>/dev/null | sort
```

### Determine why the limit did not protect the filesystem

- Is the effective `--spacelimit` different from the default file?
- Are large files administrator-generated result bundles outside collector-managed retention?
- Is output written to a different mount than expected?
- Is compression failing?
- Is one custom collector producing abnormal volume?
- Is the filesystem consumed by something outside ExaWatcher?
- Did a patch/configuration change reset the limit or result directory?

### Recovery

Protect the incident window first. Prefer supported space-limit correction and relocation of generated result bundles. If filesystem exhaustion is imminent, an approved temporary service stop may prevent further damage, but creates a data gap.

Do not provide or run a broad delete command. Manual cleanup requires exact candidate paths, explicit approval, and an image/release-aware procedure.

## Result extraction fails or returns no data

### Validate syntax and scope

```bash
cd /opt/oracle.ExaWatcher
./GetExaWatcherResults.sh --help
date -Is
timedatectl status
```

Documented example:

```bash
./GetExaWatcherResults.sh \
  --from 08/26/2026_10:00:00 \
  --to   08/26/2026_10:30:00 \
  --resultdir /var/tmp/exawatcher-results
```

### Check archive coverage

```bash
find /opt/oracle.ExaWatcher -maxdepth 3 -type f \
  -printf '%TY-%Tm-%Td %TH:%TM:%TS %12s %p\n' 2>/dev/null | sort | tail -100
```

### Check destination

```bash
namei -l /var/tmp/exawatcher-results
df -h /var/tmp/exawatcher-results
df -i /var/tmp/exawatcher-results
```

### Frequent causes

| Symptom | Check |
| --- | --- |
| No matching data | wrong node, local/UTC confusion, expired retention, date syntax |
| Immediate argument error | installed `--help`, Unicode dash copied from prose, quoting |
| Partial archive | insufficient destination space/inodes, interrupted extraction |
| Permission error | root execution, destination ownership/mode, mount options |
| Slow/high load | requested range too broad, chart generation/compression, busy node |
| Copy failure | network/DNS/SSH policy, destination, credentials; use approved transfer |

Begin with 15–30 minutes. If that works, expand incrementally.

## Charts are absent or do not render

Oracle documents charts for I/O, CPU, CPU detail, cell statistics, and alert history. They are a subset of the result.

Check:

1. The result archive completed successfully.
2. The archive was extracted locally, not browsed inside a compressed file.
3. The expected path exists:

   ```text
   Charts.ExaWatcher.<hostname>/<timestamp>_<duration>/index.html
   ```

4. Source collectors were enabled and had data in the selected range.
5. The local browser can load required resources; Oracle documents Internet access for viewing the HTML pages.
6. Browser security policy did not block local content.

If charts remain unavailable, inspect raw files. Do not rerun a much larger extraction merely to make charts appear.

## Timestamps do not align across nodes

Capture on every node:

```bash
hostname -f
date -Is
date -u -Is
timedatectl status
chronyc tracking 2>/dev/null || true
chronyc sources -v 2>/dev/null || true
```

Check for:

- Different timezones or DST offsets.
- Clock step/slew during the incident.
- NTP/chrony loss.
- Application logs in UTC and host logs in local time.
- Archive names generated before or after a timezone change.

Build a UTC timeline but retain each source's original timestamp and offset. Never silently add an offset without documenting it.

## Data differs between nodes

Some differences are expected by role, hardware generation, and image release. Compare:

```bash
imageinfo -ver
./ExaWatcher.sh --lastconf
./ExaWatcher.sh --listcmd Full
systemctl cat ExaWatcher
sha256sum ExaWatcher.conf
```

Classify differences:

- Expected node-role collectors (`CellSrvStat` on storage cells).
- Hardware-specific collectors or minimum intervals.
- Release changes.
- Approved local customization.
- Configuration drift.
- Failed/partial patch.

Do not clone a configuration from a storage cell to a database server or between different releases.

## Failure after patching or reboot

### Compare pre/post evidence

```bash
imageinfo -ver 2>/dev/null || true
systemctl status ExaWatcher --no-pager
systemctl is-enabled ExaWatcher
systemctl cat ExaWatcher
journalctl -u ExaWatcher -b --no-pager
./ExaWatcher.sh --help
./ExaWatcher.sh --lastconf
./ExaWatcher.sh --listcmd Full
```

Compare with saved unit, config, collector inventory, and checksums.

Common post-maintenance patterns:

- Service was not enabled or failed early at boot.
- Old configuration uses an option or collector changed by the new release.
- Result mount was not available when the service started.
- Permissions/labels changed.
- Update was incomplete or rolled back partially.
- Service is healthy but no post-boot file activity was verified.

Do not overwrite new-release files with backups wholesale. Restore only reviewed intended settings using the new release's syntax.

## Escalation package

Provide a concise package with:

- Host names and node roles.
- Exadata System Software release.
- Incident start/end, local timezone, and UTC equivalent.
- Exact symptom and operational impact.
- Service status, unit definition, journal, and process list.
- `--lastconf` and `--listcmd Full` output.
- Filesystem byte/inode state and directory size inventory.
- Recent file list showing staleness or growth.
- Exact extractor command and error output.
- Smallest ExaWatcher result window covering the incident, from all relevant nodes.
- Corresponding AWR/ASH/SQL Monitor, alert logs, and CellCLI evidence as applicable.
- Changes, patch/reboot times, and configuration diffs.

Do not include credentials. Review process, open-file, and network output for sensitive information before upload.

## Recovery and validation

After any correction:

```bash
date -Is
systemctl is-active ExaWatcher
systemctl status ExaWatcher --no-pager
journalctl -u ExaWatcher --since '10 minutes ago' --no-pager
./ExaWatcher.sh --lastconf
./ExaWatcher.sh --listcmd Full
df -h /opt
df -i /opt
find /opt/oracle.ExaWatcher -maxdepth 3 -type f -mmin -10 -ls 2>/dev/null
```

Then generate a short result bundle and verify:

- Requested interval is covered.
- Raw collector files are present.
- Expected chart directory is present when supported.
- Archive checksum is recorded.
- CPU/I/O overhead is acceptable.
- Retention and filesystem headroom are acceptable.
- Any collection gap is documented.

