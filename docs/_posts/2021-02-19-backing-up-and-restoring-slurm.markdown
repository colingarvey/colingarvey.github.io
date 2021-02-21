---
layout: single
title: "Slurm Accounting Database Backup and Restore"
date: 2021-02-20 03:05:05 -0800
categories:
    - tech
---
Slurm is a workflow and resource manager that runs on High Performance Computing Clusters (read Supercomputers.) This article is a brain dump of 
my experience performing changes to the associations table in its database. The associations table manages relationships between users and "bank accounts".
Bank accounts are a way to charge for cluster resource utilization, primarily cores, but including other finite resources.


Sacctmgr is the CLI tool used by Slurm to manage its accounting database. Prior to making changes using sacctmgr on a large scale, it is always beneficial to create a backup.

## Creating Backups
### sacctmgr approach
This backs up the existing sacctmgr associations data to a config file that includes all accounting information. NOTE: restoring from this backup does not delete previous associations.
This is a Slurm bug. Therefore, I would highly recommend performing both this for convenience and a MySQL database dump as well.

This is relatively lightweight on the database and can be performed on any node.

```bash
#!/usr/bin/env bash
# Must be run as root or slurmadm

CLUSTER=slurm_cluster # Rosalind cluster name
OUTPUT_FILE=$CLUSTER.cfg
sacctmgr dump slurm_cluster file=slurm_cluster.cfg
```

### slurmdbd approach
This backs up the entire slurm_acct_db database from your head node.

**CAUTION** I am unsure whether this has an impact to a running Slurm database or its existing jobs. Dumps can cause downtime while MySQL or MariaDB is under high load in my experience.

1. ssh to your head node
2. Create a bash script or run the following commands to back up the file to a .sql file
```bash
#!/usr/bin/env bash
CURRENT_DATE=$(date +"%Y%m%d_%H%M00")
DATABASE=slurm_acct_db
DB_USER=slurm
DUMP_OUTPUT_FILE=/tmp/${DATABASE}_${CURRENT_DATE}.sql
mysqldump -u $DB_USER -p --databases $DATABASE > $DUMP_OUTPUT_FILE
```

## Restoring from Backups
### sacctmgr approach
This restores the existing sacctmgr associations data from a config file that includes all accounting information, that was created via the `sacctmgr load` command above

This is relatively lightweight on the database and can be performed on any node.

```bash
#!/usr/bin/env bash
# Must be run as root or slurmadm

CLUSTER=slurm_cluster # Rosalind cluster name
OUTPUT_FILE=$CLUSTER.cfg
sacctmgr load slurm_cluster file=slurm_cluster.cfg

```

### slurmdbd approach
This restores the entire slurm_acct_db database from a SQL Dump created in the previous step.

**CAUTION** I am unsure whether this **restore** has an impact to a running Slurm database or its existing jobs. Large SQL operations can cause downtime while MySQL or MariaDB is under high load in my experience.
1. Stop slurmdbd
2. Connect to MySQL as root
`mysql -u root -p`
4. Perform the following to restore database
```sql
DROP DATABASE slurm_acct_db
CREATE slurm_acct_db
EXIT
```
```bash
## STOP SLURMDBD first
DUMP_INPUT_FILE=""
mysql -u root -p slurm_acct_db < ${DUMP_INPUT_FILE}
```
5. Start slurmdbd

