---
name: oracle-connection-manager
description: Configure, operate, secure, patch, and troubleshoot Oracle Connection Manager (CMAN), including cman.ora access rules, CMCTL administration, client source routing, database service registration, connection monitoring, ADR diagnostics, maintenance and restart procedures, and version-aware guidance for classic CMAN, CMAN Traffic Director Mode, and tunneling. Use when working with Oracle Net proxy routing, CMAN listener or gateway processes, cman.ora, cmctl, CMADMIN, CMGW, cmon rules, SOURCE_ROUTE, or database connections through an Oracle Connection Manager host.
---

# Oracle Connection Manager

Use this skill to guide safe administration of Oracle Connection Manager (CMAN). Treat classic CMAN, Traffic Director Mode (TDM), and tunneling as distinct operating modes. Default to classic CMAN unless the user explicitly needs TDM connection pooling/application continuity features or reverse-connection tunneling.

## Workflow

1. Establish the Oracle Database and CMAN release, operating system, Oracle home, configuration location, CMAN instance name, topology, listener ports, database services, and client source networks.
2. Identify the requested mode:
   - Use classic CMAN for Oracle Net proxying, access control, protocol routing, and optional shared-server multiplexing.
   - Use TDM only after confirming driver/database compatibility, proxy-user and wallet requirements, and whether pooled or dedicated mode is intended.
   - Use tunneling only for a reverse-connect topology that requires paired client and server CMAN instances.
3. Read the smallest relevant reference:
   - For architecture, installation planning, first configuration, client routing, service registration, security, HA, patching, and upgrades, read [references/tutorial.md](references/tutorial.md).
   - For daily commands, configuration templates, and checklists, read [references/cheatsheet.md](references/cheatsheet.md).
   - For incident triage, ADR, connection failures, rule problems, registration issues, and capacity diagnosis, read [references/troubleshooting.md](references/troubleshooting.md).
4. Prefer read-only inspection before proposing a change.
5. Preserve the current `cman.ora`, `tnsnames.ora`, `sqlnet.ora`, and database listener registration settings before modification.
6. Apply least-privilege rules and keep CMCTL administration local through Local Operating System Authentication (LOSA).
7. Validate the effective runtime state with CMCTL after start or reload; do not assume that a syntactically plausible file is active.

## Safety rules

- Never replace an existing RAC `REMOTE_LISTENER` value blindly. Inspect `REMOTE_LISTENER`, `LOCAL_LISTENER`, and `LISTENER_NETWORKS`, then integrate CMAN without breaking SCAN registration.
- Never expose a broad `(SRC=*)(DST=*)(SRV=*)(ACT=accept)` rule as a production default.
- Put narrow rules before broad rules. CMAN applies the action from the first matching rule.
- Include a dedicated `SRV=cmon` administration rule. Restrict its source and destination to the CMAN host or approved administration network.
- Prefer `SHUTDOWN NORMAL` and connection draining for maintenance. Use `SHUTDOWN ABORT` only when immediate termination and loss of active connections are explicitly acceptable.
- Treat `SET` changes in CMCTL as runtime-only. Persist intended changes in `cman.ora`, use `RELOAD` where supported, and verify with `SHOW PARAMETERS` and `SHOW RULES`.
- Do not enable tracing at `SUPPORT` level for longer than necessary; it can generate substantial data.
- Do not claim that classic CMAN multiplexes dedicated-server sessions. Classic multiplexing requires a shared-server configuration with multiplexing enabled; TDM pooling is a separate feature.
- Confirm parameter and command support against the exact Oracle release before using newer features such as REST administration, IP/service connection-rate controls, client IP forwarding, `STARTUP -MIGRATE`, TDM, or tunneling.

## Response pattern

When answering an operational request:

1. State assumptions and the affected connection path.
2. Show read-only checks first.
3. Show the smallest configuration or command change.
4. Explain expected impact, reload/restart requirements, rollback, and verification.
5. Separate classic CMAN instructions from TDM or tunneling instructions.

