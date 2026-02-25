
---

## 🚀 Oracle Fast-Start Failover (FSFO) Cheatsheet

### 1. Functional Overview

* **The Observer:** A lightweight DGMGRL client process that pings the primary and standby. It acts as the "quorum" to prevent split-brain scenarios.
* **Automatic Reinstatement:** Once the failed primary comes back online, FSFO automatically reinstates it as a new standby (requires Flashback Database).
* **Zero Data Loss:** Typically used with `MaxAvailability` (SYNC) to guarantee no data loss, but also supports `MaxPerformance` (ASYNC) with a specific lag limit.

---

### 2. Prerequisites

Before enabling FSFO, ensure the following are met:

* **Broker Configured:** Both databases must be in a Data Guard Broker configuration.
* **Flashback Database:** Enabled on **both** Primary and Standby (`ALTER DATABASE FLASHBACK ON;`).
* **Standby Redo Logs (SRLs):** Configured on both sides.
* **Static Service:** A static listener entry is required so the Observer can restart a failed instance.

---

### 3. Configuration Steps (DGMGRL)

Execute these commands within the `DGMGRL` utility:

| Step | Command | Description |
| --- | --- | --- |
| **1. Set Target** | `EDIT DATABASE 'PrimaryDB' SET PROPERTY FastStartFailoverTarget = 'StandbyDB';` | Defines who the failover partner is. |
| **2. Set Threshold** | `EDIT CONFIGURATION SET PROPERTY FastStartFailoverThreshold = 30;` | Seconds to wait before failing over (Default: 30s). |
| **3. Set Protection** | `EDIT CONFIGURATION SET PROTECTION MODE AS MAXAVAILABILITY;` | Recommended for zero data loss. |
| **4. Enable FSFO** | `ENABLE FAST_START FAILOVER;` | Turns on the automated logic. |
| **5. Start Observer** | `START OBSERVER;` | Run this on a separate host (third site). |

> **Note:** For ASYNC configurations, set `FastStartFailoverLagLimit` (in seconds) to define acceptable data loss.

---

### 4. Monitoring & Verification

Use these commands to ensure your "auto-pilot" is ready:

* **Check FSFO Status:**
```sql
DGMGRL> SHOW FAST_START FAILOVER;

```


*Look for:* `Status: ENABLED` and `Observer: <hostname> is connected`.
* **View Configuration Details:**
```sql
DGMGRL> SHOW CONFIGURATION VERBOSE;

```


* **Database Level Check (SQL):**
```sql
SELECT FS_FAILOVER_STATUS, FS_FAILOVER_OBSERVER_PRESENT, FS_FAILOVER_OBSERVER_HOST 
FROM V$DATABASE;

```



---

### 5. Troubleshooting Common Issues

* **"FSFO Target is Unsynchronized":** Check network latency or redo transport errors. FSFO will not trigger if the standby isn't in sync.
* **Observer Not Running:** Ensure the observer process is active on a host that is *not* the primary or standby server.
* **Reinstate Failed:** If auto-reinstatement fails, manually try:
`DGMGRL> REINSTATE DATABASE 'OldPrimaryName';`

---

### 6. Helpful Maintenance Commands

* **Stop Observer:** `STOP OBSERVER;`
* **Disable FSFO:** `DISABLE FAST_START FAILOVER;` (Do this before planned maintenance to avoid accidental failovers).
* **Switchover:** `SWITCHOVER TO 'StandbyDB';` (Safe, manual role reversal).



To configure the Oracle Data Guard Observer, you need to treat it as a separate entity—ideally installed on a **third host** to avoid a single point of failure.

Here is the markdown cheatsheet for configuring, running, and managing the Observer.

---

## 🛰️ Configuring the Oracle Data Guard Observer

### 1. Networking (The Foundation)

The Observer uses the Oracle Net aliases defined in `tnsnames.ora` to connect to both the Primary and Standby. Ensure the `tnsnames.ora` on the **Observer Host** contains entries for both databases.

```text
# Example tnsnames.ora on Observer Host
PRIMARY_DB = (DESCRIPTION=(ADDRESS=(PROTOCOL=TCP)(HOST=host1)(PORT=1521))(CONNECT_DATA=(SERVICE_NAME=db_pr)))
STANDBY_DB = (DESCRIPTION=(ADDRESS=(PROTOCOL=TCP)(HOST=host2)(PORT=1521))(CONNECT_DATA=(SERVICE_NAME=db_st)))

```

---

### 2. Wallet/Credentials (Best Practice)

To allow the Observer to run automatically without prompting for a password (and to keep it secure), use an **Oracle Password Wallet**.

1. **Create the Wallet:**
```bash
mkstore -wwallet ./wallet -create

```


2. **Add Credentials:**
```bash
mkstore -wwallet ./wallet -createCredential PRIMARY_DB sys password123
mkstore -wwallet ./wallet -createCredential STANDBY_DB sys password123

```


3. **Update sqlnet.ora:** Ensure `sqlnet.ora` on the Observer host points to this wallet location.

---

### 3. Starting the Observer

The Observer is started via the `DGMGRL` utility. You have three main ways to run it:

#### Option A: Interactive Mode (For Testing)

```bash
dgmgrl sys/password@PRIMARY_DB
DGMGRL> START OBSERVER;

```

#### Option B: Background Mode (Standard Production)

Use the `-logfile` and `&` to run it as a background process.

```bash
dgmgrl -silent sys/password@PRIMARY_DB "START OBSERVER FILE='fsfo_obs.dat'" logfile observer.log &

```

#### Option C: Using a Configuration File (Recommended)

Since Oracle 12c/19c, you can use an observer configuration file to manage multiple observers.

```bash
DGMGRL> START OBSERVER CONFIGFILE='/u01/app/oracle/network/admin/observer.ini';

```

---

### 4. Observer Best Practices

| Feature | Recommendation |
| --- | --- |
| **Location** | Always place on a different network segment/host than the DBs. |
| **Version** | The Observer software version must match or be higher than the DB version. |
| **Persistence** | Use `nohup` or a system service (systemd) to ensure it restarts on reboot. |
| **Redundancy** | Since 12.1, you can start **Multiple Observers**. Only one is "Master"; the others are "Backup". |

---

### 5. Managing Multiple Observers (12c/19c/21c)

If you want high availability for the Observer itself:

1. **Start a second observer** on a different host using the same DGMGRL commands.
2. **Check their status:**
```sql
DGMGRL> SHOW CONFIGURATION VERBOSE;

```


*The output will list the "Master Observer" and any "Backup Observers".*

---

### 6. Common Observer Commands

| Action | Command |
| --- | --- |
| **Stop Observer** | `STOP OBSERVER;` (From any DGMGRL session) |
| **Check Connectivity** | `SHOW OBSERVER;` |
| **Override Host** | `SET MASTER OBSERVER TO 'hostname';` |

---
