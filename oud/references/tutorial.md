# Oracle Unified Directory Tutorial

## Contents

1. [Scope and assumptions](#scope-and-assumptions)
2. [Build the right mental model](#build-the-right-mental-model)
3. [Collect an environment handover](#collect-an-environment-handover)
4. [Locate the instance and tools](#locate-the-instance-and-tools)
5. [Run the first health check](#run-the-first-health-check)
6. [Learn the LDAP minimum](#learn-the-ldap-minimum)
7. [Understand Oracle Net names in OUD](#understand-oracle-net-names-in-oud)
8. [Discover the Oracle Context](#discover-the-oracle-context)
9. [Configure an Oracle client for directory naming](#configure-an-oracle-client-for-directory-naming)
10. [List and inspect TNS aliases](#list-and-inspect-tns-aliases)
11. [Create, change, and delete a TNS alias](#create-change-and-delete-a-tns-alias)
12. [Validate a TNS alias end to end](#validate-a-tns-alias-end-to-end)
13. [Start, stop, and restart OUD](#start-stop-and-restart-oud)
14. [Back up and restore](#back-up-and-restore)
15. [Patch OUD](#patch-oud)
16. [Upgrade OUD](#upgrade-oud)
17. [Monitor the service](#monitor-the-service)
18. [Operate securely](#operate-securely)
19. [Use a practical first-month learning path](#use-a-practical-first-month-learning-path)
20. [Complete a production acceptance check](#complete-a-production-acceptance-check)
21. [Official references](#official-references)

## Scope and assumptions

This guide is for an administrator inheriting an existing Oracle Unified Directory environment. It prioritizes inspection, Oracle Net directory naming, routine lifecycle operations, monitoring, patching, upgrades, and incident handling.

Examples use these documentation-only values:

| Purpose | Example |
| --- | --- |
| Directory server | `oud01.example.com` |
| Second replica | `oud02.example.com` |
| LDAP port | `1389` |
| LDAPS port | `1636` |
| Administration connector | `4444` |
| Instance directory | `/u01/app/oracle/config/oud/asinst_1` |
| Data suffix | `dc=example,dc=com` |
| Oracle Context | `cn=OracleContext,dc=example,dc=com` |
| Administration DN | `cn=Directory Manager` |
| Trust store | `/secure/oud/client-truststore` |
| Password file | `/secure/oud/oud-bind.pwd` |

Replace every value. Do not infer production ports, DNs, passwords, or certificate locations from these examples.

The commands reflect OUD 12c and 14c conventions, with 14.1.2.1 terminology where release-specific behavior matters. Confirm the command help, Oracle certification matrix, patch README, JDK requirements, and topology instructions for the installed release.

## Build the right mental model

OUD is an LDAP directory product, not an Oracle Database. It stores a hierarchical Directory Information Tree (DIT). Each entry has a distinguished name (DN), object classes, and attributes.

An environment can contain different OUD roles:

| Role | Purpose | Important limitation |
| --- | --- | --- |
| Directory server | Stores LDAP entries in local back ends. | Data commands and indexes apply here. |
| Proxy server | Routes or virtualizes requests to remote data sources. | It may not own the data returned to clients. |
| Replication server | Relays changes among replicas; often runs in a directory-server instance. | It is not itself a substitute for a writable directory replica. |
| Replication gateway | Connects OUD replication with supported legacy directory topologies. | Use gateway-specific procedures. |
| OUDSM | Web application for browser-based administration. | OUDSM availability is separate from the OUD LDAP daemon. |

A typical directory-naming request is:

```text
Oracle client
  -> reads sqlnet.ora and ldap.ora
  -> contacts OUD over LDAP/LDAPS
  -> searches cn=<connect_identifier>,cn=OracleContext,<admin_context>
  -> receives orclNetDescString
  -> contacts the database listener in that descriptor
```

This creates two independent paths to test:

1. Oracle client to OUD for name resolution.
2. Oracle client to the database listener for the actual database connection.

`tnsping` can prove name resolution and Oracle Net reachability, but it does not authenticate a database session.

## Collect an environment handover

Do not begin administration until these facts are documented:

| Area | Questions |
| --- | --- |
| Ownership | Which OS account owns the files and processes? Which team owns LDAP data, certificates, DNS, load balancers, and databases? |
| Release | What OUD version, bundle patch, OPatch version, JDK, and operating system are used? |
| Installation | Is each instance standalone or collocated with a WebLogic domain? Which instances share an `ORACLE_HOME`? |
| Topology | Which nodes are directory servers, proxies, replication servers, gateways, or OUDSM hosts? |
| Network | What are the LDAP, LDAPS, administration, replication, HTTP/JMX, VIP, and health-check endpoints? |
| Data | What are the base DNs, back-end IDs, Oracle Context DNs, entry counts, and data owners? |
| Replication | What are the replication groups, expected peers, initialization sources, purge delay, and normal latency? |
| Security | Which trust stores, key stores, certificates, ACIs, delegated accounts, password vaults, and audit requirements apply? |
| Service management | Are systemd, init scripts, Node Manager, WebLogic component scripts, or clusterware used? |
| Recovery | Where are backups stored, how are they tested, and what are the recovery point and recovery time objectives? |
| Monitoring | Which platform consumes OUD status, logs, alerts, SNMP, JMX, or `cn=monitor` metrics? |

Ask for the current architecture diagram, operating procedures, last patch record, last restore test, and a list of known exceptions. Verify the handover against the live read-only state.

## Locate the instance and tools

Run as the approved OUD software owner:

```bash
id
ps -ef | grep -E '[o]rg\.opends|[o]racle\.directory|[s]tart-ds'
find /u01/app/oracle -type f -path '*/OUD/bin/status' -print 2>/dev/null
```

After identifying the path, define task-specific variables:

```bash
export OUD_INSTANCE=/u01/app/oracle/config/oud/asinst_1
export OUD_BIN="$OUD_INSTANCE/OUD/bin"

test -x "$OUD_BIN/status"
"$OUD_BIN/status" --version
java -version
```

Common instance locations include:

| Path | Contents |
| --- | --- |
| `INSTANCE_DIR/OUD/bin` | UNIX/Linux administration commands. |
| `INSTANCE_DIR/OUD/bat` | Windows administration commands. |
| `INSTANCE_DIR/OUD/config` | Server configuration, schema, Java properties, key/trust stores depending on setup. |
| `INSTANCE_DIR/OUD/logs` | Server PID and configured log files, commonly access, errors, audit, and replication repair logs. |
| `INSTANCE_DIR/OUD/db` | Local back-end database files. Do not manipulate them with OS tools. |
| `INSTANCE_DIR/OUD/bak` | Possible default or local backup area; confirm the configured practice. |

Do not assume `ORACLE_HOME` and `INSTANCE_DIR` are the same. Modern and upgraded installations should keep instance or domain data outside the patchable Oracle home.

For a collocated instance, identify the domain and component name:

```bash
export DOMAIN_HOME=/u01/app/oracle/config/domains/oud_domain
ls -l "$DOMAIN_HOME/bin/startComponent.sh" "$DOMAIN_HOME/bin/stopComponent.sh"
```

## Run the first health check

### Check local service status

```bash
"$OUD_BIN/status"
```

For complete online information, authenticate through the administration connector. Use the trust store approved for the environment:

```bash
"$OUD_BIN/status" \
  --hostname oud01.example.com \
  --port 4444 \
  --bindDN "cn=Directory Manager" \
  --bindPasswordFile /secure/oud/oud-bind.pwd \
  --trustStorePath /secure/oud/client-truststore \
  --no-prompt
```

Review:

- server started/stopped state;
- server version;
- administration, LDAP, and LDAPS handlers;
- base DNs and back-end IDs;
- entry counts;
- replication enabled state, missing changes, and age of the oldest missing change.

### Check sockets, filesystem, and logs

```bash
ss -ltnp
df -h "$OUD_INSTANCE"
df -i "$OUD_INSTANCE"
ps -ef | grep -E '[o]rg\.opends|[j]ava'
ls -ltr "$OUD_INSTANCE/OUD/logs" | tail -30
tail -100 "$OUD_INSTANCE/OUD/logs/errors" 2>/dev/null
```

Use the configured log filenames rather than assuming every installation has the same set.

### Check LDAP and LDAPS

Root DSE is a low-cost functional test:

```bash
"$OUD_BIN/ldapsearch" \
  -h oud01.example.com -p 1636 --useSSL \
  --trustStorePath /secure/oud/client-truststore \
  -D "cn=Directory Manager" -j /secure/oud/oud-bind.pwd \
  -s base -b "" "(objectClass=*)" \
  namingContexts supportedLDAPVersion vendorName vendorVersion
```

If policy permits anonymous Root DSE reads, omit bind credentials. Do not assume anonymous access is enabled for application data.

### Check replication

```bash
"$OUD_BIN/dsreplication" status \
  --hostname oud01.example.com \
  --port 4444 \
  --adminUID admin \
  --adminPasswordFile /secure/oud/oud-admin.pwd \
  --trustStorePath /secure/oud/client-truststore \
  --no-prompt
```

Healthy steady state normally means all expected nodes are present, statuses are `Normal` or `Up`, missing changes are zero or transiently small, and entry counts converge. Establish environment-specific thresholds instead of alerting on a single universal number.

## Learn the LDAP minimum

An entry looks like this:

```ldif
dn: cn=SALES,cn=OracleContext,dc=example,dc=com
objectClass: top
objectClass: orclNetService
cn: SALES
orclNetDescString: (DESCRIPTION=(ADDRESS=(PROTOCOL=TCP)(HOST=dbscan.example.com)(PORT=1521))(CONNECT_DATA=(SERVICE_NAME=sales.example.com)))
```

Key terms:

| Term | Meaning |
| --- | --- |
| DN | Unique path of an entry, such as `cn=SALES,cn=OracleContext,dc=example,dc=com`. |
| RDN | First component of a DN, such as `cn=SALES`. |
| Base DN | Point at which a search starts. |
| Scope | `base` for one entry, `one` for direct children, or `sub` for the full subtree. |
| Filter | Boolean selection, such as `(objectClass=orclNetService)` or `(&(objectClass=orclNetService)(cn=SALES))`. |
| Object class | Schema rule defining allowed or required attributes. |
| Attribute | Named value such as `cn` or `orclNetDescString`. |
| LDIF | Text format used to represent entries and changes. |
| ACI | Access control instruction governing LDAP operations. |

Distinguished names require escaping for special characters. Use the existing naming standard and a directory-aware tool when a connect identifier contains commas, plus signs, leading/trailing spaces, or other DN-special characters.

## Understand Oracle Net names in OUD

Directory naming centralizes Oracle connect identifiers. The common structure is:

```text
dc=example,dc=com
└── cn=OracleContext
    ├── cn=SALES        objectClass=orclNetService
    └── cn=REPORTING    objectClass=orclNetService
```

The client uses `DEFAULT_ADMIN_CONTEXT="dc=example,dc=com"`, then searches within `cn=OracleContext` for the requested connect identifier.

The main attributes are:

| Attribute/object class | Purpose |
| --- | --- |
| `orclNetService` | Object class for a net service name. |
| `cn` | Connect identifier visible to clients. |
| `orclNetDescString` | Oracle Net connect descriptor returned to the client. |
| `orclNetServiceAlias` | Optional LDAP alias object used in some legacy designs. It is not the same thing as a normal net service name. |

Before administering aliases, confirm that the Oracle schema and Oracle Context already exist. Schema provisioning and Oracle Context creation are integration tasks, not routine alias operations.

## Discover the Oracle Context

First inspect a representative client configuration:

```bash
printf 'TNS_ADMIN=%s\nLDAP_ADMIN=%s\n' "$TNS_ADMIN" "$LDAP_ADMIN"
sed -n '1,160p' "$TNS_ADMIN/ldap.ora"
sed -n '1,160p' "$TNS_ADMIN/sqlnet.ora"
```

Then search the directory for Oracle Context entries:

```bash
"$OUD_BIN/ldapsearch" \
  -h oud01.example.com -p 1636 --useSSL \
  --trustStorePath /secure/oud/client-truststore \
  -D "cn=Directory Manager" -j /secure/oud/oud-bind.pwd \
  -b "dc=example,dc=com" -s sub \
  "(cn=OracleContext)" dn objectClass
```

Confirm all of these before writing:

- the returned DN equals `cn=OracleContext,<DEFAULT_ADMIN_CONTEXT>`;
- the server exposes the Oracle schema and accepts `orclNetService` entries;
- the planned account has search/add/modify/delete rights only where needed;
- the suffix is replicated and writable through the chosen endpoint;
- the naming convention for `cn` and service descriptors is understood.

## Configure an Oracle client for directory naming

Use Oracle Net Configuration Assistant where that is the supported local process. A typical `ldap.ora` resembles:

```text
DIRECTORY_SERVERS = (oud-vip.example.com:1389:1636)
DEFAULT_ADMIN_CONTEXT = "dc=example,dc=com"
DIRECTORY_SERVER_TYPE = OID
```

`DIRECTORY_SERVERS` values use `host:port[:sslport]`. Multiple endpoints can be listed. For an OUD directory provisioned with Oracle schema, Oracle Net uses the OID-compatible directory type value.

Select LDAP in `sqlnet.ora`:

```text
NAMES.DIRECTORY_PATH = (LDAP, TNSNAMES, EZCONNECT)
```

Naming-method order matters. A local `tnsnames.ora` entry can hide a broken or stale LDAP entry when `TNSNAMES` precedes `LDAP`. During a controlled test, use an alias that exists only in OUD or temporarily test with LDAP first.

Protect TLS settings and wallets according to the Oracle client release. Confirm how the client validates the LDAPS certificate; do not solve certificate errors by disabling verification permanently.

## List and inspect TNS aliases

List aliases under the exact Oracle Context:

```bash
"$OUD_BIN/ldapsearch" \
  -h oud01.example.com -p 1636 --useSSL \
  --trustStorePath /secure/oud/client-truststore \
  -D "cn=Directory Manager" -j /secure/oud/oud-bind.pwd \
  -b "cn=OracleContext,dc=example,dc=com" -s one \
  "(objectClass=orclNetService)" dn cn orclNetDescString
```

Inspect one alias:

```bash
"$OUD_BIN/ldapsearch" \
  -h oud01.example.com -p 1636 --useSSL \
  --trustStorePath /secure/oud/client-truststore \
  -D "cn=Directory Manager" -j /secure/oud/oud-bind.pwd \
  -b "cn=SALES,cn=OracleContext,dc=example,dc=com" -s base \
  "(objectClass=*)" dn objectClass cn orclNetDescString
```

Search by name when the exact DN is not known:

```bash
"$OUD_BIN/ldapsearch" \
  -h oud01.example.com -p 1636 --useSSL \
  --trustStorePath /secure/oud/client-truststore \
  -D "cn=Directory Manager" -j /secure/oud/oud-bind.pwd \
  -b "cn=OracleContext,dc=example,dc=com" -s one \
  "(&(objectClass=orclNetService)(cn=SALES))" dn cn orclNetDescString
```

Use a delegated read account for routine inventory instead of Directory Manager.

## Create, change, and delete a TNS alias

### Change procedure

For every change:

1. Record the change ticket, owner, maintenance window, and affected applications.
2. Check replication and verify that the chosen endpoint is writable.
3. Search for exact-name and case-insensitive collisions.
4. Export the current entry for a modify/delete.
5. Review the descriptor with the database/network owner.
6. Apply the LDIF once.
7. Search the entry on the write node and a second replica.
8. Test resolution and a real database connection from a representative client.
9. Roll back if success criteria fail.

### Add an alias

Create `add-sales.ldif`:

```ldif
dn: cn=SALES,cn=OracleContext,dc=example,dc=com
changetype: add
objectClass: top
objectClass: orclNetService
cn: SALES
orclNetDescString: (DESCRIPTION=(ADDRESS=(PROTOCOL=TCP)(HOST=dbscan.example.com)(PORT=1521))(CONNECT_DATA=(SERVICE_NAME=sales.example.com)))
```

Apply with a delegated naming administrator:

```bash
"$OUD_BIN/ldapmodify" \
  -h oud01.example.com -p 1636 --useSSL \
  --trustStorePath /secure/oud/client-truststore \
  -D "cn=OracleNetAdmins,dc=example,dc=com" \
  -j /secure/oud/net-admin.pwd \
  -f add-sales.ldif
```

Rollback LDIF:

```ldif
dn: cn=SALES,cn=OracleContext,dc=example,dc=com
changetype: delete
```

### Change a descriptor

Before the change, capture the current entry on secure storage:

```bash
"$OUD_BIN/ldapsearch" \
  -h oud01.example.com -p 1636 --useSSL \
  --trustStorePath /secure/oud/client-truststore \
  -D "cn=OracleNetAdmins,dc=example,dc=com" \
  -j /secure/oud/net-admin.pwd \
  -b "cn=SALES,cn=OracleContext,dc=example,dc=com" -s base \
  "(objectClass=*)" dn objectClass cn orclNetDescString \
  > sales-before.ldif
```

Create `modify-sales.ldif`:

```ldif
dn: cn=SALES,cn=OracleContext,dc=example,dc=com
changetype: modify
replace: orclNetDescString
orclNetDescString: (DESCRIPTION=(ADDRESS_LIST=(LOAD_BALANCE=ON)(FAILOVER=ON)(ADDRESS=(PROTOCOL=TCP)(HOST=dbscan-a.example.com)(PORT=1521))(ADDRESS=(PROTOCOL=TCP)(HOST=dbscan-b.example.com)(PORT=1521)))(CONNECT_DATA=(SERVICE_NAME=sales.example.com)))
```

Apply it with the same `ldapmodify` pattern. Build rollback LDIF with `changetype: modify`, `replace: orclNetDescString`, and the exact old descriptor from the clean pre-change export.

### Delete an alias

Confirm it is unused and capture the full entry first. Create `delete-sales.ldif`:

```ldif
dn: cn=SALES,cn=OracleContext,dc=example,dc=com
changetype: delete
```

Apply with `ldapmodify`. Rollback is an `add` LDIF containing only the required object classes, `cn`, and the previous `orclNetDescString`; do not blindly re-add operational attributes from search output.

### Rename an alias

A rename changes the DN and `cn`. Treat it as a compatibility change because clients may continue using the old name. Prefer a staged migration:

1. Add the new `orclNetService` entry with the approved descriptor.
2. Validate the new name.
3. Migrate clients.
4. Observe use for the agreed period.
5. Remove the old name only after confirming it is unused.

This is safer than an immediate mod-DN for a beginner-operated production environment.

## Validate a TNS alias end to end

### Verify LDAP on two replicas

Run the single-entry search against `oud01` and `oud02`. The DN and descriptor must match.

### Verify Oracle client configuration

```bash
printf 'ORACLE_HOME=%s\nTNS_ADMIN=%s\nLDAP_ADMIN=%s\n' \
  "$ORACLE_HOME" "$TNS_ADMIN" "$LDAP_ADMIN"
command -v tnsping
tnsping SALES
```

Check the client reads the intended `sqlnet.ora` and `ldap.ora`. If `SALES` also exists locally, rename the local test entry or use a unique LDAP-only test name so the fallback cannot mask a failure.

### Verify a database session

Use an approved credential mechanism that does not expose a password on the command line, such as a secure external password store:

```bash
sqlplus /@SALES
```

Validate the resulting service and host where policy allows:

```sql
SELECT
  SYS_CONTEXT('USERENV', 'SERVICE_NAME') AS service_name,
  SYS_CONTEXT('USERENV', 'SERVER_HOST')  AS server_host
FROM dual;
```

Resolution success plus database-session success is the completion criterion.

## Start, stop, and restart OUD

### Standalone instance

```bash
# Read-only status
"$OUD_BIN/status"

# Start
"$OUD_BIN/start-ds"

# Graceful local stop
"$OUD_BIN/stop-ds" --stopReason "Approved maintenance CHG-12345"

# Full process restart
"$OUD_BIN/stop-ds" --stopReason "Approved restart CHG-12345"
"$OUD_BIN/start-ds"
```

`stop-ds --restart` with authenticated remote options performs an in-core restart, so changes requiring a new JVM do not take effect. Use a real stop followed by start for JDK, JVM, binary, and many patching changes.

### Collocated instance

Use the domain component scripts and the component name:

```bash
"$DOMAIN_HOME/bin/stopComponent.sh" oud1
"$DOMAIN_HOME/bin/startComponent.sh" oud1
```

Patching or domain maintenance may also require Node Manager, Administration Server, OUDSM, or other managed servers to stop in a specific order. Follow the exact patch README and local domain runbook.

### Maintenance sequence

1. Confirm a valid change and rollback plan.
2. Verify client failover and healthy replication.
3. Remove or drain one endpoint from the load balancer.
4. Wait for or assess active connections according to local policy.
5. Capture status, replication, logs, and disk state.
6. Stop through the supported service mechanism.
7. Perform maintenance.
8. Start and verify Root DSE, suffix search, replication convergence, TLS, and a representative TNS resolution.
9. Return the endpoint to service.
10. Observe before moving to the next node.

## Back up and restore

Replication improves availability but does not replace a backup: unwanted deletes and corrupt changes can replicate too.

List back ends:

```bash
"$OUD_BIN/list-backends"
```

Back up all configured back ends to approved storage:

```bash
"$OUD_BIN/backup" \
  --backUpAll \
  --compress \
  --backupDirectory /backup/oud/asinst_1/2026-08-24
```

Also preserve the instance configuration, schema, certificates or references to their source, service definitions, and Oracle home inventory as required by the recovery design. A back-end backup alone is not a complete environment rebuild plan.

For restore:

- identify the exact back-end ID and backup ID;
- verify integrity and retention;
- decide whether replication re-initialization is safer than restoring one replica;
- stop the server for a normal offline restore unless following a documented task-based exception;
- restore only one back end at a time with `restore`;
- prevent a stale restored replica from accepting writes until it is safely reconciled;
- verify entry counts, application searches, replication, and TNS resolution.

Do not provide or execute a generic restore command without the backup inventory and authoritative-source decision. Restore is intentionally a planned recovery operation.

## Patch OUD

A bundle patch changes binaries in `ORACLE_HOME`; it is different from a major-version upgrade.

### Before patching

1. Read the patch README for the exact OUD base release and patch revision.
2. Check prerequisites, superseded patches, conflicts, required OPatch level, JDK, platform, free space, and post-patch steps.
3. Map every standalone instance and collocated domain using the target `ORACLE_HOME`.
4. Verify replication and client failover.
5. Take the backups required by the README. Oracle OUD bundle-patch guidance calls for at least an instance configuration backup and recommends a complete cold instance backup when regular backups are absent.
6. Validate inventory:

```bash
export ORACLE_HOME=/u01/app/oracle/product/oud/14.1.2.1
"$ORACLE_HOME/OPatch/opatch" version
"$ORACLE_HOME/OPatch/opatch" lsinventory
```

7. Stage the patch outside `ORACLE_HOME`; checksum it and protect permissions.

### Apply the patch

The exact README controls the stop scope and command. A common standalone pattern is:

```bash
# Stop every instance using this ORACLE_HOME as required by the README.
"$OUD_BIN/stop-ds" --stopReason "OUD bundle patch CHG-12345"

cd /stage/PATCH_ID
"$ORACLE_HOME/OPatch/opatch" apply
```

For a collocated installation, the README may require stopping the OUD component, Node Manager, WebLogic Administration Server, OUDSM, and related managed servers. Do not reuse the standalone sequence blindly.

### After patching

```bash
"$ORACLE_HOME/OPatch/opatch" lsinventory
"$ORACLE_HOME/OPatch/opatch" lspatches
"$OUD_BIN/start-ds"
"$OUD_BIN/status"
```

Then validate:

- patched version and inventory;
- errors log from startup onward;
- LDAP and LDAPS Root DSE;
- suffix search and write through a controlled test if approved;
- replication status and convergence;
- certificates and expiry;
- proxy/load balancer health;
- TNS alias lookup and a representative database connection.

Rollback only with the procedure and patch ID documented by the README. A binary rollback may also require restoring configuration or data; decide this before patching.

## Upgrade OUD

A major upgrade must be designed as a topology project. Oracle 14.1.2.1 documentation describes upgrading from 12.2.1.4 with new 14c software, JDK 17, instance preparation, and an instance upgrade. It warns that the upgrade is not reversible; recovery is restoration of the complete pre-upgrade environment.

At a high level:

1. Verify the supported source release and direct upgrade path.
2. Read OUD, Fusion Middleware Infrastructure, OUDSM, and application integration upgrade requirements.
3. Build and test a representative clone.
4. Back up `ORACLE_HOME`, every OUD instance or domain, data, configuration, schema, certificates, and service definitions.
5. Provide redundant service and upgrade one node at a time where the documented topology permits.
6. Install the target software in the supported mode matching the existing standalone/collocated installation.
7. Set the target JDK and update documented JDK references.
8. Run the target release's instance preparation command. For the documented 12.2.1.4 to 14.1.2.1 path, this includes:

```bash
./upgrade-oud-instances --instancePath /path/to/12c-instance
```

9. Run the instance upgrade from the upgraded instance as documented:

```bash
/path/to/instance/OUD/bin/start-ds --upgrade
```

10. Review the main and instance upgrade logs, start normally, validate the complete acceptance suite, and observe before upgrading another node.

Do not copy these two commands into another source/target release pair. Upgrade tooling and supported paths are release-specific.

## Monitor the service

### Basic status

```bash
"$OUD_BIN/status" --script-friendly
"$OUD_BIN/status" --refresh 30
```

### Replication

```bash
"$OUD_BIN/dsreplication" status \
  --hostname oud01.example.com --port 4444 \
  --adminUID admin \
  --adminPasswordFile /secure/oud/oud-admin.pwd \
  --trustStorePath /secure/oud/client-truststore \
  --no-prompt
```

Alert on persistent missing changes, increasing oldest-missing-change age, `Late`, `Bad Data Set`, `Not Connected`, `Down`, or `Unknown`, unexpected topology members, and entry counts that do not converge.

### LDAP monitor tree

Read the monitor root:

```bash
"$OUD_BIN/ldapsearch" \
  -h oud01.example.com -p 4444 --useSSL \
  --trustStorePath /secure/oud/client-truststore \
  -D "cn=Directory Manager" -j /secure/oud/oud-bind.pwd \
  -s base -b "cn=monitor" "(objectClass=*)" \
  currentTime startTime upTime currentConnections totalConnections \
  maxConnections vendorName vendorVersion
```

Discover available monitor branches without retrieving every metric:

```bash
"$OUD_BIN/ldapsearch" \
  -h oud01.example.com -p 4444 --useSSL \
  --trustStorePath /secure/oud/client-truststore \
  -D "cn=Directory Manager" -j /secure/oud/oud-bind.pwd \
  -s sub -b "cn=monitor" "(objectClass=*)" "1.1"
```

Useful branches include JVM memory, work queue, entry caches, back ends, connection handlers, client connections, disk-space monitor, and replication. Restrict access because connection monitoring can reveal client addresses and bind DNs.

### Operating-system signals

Monitor CPU, resident memory, JVM heap and garbage collection, open files, disk capacity and inodes, I/O latency, process restarts, listening sockets, certificate expiry, and NTP/time synchronization. Establish baselines for search/add/modify response time and result codes from access logs.

## Operate securely

- Use LDAPS or StartTLS for credentials and administration traffic.
- Use a trust store that validates the expected server chain and host name.
- Use `--trustAll` only for time-bounded diagnosis after independently checking the certificate fingerprint.
- Use `--bindPasswordFile`/`-j`; never use clear-text `-w` in shell history, process listings, tickets, or examples.
- Store password files outside the repository with owner-only permissions and source them from the approved secret system.
- Delegate TNS alias rights to the Oracle Context subtree instead of sharing Directory Manager.
- Audit add, modify, delete, bind failures, configuration changes, and privileged operations according to policy.
- Back up key stores, trust stores, schema, and configuration consistently with the data recovery plan.
- Rotate certificates before expiry and test both OUD administration tools and Oracle Net clients; they may use different trust stores.

## Use a practical first-month learning path

### Day 1: Read-only orientation

- Identify roles, nodes, paths, ports, service control, base DNs, and the Oracle Context.
- Run `status`, Root DSE search, one application search, and `dsreplication status`.
- Locate logs and dashboards.

### Days 2-3: Oracle Net naming

- Trace one known alias from `sqlnet.ora` and `ldap.ora` to its `orclNetService` entry.
- Compare the descriptor with listener/database service configuration.
- Test a nonproduction add, modify, rollback, and delete using LDIF.

### Week 1: Operations

- Observe or rehearse start/stop on a nonproduction node.
- Review backup jobs and prove a backup can be listed and read.
- Learn the normal replication topology and latency baseline.

### Weeks 2-3: Troubleshooting

- Rehearse LDAP/LDAPS, ACI, certificate, index, disk, replication, and Oracle Net resolution incidents.
- Learn access-log result codes and error-log alert patterns.

### Week 4: Maintenance

- Review the last patch evidence and target patch README.
- Perform a nonproduction rolling patch rehearsal and acceptance test.
- Review the major-upgrade design separately; never learn upgrades on production.

## Complete a production acceptance check

- [ ] Exact release, bundle patch, JDK, role, installation mode, and paths are recorded.
- [ ] Every OUD node, proxy/VIP, replication link, and shared Oracle home is mapped.
- [ ] Standalone or collocated lifecycle procedure is proven.
- [ ] LDAP and LDAPS Root DSE checks succeed with certificate validation.
- [ ] Application suffix searches succeed through each service endpoint.
- [ ] Replication peers are expected, status is healthy, and counts converge.
- [ ] Oracle Context and representative `orclNetService` entries are readable.
- [ ] A directory-only `tnsping` and database connection succeed from a representative client.
- [ ] Logs, monitoring, alerts, disk thresholds, and certificate-expiry alerts are active.
- [ ] Backups include data, configuration, schema, and security material; restore is tested.
- [ ] Delegated accounts and ACIs follow least privilege.
- [ ] Patch and upgrade procedures identify exact README, downtime, failover, rollback/recovery, and verification.

## Official references

- [Oracle Unified Directory 14.1.2 documentation](https://docs.oracle.com/en/middleware/idm/unified-directory/14.1.2/)
- [OUD command-line interface reference](https://docs.oracle.com/en/middleware/idm/unified-directory/14.1.2/oudag/oracle-unified-directory-command-line-interface-reference.html)
- [Managing directory data, backups, and restores](https://docs.oracle.com/en/middleware/idm/unified-directory/14.1.2/oudag/managing-directory-data.html)
- [Monitoring Oracle Unified Directory](https://docs.oracle.com/en/middleware/idm/unified-directory/14.1.2/oudag/monitoring-oracle-unified-directory.html)
- [Upgrading the OUD software](https://docs.oracle.com/en/middleware/idm/unified-directory/14.1.2/oudig/updating-oracle-unified-directory-software.html)
- [Oracle Net `ldap.ora` parameters](https://docs.oracle.com/en/database/oracle/oracle-database/19/netrf/directory-usage-parameters-in-ldap-ora-file.html)
- [Oracle Net ORA-12154 directory-naming checks](https://docs.oracle.com/en/database/oracle/oracle-database/26/netag/ora-12154-could-not-resolve-connect-identifier-specified.html)
- [Oracle Net naming entry example using `orclNetService`](https://docs.oracle.com/en/database/oracle/oracle-database/26/netrf/configure-openldap-server-use-oracle-net-naming-directory.html)
