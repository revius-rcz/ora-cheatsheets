---
name: oracle-unified-directory
description: Operate, inspect, patch, upgrade, monitor, and troubleshoot Oracle Unified Directory (OUD), including standalone and collocated instances, LDAP and LDAPS, dsconfig, dsreplication, backups, restores, logs, indexes, certificates, and Oracle Net directory naming. Use when managing an existing OUD environment, starting or stopping OUD, checking replication or health, applying an OUD bundle patch, planning a 12c-to-14c upgrade, diagnosing LDAP failures, or creating, changing, deleting, validating, and troubleshooting centrally stored TNS aliases under an Oracle Context.
---

# Oracle Unified Directory

Guide safe work in an existing Oracle Unified Directory environment. Assume the operator is new to OUD, but the environment and its availability requirements are production-sensitive.

## Workflow

1. Discover the deployment before changing it:
   - Record the OUD release and bundle patch, Java version, operating system, topology, server role, installation mode, `ORACLE_HOME`, instance path, domain path, ports, base DNs, Oracle Context DN, load balancers, and replication groups.
   - Distinguish a directory server from an OUD proxy, replication gateway, and OUDSM. Do not use directory-server data commands against a proxy without confirming support.
   - Distinguish standalone lifecycle commands from collocated WebLogic component commands.
2. Read only the reference needed for the task:
   - For concepts, environment discovery, the beginner learning path, TNS alias administration, lifecycle, backup, patching, upgrades, and acceptance checks, read [references/tutorial.md](references/tutorial.md).
   - For daily commands and short runbooks, read [references/cheatsheet.md](references/cheatsheet.md).
   - For symptom-led diagnosis, logs, replication failures, TLS, performance, and Oracle Net naming errors, read [references/troubleshooting.md](references/troubleshooting.md).
3. Start with read-only checks. Capture the failing client, endpoint, base DN, exact timestamp, complete error, and current replication state before restarting or editing.
4. Use the instance-local tools in `INSTANCE_DIR/OUD/bin` unless the deployment runbook identifies a different canonical path.
5. Use password files, a validated trust store, LDAPS or StartTLS, and a delegated administration identity. Do not put passwords in command history.
6. For a data or configuration change, show the pre-change export or backup, smallest LDIF or `dsconfig` change, expected replication behavior, verification, and rollback.
7. For patching or upgrades, follow the README and certification matrix for the exact release and patch. Treat examples as workflow templates, not substitutes for the patch README.

## Oracle Net naming rules

- Treat a “TNS alias in OUD” as an LDAP `orclNetService` entry, normally below `cn=OracleContext,<DEFAULT_ADMIN_CONTEXT>`. It is not a `tnsnames.ora` line stored verbatim as a file.
- Discover the real Oracle Context and schema before creating an entry. Do not create `cn=OracleContext`, import Oracle schema, or change its ACIs merely because a sample uses that DN.
- Search for the alias before adding it. LDAP DNs and most attribute matching are case-insensitive, so names that differ only by case are not separate aliases.
- Export the current entry before modifying or deleting it. Preserve line folding and the complete `orclNetDescString` when preparing rollback LDIF.
- Apply a replicated alias change once through one writable OUD endpoint. Verify it on at least two replicas; do not repeat the write independently on every replica.
- Use `SERVICE_NAME` unless the existing environment explicitly requires `SID`. Preserve RAC address lists, connect-time failover, load balancing, TCPS, wallets, and security parameters present in the approved descriptor.
- Validate both layers after a change: search the LDAP entry directly, then resolve and connect from a representative Oracle client configured with `NAMES.DIRECTORY_PATH` and `ldap.ora`.
- Do not confuse a net service name with an LDAP alias object. Prefer duplicate `orclNetService` entries unless the estate deliberately uses and supports `orclNetServiceAlias`.

## Safety rules

- Never run `import-ldif`, `restore`, `dsreplication initialize`, `rebuild-index`, or an upgrade merely to “repair” a server without identifying the authoritative data source and impact. These operations can replace data or require downtime.
- Never initialize a healthy replica from an unhealthy, stale, or unidentified source.
- Do not edit `config/config.ldif` while the server is running. Prefer `dsconfig`; use offline file repair only with a documented Oracle procedure and a recoverable copy.
- Do not use `--trustAll` as a production default. It is acceptable only for isolated diagnosis when the certificate fingerprint is independently verified; replace it with a trust store.
- Do not restart before collecting status, replication state, logs, disk usage, process data, and the failing request timestamp. A restart can destroy evidence and does not prove root cause.
- Do not patch a shared `ORACLE_HOME` until every OUD instance and dependent domain using that home is identified and stopped as required by the patch README.
- Do not call a major upgrade a patch. A 12c-to-14c upgrade has separate binaries, JDK and instance-upgrade steps, topology sequencing, and no in-place rollback; recovery means restoring the complete pre-upgrade environment.
- In a replicated topology, drain and maintain one service endpoint at a time when the exact patch or upgrade procedure permits rolling work. Require healthy replication and client failover before proceeding to the next node.
- Do not delete logs, backups, changelog data, or database files as a space-pressure first response. Identify the owner, retention policy, and recovery consequences.

## Response pattern

For an operational request:

1. State the assumed role, installation mode, release, endpoint, and affected suffix or Oracle Context.
2. Give read-only discovery and health checks first.
3. Mark commands as local/remote, online/offline, and standalone/collocated when that distinction matters.
4. Provide the smallest approved change with placeholders that cannot be mistaken for production values.
5. Explain availability impact, replication behavior, rollback, and success criteria.
6. Separate confirmed facts from release-dependent items that must be checked in Oracle documentation or the patch README.
