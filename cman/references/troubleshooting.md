# Oracle Connection Manager Troubleshooting and Runbook

## Contents

1. [Triage order](#triage-order)
2. [Capture the current state](#capture-the-current-state)
3. [Locate the failing hop](#locate-the-failing-hop)
4. [CMAN does not start](#cman-does-not-start)
5. [Client reaches CMAN but cannot connect](#client-reaches-cman-but-cannot-connect)
6. [Rules reject an expected client](#rules-reject-an-expected-client)
7. [Database service does not register](#database-service-does-not-register)
8. [Intermittent refusals or capacity errors](#intermittent-refusals-or-capacity-errors)
9. [Idle connections disappear](#idle-connections-disappear)
10. [Find CMAN diagnostics in ADR](#find-cman-diagnostics-in-adr)
11. [Trace safely](#trace-safely)
12. [Common Oracle Net errors](#common-oracle-net-errors)
13. [Recovery and rollback](#recovery-and-rollback)

## Triage order

Diagnose the path one hop at a time:

```text
1. client naming
2. client -> CMAN TCP/TCPS endpoint
3. CMAN listener/rules/gateway
4. CMAN -> database listener
5. database service/handler/resources
6. application authentication and session behavior
```

Do not begin with a CMAN restart. A restart erases useful transient state and disconnects sessions without proving the cause.

## Capture the current state

Run locally as the CMAN owner:

```bash
date -Is
id
printf 'ORACLE_HOME=%s\nORACLE_BASE=%s\nTNS_ADMIN=%s\n' \
  "$ORACLE_HOME" "$ORACLE_BASE" "$TNS_ADMIN"

cmctl show version -c CMAN01
cmctl show status -c CMAN01
cmctl show services -c CMAN01
cmctl show gateways -c CMAN01
cmctl show connections count -c CMAN01
cmctl show parameters -c CMAN01
cmctl show rules -c CMAN01
```

When the issue is connection-specific:

```bash
cmctl show connections detail -c CMAN01
cmctl show established connections detail for sales.example.com -c CMAN01
```

Capture application timestamp, client address, connect alias, requested service, CMAN instance, destination listener, and full Oracle error stack. Correlate all logs in one timezone.

## Locate the failing hop

### On the client

```bash
getent hosts cman01.example.com
nc -vz cman01.example.com 1522
tnsping SALES_VIA_CMAN
```

`tnsping` validates naming and listener reachability; it does not authenticate to the database or prove that the database service is usable.

Inspect the resolved descriptor and confirm:

- `SOURCE_ROUTE=YES` is present for a classic multi-hop descriptor.
- CMAN is the first address.
- The database listener is the next address.
- `SERVICE_NAME` exactly matches the intended service.
- The client is reading the expected `tnsnames.ora`/`sqlnet.ora` through its actual `TNS_ADMIN`.

### On the CMAN host

```bash
getent hosts db01.example.com
nc -vz db01.example.com 1521
tnsping SALES_DIRECT
```

Use a known direct database alias on the CMAN host for `tnsping`. If policy allows, make a controlled database connection from the CMAN host. Do not put passwords on the command line.

### On the database host

```bash
lsnrctl status
lsnrctl services
```

Check that the requested service has a ready handler and that database resource limits are not exhausted.

## CMAN does not start

### Checks

```bash
cmctl startup -c CMAN01
cmctl show status -c CMAN01
ss -ltnp | grep ':1522'
ps -ef | grep -E '[c]man|[c]mgw|[c]madmin'
```

Common causes:

| Symptom | Check | Corrective direction |
| --- | --- | --- |
| Instance not found | CMAN entry name and active `TNS_ADMIN` | Use the exact entry from the correct `cman.ora`. |
| Address already in use | `ss -ltnp`, another CMAN/listener | Stop the conflicting owner or select the planned endpoint; never kill an unknown listener blindly. |
| Configuration parse error | Recent edit, parentheses, spelling, file encoding | Restore last known good file, then reapply one small change. |
| Permission error | Oracle owner access to config, ADR base, wallet | Correct ownership/permissions through the approved OS process. |
| Wrong Oracle home | `command -v cmctl`, `cmctl show version` | Select the CMAN-enabled Oracle home. |
| CMCTL authorization failure | OS user and LOSA | Run locally as the user that started CMAN. |

Inspect the ADR alert log before changing the configuration.

## Client reaches CMAN but cannot connect

If TCP to CMAN succeeds but the database connection fails:

1. Run `cmctl show status`, `show services`, and `show gateways`.
2. Run `show rules` and compare the actual source, destination, and service.
3. Test CMAN-to-database DNS and TCP reachability.
4. Check database listener services and handler status.
5. Compare the requested service with the database service exactly, including domain suffix.
6. Check CMAN and database ADR logs at the same timestamp.

CMAN can accept a TCP connection and still reject the Oracle Net request because of a rule, unavailable gateway, outbound timeout, unknown service, or database listener failure.

## Rules reject an expected client

CMAN uses the first matching rule. Review the effective order:

```bash
cmctl show rules -c CMAN01
```

Check these frequent mismatches:

- A broad reject/drop appears before the intended allow.
- `SRC` contains the load balancer or NAT address rather than the original client.
- `DST` uses a host name but CMAN resolves a different address/name than expected.
- `SRV` omits or adds a database domain.
- A partial IP wildcard was used; CMAN supports exact addresses, a complete `*`, or `/nn` subnet notation.
- The `cmon` administration rule is missing or does not match the CMAN host.
- The file was edited but not reloaded, or a different `TNS_ADMIN` is active.

After an approved edit:

```bash
cmctl reload -c CMAN01
cmctl show rules -c CMAN01
```

Test from one known client before broadening any rule. Do not fix a rule mismatch by adding a wildcard allow at the top.

## Database service does not register

On CMAN:

```bash
cmctl show services -c CMAN01
cmctl show stats -reg -c CMAN01
cmctl show parameters -c CMAN01
```

On the database:

```sql
SHOW PARAMETER service_names
SHOW PARAMETER local_listener
SHOW PARAMETER remote_listener
SHOW PARAMETER listener_networks

ALTER SYSTEM REGISTER;
```

Validate:

- The database host resolves the CMAN listener alias.
- `REMOTE_LISTENER` or `LISTENER_NETWORKS` includes the intended CMAN endpoint.
- CMAN can receive registration from the database node.
- `VALID_NODE_CHECKING_REGISTRATION` and `REGISTRATION_INVITED_NODES` permit the actual database source address.
- Network ACLs allow database-to-CMAN registration traffic.
- RAC SCAN registration remains intact.

Do not repeatedly issue `ALTER SYSTEM REGISTER` without reading the registration statistics and logs; repeated attempts do not correct an ACL, naming, or listener-address error.

## Intermittent refusals or capacity errors

Collect:

```bash
cmctl show services -c CMAN01
cmctl show gateways -c CMAN01
cmctl show connections count -c CMAN01
cmctl show connections detail -c CMAN01
```

Check:

- `current`, `established`, and `refused` gateway counts.
- Active gateway count versus `MIN_GATEWAY_PROCESSES` and `MAX_GATEWAY_PROCESSES`.
- `MAX_CONNECTIONS` per gateway and total observed connection demand.
- CMAN host CPU, memory, file-descriptor limits, socket states, and packet loss.
- `INBOUND_CONNECT_TIMEOUT` and `OUTBOUND_CONNECT_TIMEOUT`.
- IP/service connection-rate limits on releases that support them.
- Database listener handler state and database `PROCESSES`/`SESSIONS` headroom.
- Application connection storms and pool configuration.

Do not raise CMAN limits in isolation. A higher CMAN ceiling may only move the failure to the OS or database.

## Idle connections disappear

Inspect both global parameters and matching rule overrides:

```bash
cmctl show parameters -c CMAN01
cmctl show rules -c CMAN01
cmctl show idle connections detail -c CMAN01
```

Potential timeout sources include:

- CMAN `IDLE_TIMEOUT`.
- Rule-level `MIT`.
- CMAN `SESSION_TIMEOUT` or rule-level `MCT`.
- Load balancer or firewall idle timeout.
- Database profile `IDLE_TIME`.
- Application pool lifetime/idle settings.
- TCP keepalive or Oracle Net expire settings.

Identify the shortest active timeout. Coordinate application pool validation before extending or disabling a timeout.

## Find CMAN diagnostics in ADR

ADR is enabled by default in modern releases when `DIAG_ADR_ENABLED=ON`.

```bash
adrci
```

```text
ADRCI> show homes
ADRCI> set home diag/netcman/<host>/<cman_instance>
ADRCI> show alert -tail 100 -term
ADRCI> show alert -p "originating_timestamp >= systimestamp-1/24"
ADRCI> show incident
ADRCI> show problem
```

The effective ADR base and home are shown by CMCTL status/parameters and ADRCI. Do not guess the host or instance component when multiple CMAN homes exist.

When ADR is disabled explicitly, inspect the non-ADR directories shown by:

```bash
cmctl show parameters -c CMAN01
```

## Trace safely

Start with normal logs. If trace is necessary:

```bash
cmctl set trace_level user -c CMAN01
```

Reproduce one controlled failure, capture timestamps and connection identity, then disable trace:

```bash
cmctl set trace_level off -c CMAN01
```

Use `ADMIN` or `SUPPORT` tracing only when a lower level is insufficient or Oracle Support requests it. Monitor disk space throughout. Remember that `SET` is runtime-only.

## Common Oracle Net errors

| Error/symptom | Likely layer | First checks |
| --- | --- | --- |
| `ORA-12154` | Client naming | Active `TNS_ADMIN`, alias spelling, descriptor parentheses, naming methods. |
| `ORA-12541` / no listener | TCP endpoint/listener | DNS, port, firewall, CMAN status, destination listener. Determine which address in the route failed. |
| `ORA-12514` / service unknown | Service registration/connect data | Requested `SERVICE_NAME`, CMAN/database `SHOW SERVICES`, registration ACL and logs. |
| `ORA-12516` or `ORA-12520` | Database handler/resources | Database listener handlers, shared/dedicated server mode, `PROCESSES`/`SESSIONS`. |
| CMAN connection rejected | Rule or limit | `SHOW RULES`, actual source/destination/service, first match, rate/capacity limits. |
| CMAN connection dropped with little client detail | `ACT=drop` or network device | Effective rules, CMAN log, firewall/LB logs. |
| Works direct, fails through CMAN | CMAN hop | Source route, rules, CMAN-to-database reachability, outbound timeout. |
| Works through one CMAN only | Configuration drift/HA | Compare effective rules, parameters, wallets, DNS, and outbound reachability. |

An Oracle error shown to the client may summarize only the top of a deeper network error stack. Use the CMAN and database listener logs to identify the failing layer.

## Recovery and rollback

For a failed configuration change:

1. Preserve the failing file and timestamps for analysis.
2. Restore the last known good `cman.ora` and any related network files.
3. Reload if the changed settings are reloadable; otherwise drain and restart using the approved procedure.
4. Verify `SHOW RULES`, `SHOW PARAMETERS`, `SHOW SERVICES`, and gateways.
5. Run one controlled client connection through CMAN.
6. Confirm application recovery before closing the incident.

For a failed patch or new Oracle home:

1. Keep the endpoint out of traffic.
2. Stop the failed CMAN home normally if possible.
3. Re-select the previous Oracle home without copying binaries.
4. Point it to the preserved external `TNS_ADMIN` and wallets.
5. Start, verify, test, and only then return the endpoint to service.

## Official references

- [CMCTL command reference](https://docs.oracle.com/en/database/oracle/oracle-database/26/netrf/oracle-connection-manager-control-utility.html)
- [Oracle Connection Manager parameters](https://docs.oracle.com/en/database/oracle/oracle-database/26/netrf/oracle-connection-manager-parameters.html)
- [Oracle Net logging and diagnostics](https://docs.oracle.com/en/database/oracle/oracle-database/26/netag/logging-error-information-oracle-net-services.html)
- [Automatic Diagnostic Repository](https://docs.oracle.com/en/database/oracle/oracle-database/26/netag/understanding-automatic-diagnostic-repository.html)
- [Database service registration with CMAN](https://docs.oracle.com/en/database/oracle/oracle-database/26/netag/configuring-oracle-database-server-oracle-connection-manager.html)
