# Oracle ExaWatcher Administration and Storage Management

## Contents

1. [Administrative principles](#administrative-principles)
2. [Inventory and ownership](#inventory-and-ownership)
3. [Service administration](#service-administration)
4. [Configuration management](#configuration-management)
5. [Collector administration](#collector-administration)
6. [Storage model](#storage-model)
7. [Capacity and retention planning](#capacity-and-retention-planning)
8. [Storage-pressure runbook](#storage-pressure-runbook)
9. [Result archive management](#result-archive-management)
10. [Patch, upgrade, and reboot](#patch-upgrade-and-reboot)
11. [Security and audit](#security-and-audit)
12. [Change and rollback template](#change-and-rollback-template)

## Administrative principles

- Manage ExaWatcher as part of the Exadata image, not as an independent third-party package.
- Run installed tools as `root`, as required by Oracle documentation.
- Inspect every node separately. Database servers and storage servers have different defaults and collector sets.
- Preserve evidence before restart, retention reduction, configuration replacement, or cleanup.
- Treat local help, unit definition, and effective configuration as release-specific truth.
- Keep configuration changes small and reversible.

## Inventory and ownership

Capture an inventory on every node:

```bash
hostname -f
date -Is
timedatectl status
imageinfo -ver 2>/dev/null || true

systemctl status ExaWatcher --no-pager
systemctl is-enabled ExaWatcher
systemctl cat ExaWatcher
systemctl show ExaWatcher \
  -p FragmentPath -p DropInPaths -p User -p Group \
  -p ExecStart -p ExecStop -p EnvironmentFiles

stat /opt/oracle.ExaWatcher
find /opt/oracle.ExaWatcher -maxdepth 1 -type f \
  -printf '%M %u:%g %10s %TY-%Tm-%Td %TH:%TM %f\n' | sort

/opt/oracle.ExaWatcher/ExaWatcher.sh --lastconf
/opt/oracle.ExaWatcher/ExaWatcher.sh --listcmd Full
sha256sum /opt/oracle.ExaWatcher/ExaWatcher.conf
```

Do not normalize ownership or modes merely because nodes differ. First determine whether the difference is release-, role-, or patch-specific.

## Service administration

### Inspect

```bash
systemctl status ExaWatcher --no-pager
systemctl is-active ExaWatcher
systemctl is-enabled ExaWatcher
systemctl show ExaWatcher -p ActiveState -p SubState -p MainPID -p NRestarts
journalctl -u ExaWatcher --since '2 hours ago' --no-pager
```

### Start

Starting changes collection state:

```bash
systemctl start ExaWatcher
systemctl is-active ExaWatcher
journalctl -u ExaWatcher --since '5 minutes ago' --no-pager
```

Confirm that files are new or growing in the effective result directory.

### Stop

Stopping creates a diagnostic gap. Record the time and reason:

```bash
date -Is
systemctl stop ExaWatcher
systemctl is-active ExaWatcher
journalctl -u ExaWatcher --since '10 minutes ago' --no-pager
```

The collector also documents:

```bash
/opt/oracle.ExaWatcher/ExaWatcher.sh --stop
```

This stops the collector and its processes and archives data. Do not mix it with systemd lifecycle commands without first reading `systemctl cat ExaWatcher` and local help.

### Restart

A restart interrupts samples and can erase useful transient process state. Before restarting during an incident:

```bash
date -Is
systemctl status ExaWatcher --no-pager
journalctl -u ExaWatcher --since '1 hour ago' --no-pager \
  > /var/tmp/ExaWatcher.journal.before-restart.txt
/opt/oracle.ExaWatcher/ExaWatcher.sh --lastconf \
  > /var/tmp/ExaWatcher.lastconf.before-restart.txt
```

Then, in an approved window:

```bash
systemctl restart ExaWatcher
systemctl is-active ExaWatcher
journalctl -u ExaWatcher --since '5 minutes ago' --no-pager
```

### Enablement

Inspect enablement but do not change an appliance baseline casually:

```bash
systemctl is-enabled ExaWatcher
systemctl list-unit-files ExaWatcher.service
```

Use `systemctl enable` or `disable` only when the exact image's documented standard and an approved change require it. A service that fails after reboot may have an enablement issue, but a missing unit or script is an installation/patch integrity problem, not a reason to create an ad hoc unit.

## Configuration management

The documented Oracle Linux default is:

```text
/opt/oracle.ExaWatcher/ExaWatcher.conf
```

Confirm what systemd actually starts and whether a different file is passed with `--fromconf`.

### Back up

```bash
cd /opt/oracle.ExaWatcher
backup="/var/tmp/ExaWatcher.conf.$(date +%Y%m%dT%H%M%S%z).bak"
cp -a ExaWatcher.conf "$backup"
sha256sum ExaWatcher.conf "$backup"
```

### Inspect effective configuration

```bash
./ExaWatcher.sh --lastconf
./ExaWatcher.sh --listcmd Full
systemctl cat ExaWatcher
```

`--lastconf` does not collect data. It is preferable to assuming that the default file was most recently used.

### Generate a candidate

```bash
candidate=/var/tmp/ExaWatcher.conf.candidate

./ExaWatcher.sh \
  --interval 10 \
  --spacelimit 6144 \
  --zip bzip2 \
  --createconf "$candidate"

test -s "$candidate"
sed -n '1,260p' "$candidate"
diff -u ExaWatcher.conf "$candidate"
```

Never omit the candidate name unless overwriting the default configuration is explicitly intended. Oracle documents that `--createconf` without a file path can overwrite the default.

### Validate and activate

There is no universal, release-independent syntax-only activation command. Validate by:

1. Checking installed `--help`.
2. Reviewing the entire candidate and diff.
3. Confirming collector names using `--listcmd Full`.
4. Checking intervals against documented minimums.
5. Running the candidate only in an approved test/change window.
6. Verifying the effective config and file activity after activation.

`--fromconf FILE` runs collection; it is not a harmless linter:

```bash
./ExaWatcher.sh --fromconf "$candidate"
```

Use the installed unit's activation model and the current Exadata maintenance procedure. Retain the backup until a test extraction succeeds.

## Collector administration

### List collectors

```bash
./ExaWatcher.sh --listcmd Nameonly
./ExaWatcher.sh --listcmd Core
./ExaWatcher.sh --listcmd CMD
./ExaWatcher.sh --listcmd Full
```

### Select modules

Oracle documents:

```bash
./ExaWatcher.sh --commandmode ALL
./ExaWatcher.sh --commandmode CORE
./ExaWatcher.sh --commandmode SELECTED
```

Use these as part of a complete candidate configuration, not as isolated production experiments.

### Disable a collector

```bash
./ExaWatcher.sh --disable 'Vmstat' --createconf /var/tmp/ExaWatcher.no-vmstat.conf
```

Before disabling, determine:

- Which diagnostic questions become impossible.
- Whether Oracle Support expects that collector.
- Whether another collector overlaps sufficiently.
- Whether the real problem is output location, interval, or an invalid custom command.

### Change a core command

Only documented core collectors can have their commands changed. Example from Oracle documentation:

```bash
./ExaWatcher.sh \
  --command 'Vmstat;; "vmstat -a"' \
  --createconf /var/tmp/ExaWatcher.vmstat-a.conf
```

Review output volume, compatibility, and parsing/chart implications. A syntactically valid replacement can still break downstream expectations.

### Custom collector

Oracle documents `--customcmd`, but custom commands are high risk:

```bash
./ExaWatcher.sh \
  --customcmd 'Lsl;; "/bin/ls -l"' \
  --createconf /var/tmp/ExaWatcher.custom.conf
```

Require review for:

- Absolute executable paths and controlled arguments.
- Bounded runtime and output.
- No shell expansion of untrusted input.
- No credentials, private keys, tokens, SQL text, or personal data.
- No destructive or state-changing commands.
- Expected behavior under overlap when execution exceeds the interval.

## Storage model

ExaWatcher writes raw samples and rotated/compressed archives into its configured result/archive location. `GetExaWatcherResults.sh` creates a separate bundle and chart workspace for the requested time range.

Storage therefore has at least two consumers:

1. Live collector-managed data.
2. Administrator-generated result bundles, including repeated or abandoned extractions.

Keep generated results outside the installation filesystem when possible.

Oracle documents these defaults:

| Setting | Database server | Storage server |
| --- | ---: | ---: |
| Space limit | 6 GB | 600 MB |
| Default sampling interval | 5 seconds | 5 seconds |
| Default archive count | 720 | 720 |
| Default compression | `bzip2` | `bzip2` |

These are documented defaults, not proof of the effective local configuration.

## Capacity and retention planning

### Measure current use

```bash
df -h /opt /var/tmp
df -i /opt /var/tmp
du -xsh /opt/oracle.ExaWatcher
du -xh --max-depth=2 /opt/oracle.ExaWatcher 2>/dev/null | sort -h | tail -40
```

Find largest files and oldest/newest evidence:

```bash
find /opt/oracle.ExaWatcher -xdev -type f \
  -printf '%s %TY-%Tm-%Td %TH:%TM:%TS %p\n' 2>/dev/null \
  | sort -nr | head -40

find /opt/oracle.ExaWatcher -xdev -type f \
  -printf '%TY-%Tm-%Td %TH:%TM:%TS %s %p\n' 2>/dev/null \
  | sort | sed -n '1,20p;$p'
```

### Estimate headroom

Measure rather than assume compression ratio:

1. Record directory size at two points several hours apart under representative load.
2. Subtract any manually generated result bundles.
3. Calculate observed growth per hour/day.
4. Compare desired incident lookback with the configured limit and filesystem free space.
5. Leave operating-system and patching headroom outside the ExaWatcher budget.

Increasing the limit cannot create capacity. Verify filesystem headroom first.

### Tune in this order

1. Remove or relocate abandoned administrator-generated result bundles.
2. Correct an unintended result directory.
3. Enforce the supported `--spacelimit` in a reviewed configuration.
4. Review archive count and compression.
5. Increase expensive collectors' intervals only with diagnostic-impact review.
6. Disable collectors only as a last resort and with support requirements considered.

Do not treat `--count` as a time duration. Retention varies with collector volume and rotation behavior.

## Storage-pressure runbook

### 1. Preserve state

```bash
date -Is
systemctl status ExaWatcher --no-pager
df -h /opt /var/tmp
df -i /opt /var/tmp
./ExaWatcher.sh --lastconf
./ExaWatcher.sh --listcmd Full
```

### 2. Identify exact consumers

```bash
du -xsh /opt/oracle.ExaWatcher
du -xh --max-depth=2 /opt/oracle.ExaWatcher 2>/dev/null | sort -h | tail -40
find /opt/oracle.ExaWatcher -xdev -type f -size +100M \
  -printf '%12s %TY-%Tm-%Td %TH:%TM %p\n' 2>/dev/null | sort -nr
```

Separate live archives from generated result packages and unrelated `/opt` consumers.

### 3. Protect the incident window

If an active incident exists, package the smallest required window to a different filesystem with enough capacity and record its checksum before any cleanup.

### 4. Stop uncontrolled growth if necessary

If the filesystem is at immediate risk and the supported limit is not functioning, an approved temporary service stop may be safer than filesystem exhaustion:

```bash
systemctl stop ExaWatcher
```

This creates a data gap. Record the exact stop/start times and do not leave collection disabled.

### 5. Correct configuration

Generate a candidate with an appropriate space limit and validate all other settings. Do not lower the limit below the evidence already protected without understanding the release's purge behavior.

### 6. Handle manual cleanup cautiously

Manual deletion is destructive and release-sensitive. Before it:

- Obtain explicit approval for exact paths and age criteria.
- Stop or quiesce the relevant writer if required by the current procedure.
- Preserve required evidence elsewhere with checksums.
- List candidate files first and confirm they are archives/results, not scripts, configuration, or active files.
- Prefer moving approved candidates to a recoverable location on another filesystem before permanent deletion.
- Follow Oracle Support when collector-managed retention is malfunctioning.

Never use a broad target such as `/opt`, `/opt/oracle.ExaWatcher/*`, an unresolved variable, or a recursive wildcard.

### 7. Verify recovery

```bash
systemctl start ExaWatcher
systemctl is-active ExaWatcher
journalctl -u ExaWatcher --since '10 minutes ago' --no-pager
df -h /opt
df -i /opt
find /opt/oracle.ExaWatcher -maxdepth 2 -type f -mmin -10 -ls 2>/dev/null
```

Perform a short extraction and document the resulting retention gap.

## Result archive management

### Use a dedicated destination

```bash
result_root=/var/tmp/exawatcher-results
install -d -m 0700 "$result_root"
df -h "$result_root"
df -i "$result_root"
```

### Label results

Use a manifest containing node, role, local timezone, UTC equivalent, image release, requested interval, command, archive name, size, and checksum.

### Protect sensitive content

Process listings, open files, network data, hostnames, and custom collector output may expose sensitive operational information. Apply least-privilege file modes, approved transfer, retention, and ticket attachment rules.

### Avoid duplicates

Repeated extraction of the same broad interval can fill a small filesystem. Inventory existing result bundles before rerunning:

```bash
find /var/tmp/exawatcher-results -maxdepth 2 -type f \
  -printf '%TY-%Tm-%Td %TH:%TM %12s %p\n' 2>/dev/null | sort
```

## Patch, upgrade, and reboot

ExaWatcher is maintained with Exadata System Software. Do not independently replace its scripts or systemd unit.

### Before maintenance

```bash
date -Is
imageinfo -ver 2>/dev/null || true
systemctl status ExaWatcher --no-pager
systemctl is-enabled ExaWatcher
systemctl cat ExaWatcher > /var/tmp/ExaWatcher.unit.before.txt
./ExaWatcher.sh --lastconf > /var/tmp/ExaWatcher.lastconf.before.txt
./ExaWatcher.sh --listcmd Full > /var/tmp/ExaWatcher.commands.before.txt
cp -a ExaWatcher.conf "/var/tmp/ExaWatcher.conf.$(date +%Y%m%dT%H%M%S%z).bak"
sha256sum ExaWatcher.conf /var/tmp/ExaWatcher.conf.*.bak
```

- Extract a representative baseline and the immediate pre-maintenance window if required.
- Record result/archive coverage and free bytes/inodes.
- Confirm whether the current update procedure stops/starts the service automatically.

### After maintenance

```bash
date -Is
imageinfo -ver 2>/dev/null || true
systemctl status ExaWatcher --no-pager
systemctl is-enabled ExaWatcher
systemctl cat ExaWatcher > /var/tmp/ExaWatcher.unit.after.txt
journalctl -u ExaWatcher -b --no-pager
./ExaWatcher.sh --lastconf > /var/tmp/ExaWatcher.lastconf.after.txt
./ExaWatcher.sh --listcmd Full > /var/tmp/ExaWatcher.commands.after.txt

diff -u /var/tmp/ExaWatcher.unit.before.txt /var/tmp/ExaWatcher.unit.after.txt || true
diff -u /var/tmp/ExaWatcher.lastconf.before.txt /var/tmp/ExaWatcher.lastconf.after.txt || true
diff -u /var/tmp/ExaWatcher.commands.before.txt /var/tmp/ExaWatcher.commands.after.txt || true
```

Verify recent file activity and perform a short extraction. Review differences as possible release changes before restoring an old configuration over the new image.

## Security and audit

- Restrict ExaWatcher configuration, archives, and result bundles to approved administrators and support channels.
- Never put SSH passwords, keys, or tokens in `--scp`, custom commands, filenames, or shell history.
- Review custom commands for data exposure and command injection.
- Record every collection gap caused by stop, restart, configuration change, or storage incident.
- Record the original result checksum before extraction or analysis.
- Preserve source archives when transformed reports are shared.

## Change and rollback template

```text
Change:
Nodes and roles:
Exadata image:
Reason:
Incident evidence preserved:
Current config checksum:
Candidate config checksum:
Exact differences:
Expected CPU/I/O/storage effect:
Collection gap expected:
Activation procedure:
Validation commands:
Rollback file and procedure:
Start time / end time / timezone:
Result of short test extraction:
```

