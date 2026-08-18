# Oracle Connection Manager Tutorial

## Contents

1. [Scope and release assumptions](#scope-and-release-assumptions)
2. [What CMAN does](#what-cman-does)
3. [Architecture](#architecture)
4. [Plan the deployment](#plan-the-deployment)
5. [Locate the software and configuration](#locate-the-software-and-configuration)
6. [Build a minimal secure classic CMAN configuration](#build-a-minimal-secure-classic-cman-configuration)
7. [Configure a client route](#configure-a-client-route)
8. [Start and verify CMAN](#start-and-verify-cman)
9. [Add database service registration](#add-database-service-registration)
10. [Understand rules](#understand-rules)
11. [Enable TLS deliberately](#enable-tls-deliberately)
12. [Scale and design for HA](#scale-and-design-for-ha)
13. [Operate CMAN](#operate-cman)
14. [Patch or reboot a CMAN host](#patch-or-reboot-a-cman-host)
15. [Choose classic CMAN, TDM, or tunneling](#choose-classic-cman-tdm-or-tunneling)
16. [Production acceptance checklist](#production-acceptance-checklist)

## Scope and release assumptions

This tutorial uses **classic Oracle Connection Manager** as the baseline and example syntax that is suitable for modern Oracle Net deployments. CMAN features vary across 19c, 21c, 23ai, and 26ai release updates. Confirm new parameters and commands in the reference for the exact installed release.

The examples use documentation-only names and networks:

| Role | Name | Address/port |
| --- | --- | --- |
| Client subnet | application clients | `10.20.10.0/24` |
| CMAN host | `cman01.example.com` | TCP `1522` |
| Database host | `db01.example.com` | TCP `1521` |
| Database service | `sales.example.com` | service name |
| CMAN instance | `CMAN01` | entry in `cman.ora` |

Replace every example value. Do not copy documentation networks or host names into production.

## What CMAN does

Oracle Connection Manager is an Oracle Net proxy placed between clients and database listeners. Common reasons to deploy it are:

- Restrict database access by client source, database destination, and service name.
- Avoid direct client connectivity to database subnets.
- Route Oracle Net traffic through one or more controlled network hops.
- Convert or terminate supported network protocols, including TCP and TCPS designs.
- Reduce server-side connection use through classic shared-server multiplexing or, in TDM, through newer proxy-resident pooling features.
- Add a stable proxy tier in front of RAC or non-RAC databases.

CMAN is not a generic SQL firewall. Its access rules match network and service attributes; they do not inspect SQL statements.

## Architecture

A classic request follows this path:

```text
Oracle client -> CMAN listener -> CMGW gateway -> database listener -> database service
```

The main CMAN components are:

| Component | Responsibility |
| --- | --- |
| CMAN listener | Accept client connections, evaluate access rules, and hand accepted requests to a gateway. |
| `CMADMIN` | Monitor the listener and gateways, maintain gateway state, and answer CMCTL administration requests. |
| `CMGW` | Relay traffic to another CMAN hop or the database server. Multiple gateway processes share the workload. |
| `CMCTL` | Start, stop, reload, inspect, and temporarily alter a CMAN instance. |

In classic CMAN, a gateway is selected according to load. Session multiplexing is available only when the database uses shared server and multiplexing is configured. It is not automatic for dedicated-server connections.

## Plan the deployment

Collect these facts before editing configuration:

```bash
# Run as the intended Oracle software owner.
id
command -v cmctl
cmctl show version
printf '%s\n' "$ORACLE_HOME" "$ORACLE_BASE" "$TNS_ADMIN"
```

Confirm the network path in both directions:

1. Clients must reach the CMAN listening endpoint, for example `cman01.example.com:1522`.
2. The CMAN host must reach each database listener, for example `db01.example.com:1521`.
3. Database hosts do not need direct inbound connections from the client subnet for this route.
4. DNS on the CMAN host must resolve every host name used in `DST` and client connect descriptors consistently.

Choose a nondefault CMAN port when it makes the proxy role clearer. There is no requirement to reuse database listener port `1521`.

## Locate the software and configuration

CMAN is installed as an Oracle Database Client component. Verify that `cmctl` exists in the selected Oracle home. Starting with newer 26ai client release updates, Oracle also distributes a CMAN image installation; older supported deployments commonly use the Oracle Client installer with the CMAN component selected.

CMAN reads `cman.ora`. Oracle searches locations such as:

1. The directory selected by `TNS_ADMIN`.
2. The Oracle base home network administration directory in read-only-home installations.
3. `$ORACLE_HOME/network/admin`.

Use an external `TNS_ADMIN` directory when possible so configuration is not tied to a patchable Oracle home:

```bash
export ORACLE_HOME=/u01/app/oracle/product/23ai/client_1
export TNS_ADMIN=/u01/app/oracle/network/admin
export PATH="$ORACLE_HOME/bin:$PATH"

install -d -m 0750 "$TNS_ADMIN"
```

The directory creation command changes the filesystem; confirm the path and ownership before using it. Keep configuration readable only by the Oracle software owner and approved administrators. Never place wallet passwords or private keys in a world-readable directory.

## Build a minimal secure classic CMAN configuration

Create `$TNS_ADMIN/cman.ora`:

```text
CMAN01 =
  (CONFIGURATION =
    (ADDRESS =
      (PROTOCOL = TCP)
      (HOST = cman01.example.com)
      (PORT = 1522)
    )
    (RULE_LIST =
      (RULE =
        (SRC = 10.20.10.0/24)
        (DST = db01.example.com)
        (SRV = sales.example.com)
        (ACT = accept)
      )
      (RULE =
        (SRC = cman01.example.com)
        (DST = cman01.example.com)
        (SRV = cmon)
        (ACT = accept)
      )
    )
    (PARAMETER_LIST =
      (DIAG_ADR_ENABLED = ON)
      (ADR_BASE = /u01/app/oracle)
      (MIN_GATEWAY_PROCESSES = 2)
      (MAX_GATEWAY_PROCESSES = 8)
      (MAX_CONNECTIONS = 256)
      (INBOUND_CONNECT_TIMEOUT = 60)
      (OUTBOUND_CONNECT_TIMEOUT = 30)
      (CONNECTION_STATISTICS = YES)
    )
  )
```

This configuration deliberately has no wildcard client allow rule. A client must match the allowed source subnet, database destination, and service. The second rule permits the CMAN control service from the CMAN host itself.

Important details:

- CMAN applies `ACT` from the first matching rule. Put specific deny/drop rules before a broader allow.
- `SRC` and `DST` accept host names, IP addresses, or CIDR-style `/nn` subnets. Partial IP wildcards are not supported.
- `SRV` is the service requested by the client. Use service names rather than SIDs for modern CDB/PDB deployments.
- `reject` returns an error; `drop` rejects without returning an error message.
- If rules are missing, connections are rejected. Maintain at least one client rule and one `cmon` rule.
- `MAX_CONNECTIONS` is per gateway process. Treat `MAX_CONNECTIONS * MAX_GATEWAY_PROCESSES` as an upper-bound planning number, not guaranteed application capacity.

## Configure a client route

On the client, define a source-routed alias in `tnsnames.ora`:

```text
SALES_VIA_CMAN =
  (DESCRIPTION =
    (SOURCE_ROUTE = YES)
    (ADDRESS_LIST =
      (ADDRESS =
        (PROTOCOL = TCP)
        (HOST = cman01.example.com)
        (PORT = 1522)
      )
      (ADDRESS =
        (PROTOCOL = TCP)
        (HOST = db01.example.com)
        (PORT = 1521)
      )
    )
    (CONNECT_DATA =
      (SERVICE_NAME = sales.example.com)
    )
  )
```

`SOURCE_ROUTE=YES` tells Oracle Net to traverse the addresses in order: first CMAN, then the database listener. Do not put `SOURCE_ROUTE` at the same address-list level as `FAILOVER` or `LOAD_BALANCE`; source routing is sequential, whereas those options select among peer addresses. Multi-CMAN designs require a correctly nested hop layout or an external load balancer.

Test name resolution and the route without embedding a password in shell history:

```bash
tnsping SALES_VIA_CMAN
sqlplus /nolog
```

Then connect interactively from SQL*Plus:

```text
SQL> connect app_user@SALES_VIA_CMAN
```

After connection, confirm the database service and source information as appropriate for the application:

```sql
SELECT
  sys_context('USERENV', 'SERVICE_NAME') AS service_name,
  sys_context('USERENV', 'HOST')         AS client_host,
  sys_context('USERENV', 'IP_ADDRESS')   AS source_ip
FROM dual;
```

By default, the database may see the CMAN host as the network peer. Newer releases support client IP forwarding with `ENABLE_IP_FORWARDING` on CMAN and `TCP.ALLOWED_PROXIES` on the database server. Enable both sides together and restrict allowed proxies to known CMAN instances.

## Start and verify CMAN

Use the Oracle software owner that should control CMAN through LOSA:

```bash
cmctl startup -c CMAN01
cmctl show status -c CMAN01
cmctl show services -c CMAN01
cmctl show gateways -c CMAN01
cmctl show rules -c CMAN01
cmctl show parameters -c CMAN01
```

The startup should create the listener, `CMADMIN`, and the configured minimum number of gateway processes. Verify all of the following:

- The instance name and configuration-file path are the expected values.
- The listener reports the planned host and port.
- The `cmgw` and `cmon` services are ready.
- The effective rules match the intended order.
- The gateway count is within the configured minimum and maximum.

Oracle recommends Local Operating System Authentication. In 26ai, password access to CMAN administration is desupported. Run CMCTL locally as the user that started CMAN and avoid designing new automation around CMAN passwords.

## Add database service registration

Static source routing can connect to an explicit database listener address. Dynamic service registration with CMAN is useful when CMAN should know registered services and handlers, especially in more advanced routing and HA designs.

On the database host, define the CMAN listener address in `tnsnames.ora`:

```text
CMAN_LISTENER =
  (ADDRESS =
    (PROTOCOL = TCP)
    (HOST = cman01.example.com)
    (PORT = 1522)
  )
```

On CMAN, allow registration only from database nodes by adding these entries to `PARAMETER_LIST`:

```text
(VALID_NODE_CHECKING_REGISTRATION = ON)
(REGISTRATION_INVITED_NODES = db01.example.com,10.20.20.11)
```

Inspect the database listener settings before changing them:

```sql
SHOW PARAMETER local_listener
SHOW PARAMETER remote_listener
SHOW PARAMETER listener_networks
```

For a simple non-RAC database with no existing remote listener, configure registration and force an immediate registration:

```sql
ALTER SYSTEM SET REMOTE_LISTENER = 'CMAN_LISTENER' SCOPE = BOTH;
ALTER SYSTEM REGISTER;
```

Then verify on CMAN:

```bash
cmctl show services -c CMAN01
cmctl show stats -reg -c CMAN01
```

Do not overwrite a RAC SCAN value or an existing registration topology. For RAC or multiple listener networks, design `REMOTE_LISTENER` or `LISTENER_NETWORKS` with the cluster owner and test registration, FAN/ONS behavior, and client failover end to end.

## Understand rules

Use rules as an ordered allowlist. A practical pattern is:

```text
(RULE_LIST =
  # Block one untrusted client first.
  (RULE=(SRC=10.20.10.99)(DST=db01.example.com)(SRV=*)(ACT=drop))

  # Permit the application subnet to one service.
  (RULE=(SRC=10.20.10.0/24)(DST=db01.example.com)
        (SRV=sales.example.com)(ACT=accept)
        (ACTION_LIST=(MIT=1800)(MCT=28800)(CONN_STATS=yes)))

  # Permit local administration.
  (RULE=(SRC=cman01.example.com)(DST=cman01.example.com)
        (SRV=cmon)(ACT=accept))
)
```

Rule-level `ACTION_LIST` settings override global settings for the matching connection:

| Setting | Meaning |
| --- | --- |
| `AUT` | Require or disable Oracle security authentication filtering for the rule. |
| `CONN_STATS` | Collect input/output statistics. |
| `MCT` | Maximum connection/session time. |
| `MIT` | Maximum idle time. |
| `MOCT` | Maximum outbound connect time. |

Before deploying an idle timeout, verify that the application pool validates connections and can recover from server-side disconnects. A syntactically valid timeout can still cause an outage for long-idle pools.

After changing `cman.ora`:

```bash
cmctl reload -c CMAN01
cmctl show rules -c CMAN01
cmctl show parameters -c CMAN01
```

`SET` commands alter runtime values only until shutdown. Make persistent changes in `cman.ora`.

## Enable TLS deliberately

A production TCPS design requires decisions about each leg:

```text
client -- TCP or TCPS --> CMAN -- TCP or TCPS --> database listener
```

Do not assume that enabling TCPS on the client-facing endpoint automatically secures the CMAN-to-database leg. Configure and test both legs. The design normally includes:

- A CMAN `TCPS` listening address.
- Oracle wallets at the required endpoints.
- Trust anchors and certificates with correct DNS identities.
- `TLS_VERSION`, `TLS_CIPHER_SUITES`, and client-authentication policy appropriate to the installed release.
- A `RULE_GROUP`/`DN_LIST` design if filtering TLS clients by certificate distinguished name.
- Database-side `sqlnet.ora` and listener configuration for the outbound TCPS leg.

Keep a separate TCP endpoint temporarily during staged migration only if the security policy allows it, and remove it after validation. Do not publish copy-paste cipher lists without checking the exact release and corporate cryptographic policy.

## Scale and design for HA

Gateway capacity is governed primarily by:

```text
approximate upper-bound slots = MAX_CONNECTIONS * active gateway processes
```

Monitor actual gateways, connections, refusal counts, CPU, memory, and network throughput. Increase limits gradually; raising slots cannot compensate for OS file-descriptor, memory, database process/session, or listener-handler limits.

A single CMAN host is a single point of failure. For HA:

1. Deploy at least two independent CMAN instances on separate failure domains.
2. Keep their rule and TLS policy consistent, but use unique instance names and diagnostics.
3. Use a supported client connect descriptor with peer CMAN endpoints or a load balancer configured for Oracle Net pass-through.
4. Test CMAN process failure, host failure, database service relocation, and rolling maintenance.
5. Drain one instance at a time.

`SOURCE_ROUTE` and peer failover/load balancing must be placed at different descriptor levels. Validate the final descriptor with the exact client release rather than improvising a production HA descriptor.

## Operate CMAN

Use the following daily inspection sequence:

```bash
cmctl show status -c CMAN01
cmctl show services -c CMAN01
cmctl show gateways -c CMAN01
cmctl show connections count -c CMAN01
cmctl show rules -c CMAN01
cmctl show parameters -c CMAN01
```

For connection details:

```bash
cmctl show connections detail -c CMAN01
cmctl show established connections detail for sales.example.com -c CMAN01
cmctl show idle connections count gt 30:00 -c CMAN01
```

Use connection statistics deliberately; they add observability but also overhead. Compare CMAN refusal/current counts with database listener services and database resource limits.

## Patch or reboot a CMAN host

Use one CMAN at a time in an HA deployment.

### Before maintenance

```bash
date
cmctl show version -c CMAN01
cmctl show status -c CMAN01
cmctl show services -c CMAN01
cmctl show gateways -c CMAN01
cmctl show connections count -c CMAN01
cmctl show parameters -c CMAN01
cmctl show rules -c CMAN01
```

Also:

1. Record `ORACLE_HOME`, `ORACLE_BASE`, `TNS_ADMIN`, and CMAN owner.
2. Back up `cman.ora`, `tnsnames.ora`, `sqlnet.ora`, wallets, and service-manager definitions using the organization's secure backup process.
3. Confirm that another CMAN endpoint is healthy and clients can use it.
4. Remove or disable the target endpoint from client selection or the load balancer.
5. Watch active connections fall.

### Drain and stop

From interactive CMCTL:

```text
CMCTL> ADMINISTER CMAN01
CMCTL> SHOW CONNECTIONS COUNT
CMCTL> SHUTDOWN NORMAL TIMEOUT 600 NOTIFY
```

`NORMAL` rejects new connections and waits for existing connections to close. `TIMEOUT` sets the wait in seconds. `NOTIFY` requests client notification on supported versions/clients. Confirm that the instance and its processes are stopped before patching.

Use `SHUTDOWN ABORT` only with explicit approval to terminate all active connections immediately.

### Patch or reboot

Follow the patch README and the Oracle Client/CMAN release documentation for the selected Oracle home. Preserve an external `TNS_ADMIN` and wallets. Do not copy binaries between Oracle homes.

### Start and validate

```bash
cmctl startup -c CMAN01
cmctl show version -c CMAN01
cmctl show status -c CMAN01
cmctl show services -c CMAN01
cmctl show gateways -c CMAN01
cmctl show rules -c CMAN01
cmctl show parameters -c CMAN01
```

Run one controlled connection through CMAN and verify the expected database service. Re-enable the endpoint, monitor refusals and errors, and only then continue to the next CMAN instance.

Newer releases provide `STARTUP -MIGRATE` for same-host migration between Oracle homes. Treat it as an advanced rolling-upgrade mechanism: validate exact release prerequisites and rehearse it outside production before relying on it.

## Choose classic CMAN, TDM, or tunneling

| Mode | Choose it for | Important prerequisites |
| --- | --- | --- |
| Classic CMAN | Proxy routing, access control, network isolation, protocol path, shared-server multiplexing. | `cman.ora`, client routing or service registration, restrictive rules. |
| CMAN TDM | Proxy-resident pooling, improved connection management, and supported application-continuity/HA scenarios. | Compatible client and database releases; `TDM=YES`; threading/pool design; proxy user and wallet; `CONNECT THROUGH`; DRCP-aware pooled clients where pooled mode is used. |
| Tunneling | Reverse connections across a network where the database side initiates/maintains tunnels to a server CMAN. | Paired server/client CMAN configuration, tunnel identifiers, tunnel-specific rules, capacity and security design. |

Do not enable TDM by adding only `TDM=YES` to a classic configuration. TDM changes authentication, pooling, driver compatibility, operational behavior, and failure handling. Build it as a separate design.

## Production acceptance checklist

- [ ] Exact CMAN, client, and database releases are documented and supported together.
- [ ] CMAN software owner, Oracle home, `TNS_ADMIN`, instance name, and startup mechanism are documented.
- [ ] Client-to-CMAN and CMAN-to-database firewall paths are restricted and tested.
- [ ] DNS forward/reverse behavior is stable for every name used in rules.
- [ ] Rules allow only approved sources, destinations, and services.
- [ ] The `cmon` rule and LOSA administration are restricted to approved administrators.
- [ ] TCP/TCPS policy is explicit for both legs.
- [ ] Dynamic registration accepts only approved database nodes.
- [ ] RAC `REMOTE_LISTENER`/SCAN settings were not overwritten.
- [ ] Capacity limits were load-tested against OS and database limits.
- [ ] At least two CMAN instances exist when HA is required.
- [ ] Drain, stop, start, rollback, and emergency-abort procedures were rehearsed.
- [ ] ADR/log rotation, monitoring, and alerting are configured.
- [ ] A controlled application connection test is part of every change.

## Official references

- [Oracle AI Database 26ai: Configuring and Administering Oracle Connection Manager](https://docs.oracle.com/en/database/oracle/oracle-database/26/netag/configuring-and-administering-oracle-connection-manager.html)
- [Oracle AI Database 26ai: Oracle Connection Manager parameters](https://docs.oracle.com/en/database/oracle/oracle-database/26/netrf/oracle-connection-manager-parameters.html)
- [Oracle AI Database 26ai: CMCTL reference](https://docs.oracle.com/en/database/oracle/oracle-database/26/netrf/oracle-connection-manager-control-utility.html)
- [Oracle AI Database 26ai: CMAN architecture](https://docs.oracle.com/en/database/oracle/oracle-database/26/netag/understanding-oracle-connection-manager-architecture.html)
- [Oracle AI Database 26ai: Configuring database service registration with CMAN](https://docs.oracle.com/en/database/oracle/oracle-database/26/netag/configuring-oracle-database-server-oracle-connection-manager.html)
- [Oracle AI Database 26ai: Address-list and SOURCE_ROUTE parameters](https://docs.oracle.com/en/database/oracle/oracle-database/26/netag/address-list-parameters.html)
- [Oracle Database 19c: Configuring Oracle Connection Manager](https://docs.oracle.com/en/database/oracle/oracle-database/19/netag/configuring-oracle-connection-manager.html)
- [Oracle LiveLabs tutorial: Install and Configure Oracle Connection Manager](https://docs.oracle.com/en/learn/oracle-cman/index.html)

