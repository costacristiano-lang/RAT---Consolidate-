# Oracle ZDM Migration Runbook

## Overview

This document provides a GitHub-ready step-by-step runbook for starting an Oracle migration project using Zero Downtime Migration (ZDM).

The structure is divided into:

- ZDM host provisioning
- Source database preparation
- Target database preparation
- Validation steps
- ZDM execution
- Post-migration checks

This runbook assumes the most common initial scenario:

- Migration method: `ONLINE_PHYSICAL`
- Data movement: `RMAN + Data Guard`
- Source and target with network connectivity
- Target already provisioned as a placeholder database

---

## 1. Provision the ZDM Host

### 1.1 Host requirements

Prepare a dedicated Linux host for ZDM with:

- Oracle Linux or RHEL supported by Oracle for ZDM
- Around `100 GB` of free disk space
- Network connectivity to both source and target environments
- No Grid Infrastructure running on the ZDM host

### 1.2 Create the ZDM user

```bash
sudo groupadd zdm
sudo useradd -g zdm zdmuser

sudo mkdir -p /u01/app/zdmhome
sudo mkdir -p /u01/app/zdmbase
sudo chown -R zdmuser:zdm /u01/app/zdmhome /u01/app/zdmbase
```

### 1.3 Install required packages

```bash
sudo yum install -y glibc-devel expect unzip libaio
```

If your OS image is minimal, you may also need packages such as `libnsl` and `ncurses-compat-libs`.

### 1.4 Validate hostname resolution

```bash
hostname
ping -c 2 <zdm-hostname>
getent hosts <zdm-hostname>
```

### 1.5 Install ZDM software

```bash
su - zdmuser

export ZDM_HOME=/u01/app/zdmhome
export ZDM_BASE=/u01/app/zdmbase

cd <directory-where-zip-was-downloaded>
unzip <zdm-kit>.zip

./zdminstall.sh setup oraclehome=$ZDM_HOME oraclebase=$ZDM_BASE ziploc=<directory-of-zip> -zdm
```

### 1.6 Start and validate the ZDM service

```bash
$ZDM_HOME/bin/zdmservice start
$ZDM_HOME/bin/zdmservice status
$ZDM_HOME/bin/zdmcli -build
```

### 1.7 Configure SSH access

The ZDM host must connect to source and target hosts through SSH.

```bash
ssh-keygen -t rsa
ssh-copy-id oracle@<source-host>
ssh-copy-id oracle@<target-host>

ssh oracle@<source-host> "hostname"
ssh oracle@<target-host> "hostname"
```

---

## 2. Prepare the Source Database

### 2.1 Validate source database configuration

```sql
sqlplus / as sysdba

SELECT * FROM v$version;
ARCHIVE LOG LIST;
SHOW PARAMETER spfile;
SHOW PARAMETER db_name;
SHOW PARAMETER db_unique_name;
SHOW PARAMETER compatible;
```

### 2.2 Enable ARCHIVELOG if needed

```sql
sqlplus / as sysdba

SHUTDOWN IMMEDIATE;
STARTUP MOUNT;
ALTER DATABASE ARCHIVELOG;
ALTER DATABASE OPEN;
ARCHIVE LOG LIST;
```

### 2.3 Validate TDE wallet

```sql
sqlplus / as sysdba

SELECT * FROM v$encryption_wallet;
```

For CDB environments:

```sql
sqlplus / as sysdba

SHOW CON_NAME;
SELECT con_id, wrl_type, status FROM v$encryption_wallet;
```

### 2.4 Validate RMAN snapshot controlfile in RAC

If the source is RAC, validate that the snapshot controlfile is on shared storage.

```bash
rman target /
CONFIGURE SNAPSHOT CONTROLFILE NAME TO '+DATA/<db_name>/snapcf_<db_name>.f';
SHOW SNAPSHOT CONTROLFILE NAME;
```

### 2.5 Validate network connectivity from source

```bash
tnsping <target_scan_service>
ssh oracle@<target-host> "hostname"
```

### 2.6 Operational recommendations on source

- Disable scheduled RMAN backup jobs during the migration window
- Validate database time synchronization
- Confirm that source and target use compatible SQL*Net settings
- Confirm that the source database uses `SPFILE`

---

## 3. Prepare the Target Database

### 3.1 Provision the placeholder database

Provision the target database before starting ZDM.  
This database will be used as the placeholder/template target.

### 3.2 Validate target parameters

```sql
sqlplus / as sysdba

SHOW PARAMETER db_name;
SHOW PARAMETER db_unique_name;
SHOW PARAMETER compatible;
SELECT * FROM v$version;
```

Checklist:

- `db_name` must match the source database
- `db_unique_name` must be different from the source
- Target patch level must be equal to or greater than the source

### 3.3 Validate TDE wallet on target

```sql
sqlplus / as sysdba

SELECT * FROM v$encryption_wallet;
```

### 3.4 Validate recovery area and storage

```sql
sqlplus / as sysdba

SHOW PARAMETER db_recovery_file_dest;
SHOW PARAMETER db_recovery_file_dest_size;
```

### 3.5 Validate `db_unique_name` for the response file

```sql
sqlplus / as sysdba

SHOW PARAMETER db_unique_name;
```

---

## 4. Create the ZDM Response File

Use the ZDM response file template and update it with your environment values.

Example:

```ini
TGT_DB_UNIQUE_NAME=<target_db_unique_name>
PLATFORM_TYPE=EXACC
MIGRATION_METHOD=ONLINE_PHYSICAL
```

Optional storage settings:

```ini
TGT_DATADG=+DATA
TGT_REDODG=+RECO
TGT_RECODG=+RECO
```

If needed:

```ini
TGT_SKIP_DATAPATCH=FALSE
```

---

## 5. Pre-Migration Validation

### 5.1 Validate the source database

```sql
sqlplus / as sysdba

SELECT name, open_mode, database_role, log_mode FROM v$database;
SHOW PARAMETER db_name;
SHOW PARAMETER db_unique_name;
SHOW PARAMETER spfile;
SELECT * FROM v$encryption_wallet;
```

### 5.2 Validate the target database

```sql
sqlplus / as sysdba

SELECT name, open_mode, database_role, log_mode FROM v$database;
SHOW PARAMETER db_name;
SHOW PARAMETER db_unique_name;
SHOW PARAMETER compatible;
SELECT * FROM v$encryption_wallet;
```

### 5.3 Validate host access and connectivity

```bash
ssh oracle@<source-host> "hostname"
ssh oracle@<target-host> "hostname"
tnsping <source_service>
tnsping <target_service>
```

---

## 6. Execute ZDM

### 6.1 Run evaluation mode

Run the ZDM evaluation first and fix all reported issues before executing the migration.

```bash
$ZDM_HOME/bin/zdmcli migrate database \
  -sourcedb <source_db_name> \
  -sourcenode <source_host> \
  -srcauth zdmauth \
  -srcarg1 user:oracle \
  -targetnode <target_host> \
  -targethome <target_oracle_home> \
  -targetbase <target_oracle_base> \
  -rsp /u01/app/zdmhome/rhp/zdm/template/my_zdm.rsp \
  -eval
```

### 6.2 Execute the migration

```bash
$ZDM_HOME/bin/zdmcli migrate database \
  -sourcedb <source_db_name> \
  -sourcenode <source_host> \
  -srcauth zdmauth \
  -srcarg1 user:oracle \
  -targetnode <target_host> \
  -targethome <target_oracle_home> \
  -targetbase <target_oracle_base> \
  -rsp /u01/app/zdmhome/rhp/zdm/template/my_zdm.rsp
```

### 6.3 Monitor the migration job

```bash
$ZDM_HOME/bin/zdmcli query job -id <job_id>
$ZDM_HOME/bin/zdmcli query job -id <job_id> -detail
```

---

## 7. Post-Migration Validation

### 7.1 Validate the target role and open mode

```sql
sqlplus / as sysdba

SELECT name, open_mode, database_role FROM v$database;
```

### 7.2 Validate database and instance information

```sql
sqlplus / as sysdba

SELECT instance_name, host_name FROM v$instance;
SELECT * FROM v$version;
```

### 7.3 Run datapatch if applicable

If the target patch level is higher than the source and datapatch was not executed automatically:

```bash
$ORACLE_HOME/OPatch/datapatch -verbose
```

---

## 8. Quick Checklist

### ZDM Host

- Dedicated host provisioned
- ZDM installed and service running
- SSH access to source and target validated
- Hostname resolution confirmed

### Source

- `ARCHIVELOG` enabled
- TDE wallet validated
- `SPFILE` confirmed
- Network connectivity to target validated
- RMAN jobs paused during migration window

### Target

- Placeholder database provisioned
- `db_name` matches source
- `db_unique_name` differs from source
- TDE wallet validated
- Recovery area and storage validated

---

## 9. Notes

- This runbook assumes `ONLINE_PHYSICAL` migration.
- For logical, offline, or Autonomous Database migrations, the flow changes.
- Always run `-eval` before the real migration.
- Adjust all placeholders before execution.
