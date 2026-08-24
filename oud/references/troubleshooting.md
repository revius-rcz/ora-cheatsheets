# Oracle Unified Directory Troubleshooting and Runbook

## Contents

1. [Triage order](#triage-order)
2. [Capture evidence first](#capture-evidence-first)
3. [Classify the failing layer](#classify-the-failing-layer)
4. [OUD is down or unreachable](#oud-is-down-or-unreachable)
5. [OUD does not start](#oud-does-not-start)
6. [LDAP bind or authorization fails](#ldap-bind-or-authorization-fails)
7. [LDAPS or certificate validation fails](#ldaps-or-certificate-validation-fails)
8. [Search is slow or times out](#search-is-slow-or-times-out)
9. [Disk, heap, or work queue is under pressure](#disk-heap-or-work-queue-is-under-pressure)
10. [Replication is late or disconnected](#replication-is-late-or-disconnected)
11. [Replication reports Bad Data Set](#replication-reports-bad-data-set)
12. [TNS alias returns ORA-12154](#tns-alias-returns-ora-12154)
13. [Alias resolves but database connection fails](#alias-resolves-but-database-connection-fails)
14. [Alias change is missing on another node](#alias-change-is-missing-on-another-node)
15. [TNS alias add or modify fails](#tns-alias-add-or-modify-fails)
16. [OUDSM is unavailable](#oudsm-is-unavailable)
17. [Patch fails](#patch-fails)
18. [Upgrade fails](#upgrade-fails)
19. [Recovery and escalation package](#recovery-and-escalation-package)

## Triage order

Use this order and stop at the first failing layer:

```text
1. host, process, filesystem, time, and sockets
2. OUD local status and errors log
3. TCP to LDAP/LDAPS/administration endpoint
4. TLS handshake and certificate validation
5. Root DSE
6. bind and ACI
7. suffix or Oracle Context search
8. replication and back-end state
9. Oracle client naming configuration
10. database listener, service, and authentication
```

Do not begin with restart, replication initialization, index rebuild, or restore. Each can hide evidence or increase impact.

## Capture evidence first

Run locally as the approved OUD owner:

```bash
date -Is
id
uname -a
java -version
printf 'ORACLE_HOME=%s\nOUD_INSTANCE=%s\nDOMAIN_HOME=%s\n' \
  "$ORACLE_HOME" "$OUD_INSTANCE" "$DOMAIN_HOME"

"$OUD_BIN/status"
ps -ef | grep -E '[o]rg\.opends|[o]racle\.directory|[j]ava'
ss -ltnp
df -h "$OUD_INSTANCE"
df -i "$OUD_INSTANCE"
ulimit -a
ls -ltr "$OUD_INSTANCE/OUD/logs" | tail -50
tail -200 "$OUD_INSTANCE/OUD/logs/errors" 2>/dev/null
```

Capture replication with the environment's approved authentication command:

```bash
"$OUD_BIN/dsreplication" status \
  --hostname oud01.example.com --port 4444 \
  --adminUID admin \
  --adminPasswordFile /secure/oud/oud-admin.pwd \
  --trustStorePath /secure/oud/client-truststore \
  --no-prompt
```

For an application incident, record:

- first and last failure time with timezone;
- exact client address and endpoint/VIP;
- protocol and port;
- bind DN or application identity, redacted as required;
- base DN, scope, filter, requested attributes, and operation type;
- complete LDAP/Oracle error and result code;
- whether all nodes, one node, one alias, or one client is affected;
- recent changes to aliases, ACIs, schema, certificates, DNS, load balancers, JDK, patches, indexes, or replication.

## Classify the failing layer

| Observation | Likely layer |
| --- | --- |
| TCP connection refused | Process down, wrong port, bind/listener failure, firewall rejection. |
| TCP timeout | Network/firewall/routing, overloaded host, or silent drop. |
| TLS handshake fails | Certificate chain/name/expiry, protocol/cipher, trust store, wrong service on port. |
| Root DSE fails after TLS works | LDAP handler, bind requirement, server overload, or protocol mismatch. |
| Root DSE succeeds but suffix search fails | Bind, ACI, base DN, proxy route, back end, or filter. |
| One replica differs | Replication delay/disconnection, wrong endpoint, or change made in another environment/context. |
| LDAP entry is correct but `tnsping` fails | Oracle client `ldap.ora`, `sqlnet.ora`, TLS/wallet, naming order, or client library. |
| `tnsping` succeeds but SQL connect fails | Database listener/service/network/authentication; usually not OUD. |
| OUDSM fails but LDAP works | WebLogic/OUDSM/domain problem, not OUD data service. |

## OUD is down or unreachable

### Checks

```bash
"$OUD_BIN/status"
ps -ef | grep -E '[o]rg\.opends|[j]ava'
ss -ltnp
nc -vz oud01.example.com 1636
getent hosts oud01.example.com
```

From a remote client, test the exact VIP/node and port. From the server, test both loopback/local name and advertised name.

| Symptom | Check | Corrective direction |
| --- | --- | --- |
| Process absent | Service manager, errors log, last stop reason, OS reboot | Start only after verifying it is expected to be down and dependencies are ready. |
| Process present, port absent | Connection-handler config, bind address, startup errors | Correct the handler or address conflict through `dsconfig`/approved offline procedure. |
| Node works, VIP fails | Load balancer pool/health check, firewall, DNS | Fix the service endpoint; do not restart healthy OUD blindly. |
| One interface works | Listener address and routing | Align handler listen address with the documented design. |
| Port owned by another process | `ss -ltnp`, process owner | Do not kill an unknown process; resolve the conflict with its owner. |

If the server was intentionally drained, confirm change records before re-adding it to service.

## OUD does not start

Run a normal start and read the first error, not just the final exit code:

```bash
"$OUD_BIN/start-ds"
tail -200 "$OUD_INSTANCE/OUD/logs/errors" 2>/dev/null
```

For a controlled foreground diagnosis:

```bash
"$OUD_BIN/start-ds" -N
```

Common causes:

| Cause | Evidence | Safe response |
| --- | --- | --- |
| Wrong OS owner or permissions | Permission-denied errors under instance/log/db/config | Restore approved ownership/mode; do not recursively chmod the entire Oracle tree. |
| Port already used | Bind exception and `ss -ltnp` | Identify the owner and planned port; do not kill blindly. |
| Full filesystem/inodes | `df -h`, `df -i`, log/database write errors | Stop growth safely, extend space, preserve evidence, then start. |
| Invalid configuration | Error references `config.ldif` or recent `dsconfig`/manual edit | Restore the exact last-known-good configuration or reverse the approved change. |
| JDK incompatible/moved | Java-version or path errors; changed `java.properties` | Restore the certified JDK/path; do not select an arbitrary system Java. |
| Patch incomplete/conflicted | OPatch inventory and startup class/file errors | Keep service drained; follow patch rollback/support procedure. |
| Database/back-end corruption | Back-end recovery or JE errors | Preserve files and logs; use Oracle-supported recovery. Do not delete lock/database files. |

Do not edit `config/config.ldif` while OUD is running. Offline edits should be a last-resort documented repair with a recoverable copy.

## LDAP bind or authorization fails

Test the layers separately:

1. Root DSE without application data.
2. Bind with the intended identity.
3. Base-object search for the target entry.
4. Intended operation: search, add, modify, or delete.

```bash
"$OUD_BIN/ldapsearch" \
  -h oud01.example.com -p 1636 --useSSL \
  --trustStorePath /secure/oud/client-truststore \
  -D "cn=OracleNetAdmins,dc=example,dc=com" \
  -j /secure/oud/net-admin.pwd \
  -b "cn=OracleContext,dc=example,dc=com" -s base \
  "(objectClass=*)" dn objectClass
```

Common LDAP result codes:

| Code | Meaning | Check |
| --- | --- | --- |
| `0` | Success | Confirm returned entries/attributes, not just result code. |
| `32` | No such object | Base DN/DN, escaping, environment, replication, parent existence. |
| `49` | Invalid credentials | Correct DN, password source, account state; do not log the password. |
| `50` | Insufficient access | Effective ACI, target attributes/operation, proxy authorization. |
| `53` | Unwilling to perform | Server policy, writability mode, control/operation restrictions. |
| `65` | Object class violation | Required object class/attribute, schema, attribute syntax. |
| `68` | Entry already exists | DN/case collision or previous successful attempt. |
| `80` | Other/server error | Errors log and full diagnostic message. |

Check access logs at the same timestamp for bind DN, operation, target DN, result, and diagnostic message. Do not broaden an ACI to `all` as a diagnostic shortcut in production. Compare against a known-good delegated account and the intended subtree.

## LDAPS or certificate validation fails

Inspect the server presentation without changing trust:

```bash
openssl s_client \
  -connect oud01.example.com:1636 \
  -servername oud01.example.com \
  -showcerts </dev/null
```

Check:

- certificate validity dates;
- subject alternative name contains the host or VIP clients use;
- complete chain is presented;
- issuer exists in the client's actual trust store;
- client and server share a supported TLS version/cipher;
- LDAPS is really listening on the chosen port;
- system time is correct;
- load balancer certificate and backend certificate are not being confused.

`PKIX path building failed` usually means the client cannot build a chain to a trusted CA. Import the approved CA/intermediate chain into the correct trust store and retest. Do not permanently replace validation with `--trustAll`.

If diagnosis temporarily uses `--trustAll`, first verify the fingerprint through an independent trusted channel, limit the test, and remove the bypass immediately.

## Search is slow or times out

Capture the exact base DN, scope, filter, attributes, result code, entries scanned/returned, and processing time from the access log.

Check in this order:

1. One node or all nodes?
2. One filter/base DN or every search?
3. Network/TLS time versus server processing time?
4. Unindexed-search warnings or high candidate counts?
5. JVM heap/GC, work queue, CPU, disk I/O, and open files?
6. Back-end/cache state and entry count?
7. Replication backlog or storage contention?
8. Recent schema/index/data growth or configuration change?

Read monitor branches:

```bash
# Monitor root
"$OUD_BIN/ldapsearch" \
  -h oud01.example.com -p 4444 --useSSL \
  --trustStorePath /secure/oud/client-truststore \
  -D "cn=Directory Manager" -j /secure/oud/oud-bind.pwd \
  -s base -b "cn=monitor" "(objectClass=*)"

# Discover available branches
"$OUD_BIN/ldapsearch" \
  -h oud01.example.com -p 4444 --useSSL \
  --trustStorePath /secure/oud/client-truststore \
  -D "cn=Directory Manager" -j /secure/oud/oud-bind.pwd \
  -s sub -b "cn=monitor" "(objectClass=*)" "1.1"
```

Do not add an index based on one slow request. Confirm query frequency, selectivity, write/storage cost, existing indexes, and whether rebuild is required. `rebuild-index` is an offline/high-impact operation in common procedures; plan it explicitly.

## Disk, heap, or work queue is under pressure

### Disk/inodes

```bash
df -h "$OUD_INSTANCE"
df -i "$OUD_INSTANCE"
du -x -h --max-depth=2 "$OUD_INSTANCE/OUD" 2>/dev/null | sort -h | tail -30
```

Identify growth by category: access/debug/audit logs, backups, exports, database, replication data, core/heap dumps, or patch staging. Use configured rotation/retention and approved backup relocation. Never delete back-end, changelog, or unknown lock files.

### Heap/GC

Read `cn=JVM Memory Usage,cn=monitor`, GC logs if configured, and OS resident memory. Separate:

- healthy heap cycling from a leak;
- Java heap exhaustion from OS memory pressure;
- long GC pauses from slow storage or an overloaded work queue.

Do not increase heap beyond physical/container limits. Review `java.properties`, certified JVM options, cache sizing, and workload together; run `dsjavaproperties` when the documented procedure requires it after changing Java settings.

### Work queue/connections

Read `cn=Work Queue,cn=monitor`, connection-handler statistics, and `cn=Client Connections,cn=monitor`. Look for connection leaks, sudden bind/search spikes, stuck operations, pool misconfiguration, or external health checks that create excessive load.

## Replication is late or disconnected

```bash
"$OUD_BIN/dsreplication" status \
  --hostname oud01.example.com --port 4444 \
  --adminUID admin \
  --adminPasswordFile /secure/oud/oud-admin.pwd \
  --trustStorePath /secure/oud/client-truststore \
  --dataToDisplay compat-view \
  --no-prompt
```

Check:

- expected topology members and replication-server ports;
- DNS and TCP reachability between peers;
- TLS/trust and certificate expiry for secure replication;
- clock synchronization;
- disk, I/O, heap, CPU, and error logs on the lagging node;
- whether missing changes and their age are growing or recovering;
- whether the node was offline longer than the replication purge delay;
- recent restore, import, initialization, topology, or group-ID changes.

`Late` with a shrinking backlog can be transient recovery. Persistent growth, `Not Connected`, `Down`, or `Unknown` requires action. Keep a suspect replica out of client write service until its state is understood.

Do not initialize as the first repair. An initialization replaces destination data for the selected base DN and can destroy the only good copy if source/destination are reversed.

## Replication reports Bad Data Set

Treat `Bad Data Set` as a high-severity data-consistency problem.

1. Stop routing client writes to the affected replica if the design permits.
2. Capture full topology status and errors from all involved nodes.
3. Identify the last restore/import/initialize/upgrade/topology change.
4. Compare base DN, server IDs, generation/data-set identifiers, entry counts, and authoritative business state.
5. Choose the authoritative source with the data owner and recovery lead.
6. Back up the affected and source states before repair.
7. Use the exact Oracle procedure for initialization or recovery.

Never guess source and destination. Never initialize two nodes in opposite directions.

## TNS alias returns ORA-12154

ORA-12154 means the connect identifier was not resolved. Diagnose the naming layer before the database listener.

### 1. Confirm the Oracle client files

```bash
printf 'ORACLE_HOME=%s\nTNS_ADMIN=%s\nLDAP_ADMIN=%s\n' \
  "$ORACLE_HOME" "$TNS_ADMIN" "$LDAP_ADMIN"
sed -n '1,160p' "$TNS_ADMIN/sqlnet.ora"
sed -n '1,160p' "$TNS_ADMIN/ldap.ora"
```

Check:

- `NAMES.DIRECTORY_PATH` contains `LDAP` in the intended order;
- `DIRECTORY_SERVERS` has the correct host and LDAP/LDAPS ports;
- `DEFAULT_ADMIN_CONTEXT` is the parent containing `cn=OracleContext`;
- `DIRECTORY_SERVER_TYPE` matches the provisioned Oracle directory naming configuration;
- the application uses the same Oracle home and environment as the interactive shell;
- no local `tnsnames.ora` entry or different Oracle home masks the result.

### 2. Check DNS, TCP, and TLS

```bash
getent hosts oud-vip.example.com
nc -vz oud-vip.example.com 1636
openssl s_client -connect oud-vip.example.com:1636 \
  -servername oud-vip.example.com </dev/null
```

### 3. Search the exact LDAP entry

```bash
"$OUD_BIN/ldapsearch" \
  -h oud-vip.example.com -p 1636 --useSSL \
  --trustStorePath /secure/oud/client-truststore \
  -D "cn=OracleNetReaders,dc=example,dc=com" \
  -j /secure/oud/net-reader.pwd \
  -b "cn=SALES,cn=OracleContext,dc=example,dc=com" -s base \
  "(objectClass=*)" dn objectClass cn orclNetDescString
```

If a direct authenticated search works but Oracle Net fails, verify how Oracle Net authenticates to the directory. Some estates allow tightly scoped anonymous reads for net service entries; others use wallet-based authenticated bind. Do not enable broad anonymous access as a quick fix.

### 4. Test without fallback masking

```bash
tnsping SALES
```

Use an LDAP-only alias or controlled `NAMES.DIRECTORY_PATH` so `TNSNAMES` or `EZCONNECT` cannot satisfy the request instead.

Frequent causes:

- wrong `TNS_ADMIN`/`LDAP_ADMIN` or Oracle home;
- wrong Oracle Context or environment;
- alias deleted, renamed, misspelled, or only present on another replica;
- LDAPS certificate not trusted by the Oracle client;
- client cannot bind/search because of ACI/authentication changes;
- local naming fallback hides different results across hosts;
- multiple `ldap.ora` files with unexpected search precedence;
- special characters in the requested name/DN not handled consistently.

## Alias resolves but database connection fails

If `tnsping SALES` resolves the LDAP entry but SQL connection fails, inspect the returned descriptor and move downstream:

```bash
getent hosts dbscan.example.com
nc -vz dbscan.example.com 1521
sqlplus /@SALES
```

| Error/symptom | Likely direction |
| --- | --- |
| ORA-12514 / listener does not know service | `SERVICE_NAME`, database registration, listener services. |
| ORA-12541 / no listener | Host/port, listener down, network path. |
| ORA-12170 / timeout | Firewall, routing, listener/database overload, connect timeout. |
| Authentication error | Database/user/wallet credential path, not LDAP naming. |
| Connects to wrong database | Stale/wrong `orclNetDescString`, local fallback, DNS/load-balancer target. |
| TCPS/wallet error after resolution | Oracle Net wallet, certificate chain/name, cipher/protocol. |

Confirm database listener services with the database administrator. OUD only returns the descriptor; it does not make the database service register.

## Alias change is missing on another node

1. Search the exact DN directly on the write node and each replica, bypassing the VIP.
2. Record `modifyTimestamp`/operational metadata only if authorized and useful.
3. Run `dsreplication status` and inspect missing-change count/age.
4. Confirm both endpoints belong to the same topology, suffix, and environment.
5. Check whether a proxy/VIP routes to an unexpected backend.
6. Check write result code; the client command may have failed even if the LDIF looked correct.
7. Inspect replication/error logs.

Do not repeat the change on each node. Independent writes can create conflicts and obscure replication failure.

## TNS alias add or modify fails

| LDAP result | Likely cause | Response |
| --- | --- | --- |
| `32` no such object | Oracle Context/parent absent or wrong DN | Discover the real context; do not create a parallel context casually. |
| `50` insufficient access | Delegated account lacks operation/attribute rights | Review effective ACI with directory security owner. |
| `53` unwilling | Read-only/internal-only writability, policy, proxy/backend restriction | Check chosen endpoint and back-end state. |
| `65` object class violation | Oracle schema missing, required attribute absent, descriptor syntax/attribute rule | Verify schema and LDIF; do not import random schema files. |
| `68` already exists | Alias/DN already present, including case-normalized collision | Search exact context and decide modify versus new approved name. |
| TLS/bind failure | Trust, certificate, identity, password/account state | Fix secure connection before retrying the write. |

For a modify, use `replace: orclNetDescString` only after exporting the old value. Confirm LDIF continuation lines start with one space and were not altered by email/ticket formatting.

## OUDSM is unavailable

First determine whether LDAP itself is healthy:

```bash
"$OUD_BIN/status"
"$OUD_BIN/ldapsearch" \
  -h oud01.example.com -p 1636 --useSSL \
  --trustStorePath /secure/oud/client-truststore \
  -D "cn=Directory Manager" -j /secure/oud/oud-bind.pwd \
  -s base -b "" "(objectClass=*)" namingContexts vendorVersion
```

If LDAP works, troubleshoot the OUDSM/WebLogic layer:

- Administration Server and managed-server status;
- Node Manager;
- deployment state and target;
- domain listen address/port and reverse proxy;
- OUDSM data source/connection configuration;
- WebLogic/OUDSM logs;
- patch post-steps and auto-redeployment requirements.

Do not restart directory servers to repair an OUDSM-only failure.

## Patch fails

### Before or during `opatch apply`

Capture:

```bash
"$ORACLE_HOME/OPatch/opatch" version
"$ORACLE_HOME/OPatch/opatch" lsinventory
df -h "$ORACLE_HOME" /tmp /stage
ps -ef | grep -E '[o]rg\.opends|[W]ebLogic|[N]odeManager'
```

Check exact README prerequisites, inventory validity, file permissions, free space, OPatch level, conflicting/subset patches, active processes, and whether the correct `ORACLE_HOME` is selected.

If a conflict is not explicitly documented as a superseded subset handled by the bundle patch, stop and use Oracle Support guidance. Do not force an apply or manually copy patched files.

### Patch applies but OUD does not start

Keep the endpoint drained. Collect OPatch logs, `lsinventory`, OUD errors, Java version/path, file ownership, and the patch README's post-steps. Decide between documented patch rollback and restoring the pre-patch environment. Do not advance to the next node.

## Upgrade fails

An OUD major upgrade is documented as non-reversible. Do not try to “downgrade” the modified instance in place.

Capture:

- source and target versions;
- `JAVA_HOME` and Java version;
- target installation mode;
- `upgrade-oud-instances` console output;
- `/tmp/preUpgradeActions-<timestamp>.log` or platform equivalent;
- `INSTANCE_DIR/OUD/logs/preUpgradeActions-<timestamp>.log`;
- `start-ds --upgrade` output and errors log;
- complete pre-upgrade backup inventory.

If the documented recovery condition is met, stop the upgrade and restore the entire pre-upgrade environment, then diagnose on a clone. Do not run the upgrade repeatedly against a partially modified production instance unless Oracle's procedure explicitly says it is resumable.

## Recovery and escalation package

Provide a concise timeline and these artifacts:

- architecture and affected endpoints/roles;
- exact OUD, bundle patch, OPatch, JDK, OS, and installation mode;
- instance/domain/Oracle home paths;
- `status` and replication output from all relevant nodes;
- Root DSE and minimal failing LDAP command with secrets removed;
- exact DN/base/scope/filter/attributes and LDAP result/diagnostic;
- errors, access, audit, replication, patch, or upgrade log slices covering the incident;
- CPU/memory/disk/inode/I/O/open-file evidence;
- certificate chain, subject alternative names, expiry, and trust-store identity;
- recent change records;
- backup inventory and last successful restore test;
- actions already taken and their observed results.

Redact passwords, private keys, tokens, and sensitive directory data. Preserve original files and timestamps; work on copies for redaction.
