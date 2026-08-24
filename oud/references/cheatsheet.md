# Oracle Unified Directory Cheatsheet

## Contents

1. [Set the environment](#set-the-environment)
2. [Identify the deployment](#identify-the-deployment)
3. [Daily health checks](#daily-health-checks)
4. [Lifecycle](#lifecycle)
5. [LDAP searches](#ldap-searches)
6. [TNS alias inventory](#tns-alias-inventory)
7. [TNS alias changes](#tns-alias-changes)
8. [Oracle client configuration](#oracle-client-configuration)
9. [Replication](#replication)
10. [Configuration with dsconfig](#configuration-with-dsconfig)
11. [Backups, exports, and tasks](#backups-exports-and-tasks)
12. [Monitoring](#monitoring)
13. [Patching checklist](#patching-checklist)
14. [Upgrade boundary](#upgrade-boundary)
15. [Logs and evidence](#logs-and-evidence)
16. [Safety reminders](#safety-reminders)

## Set the environment

Use the actual instance path; do not assume it equals `ORACLE_HOME`.

```bash
export OUD_INSTANCE=/u01/app/oracle/config/oud/asinst_1
export OUD_BIN="$OUD_INSTANCE/OUD/bin"
export ORACLE_HOME=/u01/app/oracle/product/oud/14.1.2.1

test -x "$OUD_BIN/status"
"$OUD_BIN/status" --version
java -version
```

Reusable secure-connection options in the examples:

```text
-h oud01.example.com -p 1636 --useSSL
--trustStorePath /secure/oud/client-truststore
-D "cn=Directory Manager" -j /secure/oud/oud-bind.pwd
```

Prefer a delegated DN. `cn=Directory Manager` is shown only as a recognizable placeholder.

## Identify the deployment

```bash
id
ps -ef | grep -E '[o]rg\.opends|[o]racle\.directory|[j]ava'
find /u01/app/oracle -type f -path '*/OUD/bin/status' -print 2>/dev/null
ss -ltnp
df -h "$OUD_INSTANCE"
df -i "$OUD_INSTANCE"
```

| Question | Evidence |
| --- | --- |
| Standalone or collocated? | Instance scripts versus `DOMAIN_HOME/bin/*Component.sh`; deployment records. |
| Directory or proxy? | `status`, configured back ends/suffixes, topology documentation. |
| Which ports? | `status`, `ss`, connection handlers from `dsconfig`. |
| Which suffixes? | `status`, `list-backends`, Root DSE `namingContexts`. |
| Which patch? | `status --version`, `opatch lsinventory`, patch records. |
| Which replicas? | `dsreplication status`. |
| Which Oracle Context? | Client `DEFAULT_ADMIN_CONTEXT` plus LDAP search for `cn=OracleContext`. |

## Daily health checks

```bash
# Basic local state
"$OUD_BIN/status"

# Root DSE over LDAPS
"$OUD_BIN/ldapsearch" \
  -h oud01.example.com -p 1636 --useSSL \
  --trustStorePath /secure/oud/client-truststore \
  -D "cn=Directory Manager" -j /secure/oud/oud-bind.pwd \
  -s base -b "" "(objectClass=*)" \
  namingContexts vendorName vendorVersion supportedLDAPVersion

# Replication
"$OUD_BIN/dsreplication" status \
  --hostname oud01.example.com --port 4444 \
  --adminUID admin \
  --adminPasswordFile /secure/oud/oud-admin.pwd \
  --trustStorePath /secure/oud/client-truststore \
  --no-prompt

# Recent errors and capacity
tail -100 "$OUD_INSTANCE/OUD/logs/errors" 2>/dev/null
df -h "$OUD_INSTANCE"
df -i "$OUD_INSTANCE"
```

Healthy expectations:

- expected process and listeners are present;
- Root DSE and one representative suffix search succeed;
- TLS chain and host name validate;
- replicas are `Normal`/`Up` and converge;
- no persistent missing changes or growing oldest-change age;
- disk, inodes, heap, open files, and work queue have headroom;
- error and access logs show no new recurring failure pattern.

## Lifecycle

### Standalone

```bash
"$OUD_BIN/status"
"$OUD_BIN/start-ds"
"$OUD_BIN/stop-ds" --stopReason "Approved maintenance CHG-12345"
```

Full process restart:

```bash
"$OUD_BIN/stop-ds" --stopReason "Approved restart CHG-12345"
"$OUD_BIN/start-ds"
```

Foreground start for controlled diagnosis:

```bash
"$OUD_BIN/start-ds" -N
```

`stop-ds --restart` with authentication can reinitialize inside the same JVM. It is not a substitute for a full stop/start when JVM, JDK, or binaries changed.

### Collocated

```bash
export DOMAIN_HOME=/u01/app/oracle/config/domains/oud_domain
"$DOMAIN_HOME/bin/stopComponent.sh" oud1
"$DOMAIN_HOME/bin/startComponent.sh" oud1
```

Use the domain runbook for Node Manager, Administration Server, OUDSM, and managed-server ordering.

### Before any restart

```bash
date -Is
"$OUD_BIN/status"
"$OUD_BIN/dsreplication" status --help
df -h "$OUD_INSTANCE"
df -i "$OUD_INSTANCE"
ps -ef | grep -E '[o]rg\.opends|[j]ava'
ss -ltnp
tail -200 "$OUD_INSTANCE/OUD/logs/errors" 2>/dev/null
```

Capture the real replication-status command with approved credentials. Do not restart away the evidence.

## LDAP searches

### Root DSE

```bash
"$OUD_BIN/ldapsearch" \
  -h oud01.example.com -p 1636 --useSSL \
  --trustStorePath /secure/oud/client-truststore \
  -D "cn=Directory Manager" -j /secure/oud/oud-bind.pwd \
  -s base -b "" "(objectClass=*)" namingContexts vendorVersion
```

### Base, one-level, and subtree

```bash
# One exact entry
"$OUD_BIN/ldapsearch" -h oud01.example.com -p 1636 --useSSL \
  --trustStorePath /secure/oud/client-truststore \
  -D "cn=Directory Manager" -j /secure/oud/oud-bind.pwd \
  -b "uid=user1,ou=People,dc=example,dc=com" -s base \
  "(objectClass=*)" dn uid cn

# Direct children
"$OUD_BIN/ldapsearch" -h oud01.example.com -p 1636 --useSSL \
  --trustStorePath /secure/oud/client-truststore \
  -D "cn=Directory Manager" -j /secure/oud/oud-bind.pwd \
  -b "ou=People,dc=example,dc=com" -s one \
  "(objectClass=person)" dn uid cn

# Entire subtree
"$OUD_BIN/ldapsearch" -h oud01.example.com -p 1636 --useSSL \
  --trustStorePath /secure/oud/client-truststore \
  -D "cn=Directory Manager" -j /secure/oud/oud-bind.pwd \
  -b "dc=example,dc=com" -s sub \
  "(&(objectClass=person)(uid=user1))" dn uid cn
```

### Discover monitor branches without values

```bash
"$OUD_BIN/ldapsearch" \
  -h oud01.example.com -p 4444 --useSSL \
  --trustStorePath /secure/oud/client-truststore \
  -D "cn=Directory Manager" -j /secure/oud/oud-bind.pwd \
  -s sub -b "cn=monitor" "(objectClass=*)" "1.1"
```

## TNS alias inventory

Define the real context:

```bash
export ORACLE_CONTEXT='cn=OracleContext,dc=example,dc=com'
```

Find Oracle Context entries:

```bash
"$OUD_BIN/ldapsearch" \
  -h oud01.example.com -p 1636 --useSSL \
  --trustStorePath /secure/oud/client-truststore \
  -D "cn=Directory Manager" -j /secure/oud/oud-bind.pwd \
  -b "dc=example,dc=com" -s sub \
  "(cn=OracleContext)" dn objectClass
```

List all net service names:

```bash
"$OUD_BIN/ldapsearch" \
  -h oud01.example.com -p 1636 --useSSL \
  --trustStorePath /secure/oud/client-truststore \
  -D "cn=Directory Manager" -j /secure/oud/oud-bind.pwd \
  -b "$ORACLE_CONTEXT" -s one \
  "(objectClass=orclNetService)" dn cn orclNetDescString
```

Inspect `SALES`:

```bash
"$OUD_BIN/ldapsearch" \
  -h oud01.example.com -p 1636 --useSSL \
  --trustStorePath /secure/oud/client-truststore \
  -D "cn=Directory Manager" -j /secure/oud/oud-bind.pwd \
  -b "cn=SALES,$ORACLE_CONTEXT" -s base \
  "(objectClass=*)" dn objectClass cn orclNetDescString
```

Find exact-name collision:

```bash
"$OUD_BIN/ldapsearch" \
  -h oud01.example.com -p 1636 --useSSL \
  --trustStorePath /secure/oud/client-truststore \
  -D "cn=Directory Manager" -j /secure/oud/oud-bind.pwd \
  -b "$ORACLE_CONTEXT" -s one \
  "(&(objectClass=orclNetService)(cn=SALES))" dn cn
```

## TNS alias changes

Write once through one writable endpoint; replication distributes the entry. Verify on a second node.

### Add

```ldif
dn: cn=SALES,cn=OracleContext,dc=example,dc=com
changetype: add
objectClass: top
objectClass: orclNetService
cn: SALES
orclNetDescString: (DESCRIPTION=(ADDRESS=(PROTOCOL=TCP)(HOST=dbscan.example.com)(PORT=1521))(CONNECT_DATA=(SERVICE_NAME=sales.example.com)))
```

```bash
"$OUD_BIN/ldapmodify" \
  -h oud01.example.com -p 1636 --useSSL \
  --trustStorePath /secure/oud/client-truststore \
  -D "cn=OracleNetAdmins,dc=example,dc=com" \
  -j /secure/oud/net-admin.pwd \
  -f add-sales.ldif
```

Rollback:

```ldif
dn: cn=SALES,cn=OracleContext,dc=example,dc=com
changetype: delete
```

### Modify descriptor

```ldif
dn: cn=SALES,cn=OracleContext,dc=example,dc=com
changetype: modify
replace: orclNetDescString
orclNetDescString: (DESCRIPTION=(ADDRESS=(PROTOCOL=TCP)(HOST=new-dbscan.example.com)(PORT=1521))(CONNECT_DATA=(SERVICE_NAME=sales.example.com)))
```

Rollback uses the same `replace` operation with the previous descriptor captured before the change.

### Delete

```ldif
dn: cn=SALES,cn=OracleContext,dc=example,dc=com
changetype: delete
```

Before delete, export `dn`, `objectClass`, `cn`, and `orclNetDescString`. Roll back with a clean `changetype: add` LDIF; exclude server-managed operational attributes.

### Change checklist

- [ ] Correct Oracle Context and writable endpoint.
- [ ] Replication healthy before change.
- [ ] Alias does not collide by case or DN normalization.
- [ ] Current entry exported for rollback.
- [ ] Descriptor reviewed: protocol, hosts, ports, `SERVICE_NAME`, RAC/HA, TCPS, security options.
- [ ] Delegated account and LDAPS used.
- [ ] Change applied once.
- [ ] Entry matches on at least two replicas.
- [ ] `tnsping` uses LDAP rather than local fallback.
- [ ] Real database connection succeeds.
- [ ] Rollback tested or ready.

## Oracle client configuration

`ldap.ora`:

```text
DIRECTORY_SERVERS = (oud-vip.example.com:1389:1636)
DEFAULT_ADMIN_CONTEXT = "dc=example,dc=com"
DIRECTORY_SERVER_TYPE = OID
```

`sqlnet.ora`:

```text
NAMES.DIRECTORY_PATH = (LDAP, TNSNAMES, EZCONNECT)
```

Client checks:

```bash
printf 'ORACLE_HOME=%s\nTNS_ADMIN=%s\nLDAP_ADMIN=%s\n' \
  "$ORACLE_HOME" "$TNS_ADMIN" "$LDAP_ADMIN"
tnsping SALES
sqlplus /@SALES
```

A local `tnsnames.ora` entry can mask a failed LDAP lookup. Test an LDAP-only name or inspect naming-method order.

## Replication

Minimal status:

```bash
"$OUD_BIN/dsreplication" status \
  --hostname oud01.example.com --port 4444 \
  --adminUID admin \
  --adminPasswordFile /secure/oud/oud-admin.pwd \
  --trustStorePath /secure/oud/client-truststore \
  --no-prompt
```

Compatibility view with more columns:

```bash
"$OUD_BIN/dsreplication" status \
  --hostname oud01.example.com --port 4444 \
  --adminUID admin \
  --adminPasswordFile /secure/oud/oud-admin.pwd \
  --trustStorePath /secure/oud/client-truststore \
  --dataToDisplay compat-view \
  --no-prompt
```

| Status/signal | Meaning/action |
| --- | --- |
| `Normal` | Directory replica is connected with the correct data set. |
| `Late` | Missing changes exceed the configured threshold; investigate sustained latency. |
| `Bad Data Set` | Data set is incompatible; stop writes/recovery improvisation and identify the authoritative source. |
| `Not Connected` | Directory server is not connected to a replication server. |
| `Up`/`Down` | Replication-server connectivity state. |
| `Unknown` | Node is unreachable/down or state cannot be determined. |
| M.C. | Missing changes not yet replayed. Persistent growth is a problem. |
| A.O.M.C. | Approximate age of oldest missing change. Persistent aging is a problem. |

Never run `dsreplication initialize` until the source and destination are explicitly reviewed. Initialization replaces destination data for the selected base DN.

## Configuration with dsconfig

Use interactive mode for learning and noninteractive mode for repeatable approved changes.

```bash
# Interactive
"$OUD_BIN/dsconfig" \
  -h oud01.example.com -p 4444 \
  -D "cn=Directory Manager" -j /secure/oud/oud-bind.pwd \
  --trustStorePath /secure/oud/client-truststore

# Discover subcommands
"$OUD_BIN/dsconfig" --help

# List back ends, read-only
"$OUD_BIN/dsconfig" list-backends \
  -h oud01.example.com -p 4444 \
  -D "cn=Directory Manager" -j /secure/oud/oud-bind.pwd \
  --trustStorePath /secure/oud/client-truststore \
  --no-prompt

# Inspect one back end, read-only
"$OUD_BIN/dsconfig" get-backend-prop \
  --backend-name userRoot \
  -h oud01.example.com -p 4444 \
  -D "cn=Directory Manager" -j /secure/oud/oud-bind.pwd \
  --trustStorePath /secure/oud/client-truststore \
  --no-prompt
```

Before a `set-*`, `create-*`, or `delete-*` command:

1. Run the corresponding `get-*` or `list-*` command.
2. Capture exact output and current `config/config.ldif` backup.
3. Check whether the property is dynamic or needs restart.
4. Generate a precise rollback command.
5. Apply to the correct scope; determine whether configuration is replicated or node-local.
6. Read back the effective value and test behavior.

## Backups, exports, and tasks

List back ends:

```bash
"$OUD_BIN/list-backends"
```

Back up all back ends:

```bash
"$OUD_BIN/backup" \
  --backUpAll --compress \
  --backupDirectory /backup/oud/asinst_1/2026-08-24
```

Back up one back end:

```bash
"$OUD_BIN/backup" \
  --backendID userRoot --compress \
  --backupDirectory /backup/oud/asinst_1/2026-08-24
```

Export a suffix to LDIF:

```bash
"$OUD_BIN/export-ldif" \
  --backendID userRoot \
  --ldifFile /backup/oud/exports/userRoot-2026-08-24.ldif
```

Command option names can differ across releases; confirm with `--help`. Online backup/export requires task connection options. Remote commands write data files on the server host, not on the operator workstation.

List/manage scheduled tasks:

```bash
"$OUD_BIN/manage-tasks" --help
```

Restore and offline import can replace data and require downtime. Use the full recovery runbook, not a cheatsheet command.

## Monitoring

Monitor root:

```bash
"$OUD_BIN/ldapsearch" \
  -h oud01.example.com -p 4444 --useSSL \
  --trustStorePath /secure/oud/client-truststore \
  -D "cn=Directory Manager" -j /secure/oud/oud-bind.pwd \
  -s base -b "cn=monitor" "(objectClass=*)" \
  currentTime startTime upTime currentConnections totalConnections \
  maxConnections vendorVersion
```

Client connections:

```bash
"$OUD_BIN/ldapsearch" \
  -h oud01.example.com -p 4444 --useSSL \
  --trustStorePath /secure/oud/client-truststore \
  -D "cn=Directory Manager" -j /secure/oud/oud-bind.pwd \
  -s base -b "cn=Client Connections,cn=monitor" \
  "(objectClass=*)"
```

Back-end monitor example:

```bash
"$OUD_BIN/ldapsearch" \
  -h oud01.example.com -p 4444 --useSSL \
  --trustStorePath /secure/oud/client-truststore \
  -D "cn=Directory Manager" -j /secure/oud/oud-bind.pwd \
  -s base -b "cn=userRoot Backend,cn=monitor" \
  "(objectClass=*)"
```

Monitor branches are configuration-dependent. Discover DNs first with the `1.1` attribute search.

## Patching checklist

### Before

- [ ] Exact patch README and support notes reviewed.
- [ ] Base release, JDK, OS, OPatch version, free space, and conflicts checked.
- [ ] Every instance/domain using the Oracle home identified.
- [ ] Replication healthy; failover and drain tested.
- [ ] Data, instance configuration, and cold/full backups taken as required.
- [ ] Rollback and abort criteria approved.

```bash
"$ORACLE_HOME/OPatch/opatch" version
"$ORACLE_HOME/OPatch/opatch" lsinventory
```

### During

- [ ] Endpoint drained/removed from service.
- [ ] All processes required by the README stopped.
- [ ] Patch staged outside `ORACLE_HOME` and verified.
- [ ] Exact README apply command used.

Common pattern only when the README confirms it:

```bash
cd /stage/PATCH_ID
"$ORACLE_HOME/OPatch/opatch" apply
```

### After

```bash
"$ORACLE_HOME/OPatch/opatch" lsinventory
"$ORACLE_HOME/OPatch/opatch" lspatches
"$OUD_BIN/start-ds"
"$OUD_BIN/status"
```

- [ ] Startup log clean.
- [ ] LDAP/LDAPS and TLS valid.
- [ ] Suffix search/write test passed as approved.
- [ ] Replication converged.
- [ ] TNS lookup and database connection passed.
- [ ] Endpoint observed, then returned to service.

## Upgrade boundary

Do not treat a major upgrade as OPatch work. The documented 12.2.1.4 to 14.1.2.1 path includes new target software, JDK 17, instance preparation, and:

```bash
./upgrade-oud-instances --instancePath /path/to/12c-instance
/path/to/instance/OUD/bin/start-ds --upgrade
```

These commands are not generic. Confirm the source/target release and full Oracle upgrade guide. Oracle documents the upgrade as non-reversible; recovery requires restoring the complete pre-upgrade environment.

## Logs and evidence

Common instance location:

```bash
ls -ltr "$OUD_INSTANCE/OUD/logs"
tail -200 "$OUD_INSTANCE/OUD/logs/errors" 2>/dev/null
```

Depending on configuration, collect:

- errors log and alert messages;
- access logs for operation, result code, bind DN, client, and processing time;
- audit log for data/configuration changes;
- debug log only when deliberately enabled;
- replication repair log for conflict handling;
- upgrade and pre-upgrade logs;
- OPatch logs and `lsinventory`;
- OUDSM/WebLogic logs for UI-only problems;
- OS journal/service-manager logs.

Correlate timestamps in one timezone. Redact passwords, tokens, and sensitive directory attributes before attaching evidence.

## Safety reminders

- Prefer read-only inspection before change.
- Use LDAPS/StartTLS, a trust store, password files, and delegated accounts.
- Do not use `--trustAll` as a normal production option.
- Do not edit `config.ldif` online.
- Do not run restore, import, initialize, rebuild-index, or upgrade as an exploratory fix.
- Write a replicated TNS alias once; verify it on other replicas.
- Back up the current alias before modify/delete.
- A restart is not diagnosis.
- Replication is not a backup.
- A patch README overrides generic examples.
