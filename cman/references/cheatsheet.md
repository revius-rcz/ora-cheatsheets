# Oracle Connection Manager Cheatsheet

## Contents

1. [Environment](#environment)
2. [Process model](#process-model)
3. [CMCTL command mode](#cmctl-command-mode)
4. [Minimal production-oriented cman.ora](#minimal-production-oriented-cmanora)
5. [Restrictive rule patterns](#restrictive-rule-patterns)
6. [Client tnsnames.ora](#client-tnsnamesora)
7. [Database registration with CMAN](#database-registration-with-cman)
8. [Runtime changes](#runtime-changes)
9. [Capacity](#capacity)
10. [ADR quick access](#adr-quick-access)
11. [Maintenance checklist](#maintenance-checklist)
12. [Security reminders](#security-reminders)

## Environment

```bash
export ORACLE_HOME=/u01/app/oracle/product/23ai/client_1
export TNS_ADMIN=/u01/app/oracle/network/admin
export PATH="$ORACLE_HOME/bin:$PATH"

command -v cmctl
cmctl show version
```

Default configuration file: `cman.ora` under `TNS_ADMIN`, Oracle base home network administration, or `$ORACLE_HOME/network/admin`, depending on release and home layout.

## Process model

```text
client -> CMAN listener -> CMGW -> database listener
                         CMADMIN
                            ^
                          CMCTL
```

| Name | Purpose |
| --- | --- |
| CMAN listener | Accept and filter connections. |
| `CMADMIN` | Supervise gateways/listener and serve control requests. |
| `CMGW` | Relay client/database traffic. |
| `CMCTL` | Administrative CLI. |
| `cmon` | CMAN control service used by CMCTL. |

## CMCTL command mode

Replace `CMAN01` with the entry name from `cman.ora`.

```bash
# Lifecycle
cmctl startup -c CMAN01
cmctl shutdown -c CMAN01
cmctl reload -c CMAN01

# Health and configuration
cmctl show version -c CMAN01
cmctl show status -c CMAN01
cmctl show services -c CMAN01
cmctl show gateways -c CMAN01
cmctl show parameters -c CMAN01
cmctl show rules -c CMAN01
cmctl show all -c CMAN01

# Connections
cmctl show connections count -c CMAN01
cmctl show connections detail -c CMAN01
cmctl show established connections detail for sales.example.com -c CMAN01
cmctl show idle connections count gt 30:00 -c CMAN01

# Registration statistics on supported releases
cmctl show stats -reg -c CMAN01
```

`SHUTDOWN` without a mode defaults to normal shutdown. For an explicit drain, use interactive CMCTL:

```text
CMCTL> ADMINISTER CMAN01
CMCTL> SHOW CONNECTIONS COUNT
CMCTL> SHUTDOWN NORMAL TIMEOUT 600 NOTIFY
```

Emergency only:

```text
CMCTL> SHUTDOWN ABORT
```

## Minimal production-oriented `cman.ora`

```text
CMAN01 =
  (CONFIGURATION =
    (ADDRESS=(PROTOCOL=TCP)(HOST=cman01.example.com)(PORT=1522))
    (RULE_LIST =
      (RULE=(SRC=10.20.10.0/24)(DST=db01.example.com)
            (SRV=sales.example.com)(ACT=accept))
      (RULE=(SRC=cman01.example.com)(DST=cman01.example.com)
            (SRV=cmon)(ACT=accept))
    )
    (PARAMETER_LIST =
      (DIAG_ADR_ENABLED=ON)
      (ADR_BASE=/u01/app/oracle)
      (MIN_GATEWAY_PROCESSES=2)
      (MAX_GATEWAY_PROCESSES=8)
      (MAX_CONNECTIONS=256)
      (INBOUND_CONNECT_TIMEOUT=60)
      (OUTBOUND_CONNECT_TIMEOUT=30)
      (CONNECTION_STATISTICS=YES)
    )
  )
```

## Restrictive rule patterns

Rules are ordered. The first matching rule wins.

```text
(RULE_LIST =
  # Block first.
  (RULE=(SRC=10.20.10.99)(DST=db01.example.com)(SRV=*)(ACT=drop))

  # Permit only the application service.
  (RULE=(SRC=10.20.10.0/24)(DST=db01.example.com)
        (SRV=sales.example.com)(ACT=accept)
        (ACTION_LIST=(MIT=1800)(MCT=28800)(MOCT=30)(CONN_STATS=yes)))

  # Local administration.
  (RULE=(SRC=cman01.example.com)(DST=cman01.example.com)
        (SRV=cmon)(ACT=accept))
)
```

| Field | Meaning |
| --- | --- |
| `SRC` | Client host, IP, `*`, or `/nn` subnet. |
| `DST` | Destination database host/IP. |
| `SRV` | Requested database service; `cmon` for CMCTL. |
| `ACT=accept` | Allow. |
| `ACT=reject` | Deny and return an error. |
| `ACT=drop` | Deny without returning an error. |
| `AUT` | Rule-level authentication filter override. |
| `CONN_STATS` | Rule-level connection statistics. |
| `MCT` | Maximum connection/session time, seconds. |
| `MIT` | Maximum idle time, seconds. |
| `MOCT` | Maximum outbound connect time, seconds. |

Do not use partial IP wildcards such as `10.20.*`. Use a complete `*`, exact address, host name, or CIDR-style subnet.

## Client `tnsnames.ora`

```text
SALES_VIA_CMAN =
  (DESCRIPTION =
    (SOURCE_ROUTE = YES)
    (ADDRESS_LIST =
      (ADDRESS=(PROTOCOL=TCP)(HOST=cman01.example.com)(PORT=1522))
      (ADDRESS=(PROTOCOL=TCP)(HOST=db01.example.com)(PORT=1521))
    )
    (CONNECT_DATA=(SERVICE_NAME=sales.example.com))
  )
```

```bash
tnsping SALES_VIA_CMAN
sqlplus /nolog
```

```text
SQL> connect app_user@SALES_VIA_CMAN
```

## Database registration with CMAN

Database-host `tnsnames.ora`:

```text
CMAN_LISTENER =
  (ADDRESS=(PROTOCOL=TCP)(HOST=cman01.example.com)(PORT=1522))
```

CMAN `PARAMETER_LIST`:

```text
(VALID_NODE_CHECKING_REGISTRATION=ON)
(REGISTRATION_INVITED_NODES=db01.example.com,10.20.20.11)
```

Inspect first:

```sql
SHOW PARAMETER local_listener
SHOW PARAMETER remote_listener
SHOW PARAMETER listener_networks
```

Simple non-RAC setup only:

```sql
ALTER SYSTEM SET REMOTE_LISTENER='CMAN_LISTENER' SCOPE=BOTH;
ALTER SYSTEM REGISTER;
```

Verify:

```bash
cmctl show services -c CMAN01
cmctl show stats -reg -c CMAN01
```

Never overwrite an existing RAC SCAN `REMOTE_LISTENER` value without redesigning the registration topology.

## Runtime changes

```bash
# Runtime only; lost at CMAN shutdown.
cmctl set connection_statistics yes -c CMAN01
cmctl set idle_timeout 1800 -c CMAN01
cmctl set inbound_connect_timeout 60 -c CMAN01
cmctl set outbound_connect_timeout 30 -c CMAN01
cmctl set trace_level user -c CMAN01
cmctl set trace_level off -c CMAN01
```

To persist a setting:

1. Edit `cman.ora`.
2. `cmctl reload -c CMAN01`.
3. Verify with `SHOW PARAMETERS` and `SHOW RULES`.

## Capacity

```text
upper-bound connection slots ~= MAX_CONNECTIONS * gateway count
```

Check all layers before increasing it:

- CMAN gateway current/refused counts.
- CMAN host CPU, memory, file descriptors, and sockets.
- Database listener handlers.
- Database `PROCESSES`, `SESSIONS`, shared-server settings, and application pool size.

## ADR quick access

```bash
adrci
```

```text
ADRCI> show homes
ADRCI> set home diag/netcman/<host>/<cman_instance>
ADRCI> show alert -tail 100 -term
ADRCI> show incident
```

With ADR enabled, do not rely on non-ADR `LOG_DIRECTORY`/`TRACE_DIRECTORY` settings.

## Maintenance checklist

### Precheck

```bash
cmctl show version -c CMAN01
cmctl show status -c CMAN01
cmctl show services -c CMAN01
cmctl show gateways -c CMAN01
cmctl show connections count -c CMAN01
cmctl show rules -c CMAN01
cmctl show parameters -c CMAN01
```

- Record Oracle home, `TNS_ADMIN`, owner, and config checksum.
- Back up network configuration and wallets securely.
- Verify another CMAN endpoint before taking one down.
- Remove the endpoint from rotation and drain.

### Postcheck

```bash
cmctl startup -c CMAN01
cmctl show version -c CMAN01
cmctl show status -c CMAN01
cmctl show services -c CMAN01
cmctl show gateways -c CMAN01
cmctl show rules -c CMAN01
cmctl show parameters -c CMAN01
```

- Test one real client alias through CMAN.
- Confirm the expected database service.
- Monitor refused connections and ADR errors.
- Restore endpoint traffic gradually.

## Security reminders

- Use LOSA and run CMCTL locally as the CMAN owner; password administration is desupported in 26ai.
- Restrict the `cmon` rule.
- Do not deploy wildcard production allow rules.
- For client IP forwarding, configure both `ENABLE_IP_FORWARDING` and database-side `TCP.ALLOWED_PROXIES`.
- For TCPS, secure and test both client-to-CMAN and CMAN-to-database legs.
- Keep `SUPPORT` tracing temporary.

## Official references

- [CMCTL command reference](https://docs.oracle.com/en/database/oracle/oracle-database/26/netrf/oracle-connection-manager-control-utility.html)
- [`cman.ora` parameter reference](https://docs.oracle.com/en/database/oracle/oracle-database/26/netrf/oracle-connection-manager-parameters.html)
- [Starting and stopping CMAN](https://docs.oracle.com/en/database/oracle/oracle-database/26/netag/using-oracle-connection-manager-control-utility-administer-oracle-connection-manager.html)
