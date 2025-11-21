# Oracle Data Guard Fast-Start Failover (FSFO) Cheat Sheet

## 1. FSFO Overview

- **Purpose:** Automatic failover to a standby database when the primary becomes unavailable, ensuring minimal downtime.
- **Components:**
  - **Data Guard Broker:** Automates and manages FSFO.
  - **Observer:** External process that monitors the primary and standby databases asynchronously.

---

## 2. Pre-requisites

- Data Guard Broker must be configured and enabled on both primary and standby.
- Both databases in DG configuration must be in appropriate **protection mode**:
  - Maximum Availability
  - Maximum Performance
  - **Not** Maximum Protection
- Observer process must be running (typically on a different host).
- Standby database should be in **SYNC** mode for full protection (recommended).

---

## 3. Setup Commands

**On both databases:**
```sh
DGMGRL> CREATE CONFIGURATION ...;
DGMGRL> ADD DATABASE ... AS CONNECT IDENTIFIER IS ...;
DGMGRL> ENABLE CONFIGURATION;
