# Authoritative Sources

Last reviewed: 2026-08-26.

Use the documentation for the installed Exadata System Software release when it differs from these current pages.

## Oracle documentation

- [ExaWatcher overview](https://docs.oracle.com/en/engineered-systems/exadata-database-machine/sagug/exawatcher.html) — purpose, per-server collection, typical five-second granularity, extraction, charts, and baseline guidance.
- [System diagnostics gathering with sosreport and ExaWatcher](https://docs.oracle.com/en/engineered-systems/exadata-database-machine/sagug/system-diagnostics-data-gathering-sosreports-oracle-exawatcher.html) — collector descriptions, service start, `ExaWatcher.sh` options, configuration, intervals, space limits, archive count, result directory, stop, and compression.
- [About ExaWatcher Charts](https://docs.oracle.com/en/engineered-systems/exadata-database-machine/dbmmn/exawatcher-charts.html) — `GetExaWatcherResults.sh` example, result archive, SCP option, chart content, and browser workflow.
- [Using ExaWatcher Charts to Monitor Oracle Exadata Performance](https://docs.oracle.com/en/engineered-systems/exadata-database-machine/dbmmn/exawatcher-charts1.html) — chart categories and the origin of I/O, CPU, CPU-detail, and cell statistics.

## Local sources of truth

Run these on the target node before a version-sensitive change:

```bash
imageinfo -ver
/opt/oracle.ExaWatcher/ExaWatcher.sh --help
/opt/oracle.ExaWatcher/ExaWatcher.sh --lastconf
/opt/oracle.ExaWatcher/ExaWatcher.sh --listcmd Full
/opt/oracle.ExaWatcher/GetExaWatcherResults.sh --help
systemctl cat ExaWatcher
```

The installed help and configuration win over examples from a different image release.

