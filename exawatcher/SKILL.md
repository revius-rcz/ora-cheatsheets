---
name: oracle-exawatcher
description: Operate, configure, size, extract, analyze, and troubleshoot Oracle ExaWatcher on Exadata database and storage servers. Use when working with the ExaWatcher systemd service, /opt/oracle.ExaWatcher, ExaWatcher.sh, ExaWatcher.conf, GetExaWatcherResults.sh, ExaWatcher archives or charts, collectors such as Iostat, Vmstat, Mpstat, Netstat, Top, or CellSrvStat, incident evidence collection, baseline capture, disk-space limits, retention, post-patch validation, or missing and stale ExaWatcher data.
---

# Oracle ExaWatcher

Use this skill to administer ExaWatcher safely and to collect evidence without destroying the incident data being investigated. ExaWatcher runs independently on every Exadata server, so always identify the affected nodes and time window before extracting results.

## Workflow

1. Establish the Exadata System Software release, node role, host name, timezone, incident window, available space, and whether the task is inspection, extraction, configuration, lifecycle administration, or troubleshooting.
2. Inspect before changing anything:

   ```bash
   hostname -f
   date -Is
   imageinfo -ver 2>/dev/null || true
   systemctl status ExaWatcher --no-pager
   systemctl is-enabled ExaWatcher
   df -h /opt /var/tmp
   df -i /opt /var/tmp
   /opt/oracle.ExaWatcher/ExaWatcher.sh --lastconf
   /opt/oracle.ExaWatcher/ExaWatcher.sh --listcmd Full
   ```

3. Read only the references needed for the request:

   - For a fast operational summary, read [references/cheatsheet.md](references/cheatsheet.md).
   - For a guided first investigation and baseline workflow, read [references/tutorial.md](references/tutorial.md).
   - For service, configuration, patching, retention, and storage tasks, read [references/administration.md](references/administration.md).
   - For symptom-driven diagnosis and recovery, read [references/troubleshooting.md](references/troubleshooting.md).
   - For option and collector syntax, read [references/command-reference.md](references/command-reference.md), then confirm it with the installed script's `--help` because options vary by image release.
   - For authoritative documentation used to build this skill, read [references/sources.md](references/sources.md).

4. Collect the smallest useful time range from every relevant node. Add a modest buffer around the incident, normally 10–15 minutes on each side.
5. Preserve the original archive and record its host, timezone, time range, size, and checksum before analysis or transfer.
6. Correlate ExaWatcher with database, cell, and application evidence. ExaWatcher is high-frequency host and network evidence; it does not replace AWR/ASH, SQL Monitor, CellCLI metrics, alert logs, or application traces.

## Safety rules

- Run the installed ExaWatcher tools as `root`, as required by Oracle documentation. Do not change their ownership, permissions, interpreter, or embedded commands to bypass privilege requirements.
- Treat the installed release as authoritative. Check `ExaWatcher.sh --help`, `GetExaWatcherResults.sh --help`, `--lastconf`, and `--listcmd` before using options copied from another Exadata image.
- Do not edit `/opt/oracle.ExaWatcher/ExaWatcher.conf` in place without a timestamped backup and a tested rollback. Generate and validate a candidate configuration separately.
- Do not overwrite the default configuration accidentally: `--createconf` with no file name may overwrite it.
- Do not disable ExaWatcher, reduce retention, or delete archives until required incident evidence is packaged elsewhere.
- Do not run broad recursive deletion commands in `/opt`. Under storage pressure, identify the exact archive/result directory, package required evidence, use the supported space limit, and obtain explicit approval for any manual deletion.
- Do not place result archives on a nearly full root filesystem. Check both bytes and inodes, and choose a destination with enough capacity for the archive plus chart-generation workspace.
- Avoid long or broad extractions. A multi-day range on every node can consume significant CPU, I/O, and space.
- Do not add arbitrary custom collectors to production. Review command safety, runtime, output volume, credentials, and sensitive-data exposure first.
- Do not copy ExaWatcher scripts from a different server to repair a missing or mismatched installation. Restore the release-consistent software through the approved Exadata patching or support procedure.
- Use service stop/restart only when the requested administrative change requires it. Capture status, logs, configuration, and recent file activity first during an incident.

## Response pattern

For an operational request:

1. State the node scope, timezone, time window, and release assumptions.
2. Show read-only checks first.
3. Show the smallest safe command or configuration change.
4. Explain operational impact, rollback, storage effect, and validation.
5. Mark commands that stop collection, restart the service, alter retention, or remove data.
6. Separate confirmed facts from hypotheses when interpreting performance data.

