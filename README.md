# PostgreSQL Backup & Recovery — Complete Guide

## Overview

Complete backup and recovery practice for PostgreSQL
support engineers. Covers all three backup methods
with both manual steps and automated CRON scripts.

## Repository Structure
```
postgresql-backup-recovery/
├── README.md                      ← this file
├── backup.sh                      ← simple backup script
│
├── 01-pgdump/
│   ├── README.md                  ← pg_dump explanation
│   ├── commands.md                ← manual commands
│   └── cron-backup.sh             ← automated backup script
│
├── 02-pgbasebackup/
│   ├── README.md                  ← pg_basebackup explanation
│   ├── commands.md                ← manual commands
│   └── cron-backup.sh             ← automated backup script
│
└── 03-pitr/
    ├── README.md                  ← PITR explanation
    ├── manual-restore.md          ← manual restore steps
    ├── setup-archiving.sh         ← WAL archiving setup
    └── pitr-restore.sh            ← automated PITR restore
```

## Three Backup Methods Covered

| Method | Tool | Best For | Data Loss |
|---|---|---|---|
| Logical | `pg_dump` | Single table/database | Hours |
| Physical | `pg_basebackup` | Full server, large DBs | Hours |
| Point in Time | PITR + WAL | Exact minute recovery | Near Zero |

## Each folder has TWO approaches
```
Manual steps   → understand HOW it works
CRON scripts   → understand HOW it runs in production
```

## The Golden Rules

### Why we need BOTH HA and Backup
```
HA Replicas = protection against INFRASTRUCTURE failure
              Server crashes → failover to replica

Backup      = protection against DATA failure
              Data deleted/corrupted → restore from backup

Replica copies EVERYTHING from primary including mistakes!
Only backup can go back in time before the mistake.
```

### The 3-2-1 Backup Rule
```
3 copies  → primary + replica + backup
2 types   → server storage + cloud (S3)
1 offsite → backup in different region

Even if entire data center burns down
→ S3 backup is safe in different region ✅
```

### What if ALL servers crash?
```
Primary  → CRASHED ❌
Replica1 → CRASHED ❌
Replica2 → CRASHED ❌
S3 Backup → SAFE ✅ (different location!)

Restore to new server from S3 → back online in 1 hour
```

## Quick Command Reference

### pg_dump (logical backup)
```bash
# Backup single database
pg_dump -U admin -d mydb -F c -f backup.dump

# Backup all databases + users
pg_dumpall -U admin -f all.sql

# Restore full database
pg_restore -U admin -d mydb -v backup.dump

# Restore single table only
pg_restore -U admin -d mydb \
  -t tablename --clean --if-exists -v backup.dump
```

### pg_basebackup (physical backup)
```bash
# Take physical backup
pg_basebackup -h localhost -p 5432 \
  -U replicator -D /tmp/backup \
  -F t -z -P -v --wal-method=stream

# Restore - extract to data volume
tar -xzf base.tar.gz -C /var/lib/postgresql/data/

# Fix permissions after restore
chown -R postgres:postgres /data
chmod 700 /data
```

### PITR (point in time recovery)
```bash
# Enable WAL archiving
psql -c "ALTER SYSTEM SET archive_mode = 'on';"
psql -c "ALTER SYSTEM SET archive_command = 'cp %p /archive/%f';"
psql -c "ALTER SYSTEM SET archive_timeout = '60';"

# Restore to specific time
./03-pitr/pitr-restore.sh "2026-03-30 02:41:58"
```

## CRON Setup (Production Automation)
```bash
# Open cron editor
crontab -e

# Add these lines:
# pg_dump every night at 2 AM
0 2 * * * /path/to/01-pgdump/cron-backup.sh

# pg_basebackup every Sunday midnight
0 0 * * 0 /path/to/02-pgbasebackup/cron-backup.sh

# WAL archiving runs continuously (setup once)
# archive_timeout = 60 (archives every 60 seconds)
```

## What We Proved in Practice

### pg_dump
```
Created employees + departments tables
Took pg_dump backup
Dropped all tables (disaster!)
Restored from backup → data back ✅
Restored single table only → works ✅
```

### pg_basebackup
```
Took physical backup of pg-node3
Stopped pg-node3 (simulated crash)
Wiped all data (simulated disk failure)
Restored from physical backup
pg-node3 rejoined cluster with Lag=0 ✅
```

### PITR
```
Enabled WAL archiving on pg-node1
Took base backup as starting point
Recorded safe time: 2026-03-30 02:41:58
Inserted test data (PITR_Test1,2,3)
Ran DELETE FROM employees (disaster!)
PITR restored to 02:41:58
All 14 employees restored ✅
DELETE completely undone ✅
Zero data loss ✅
```

## Backup vs HA Comparison

| Scenario | HA Replica | Backup |
|---|---|---|
| Server hardware crash | ✅ | ✅ |
| All servers crash | ❌ | ✅ |
| Accidental DELETE | ❌ | ✅ |
| DROP TABLE mistake | ❌ | ✅ |
| Ransomware attack | ❌ | ✅ |
| Restore to yesterday | ❌ | ✅ |
| Zero downtime maintenance | ✅ | ❌ |

## Interview Questions

**Q: What is your backup strategy?**
Three layers: nightly pg_dump for logical backup,
weekly pg_basebackup for physical backup, continuous
WAL archiving for PITR. All automated via CRON.
Backups stored offsite on S3 in different region.
Follow 3-2-1 rule.

**Q: Why need backup if you have HA replicas?**
Replicas mirror everything including mistakes.
Accidental DELETE instantly replicated to all replicas.
Only backup from before mistake can restore data.
HA protects infrastructure, backup protects data.

**Q: What is PITR?**
Point in Time Recovery. Restores database to any
exact moment using WAL files. Requires WAL archiving
and base backup. Can restore to 1 second before
accidental DELETE with near zero data loss.

**Q: What is the 3-2-1 backup rule?**
3 copies, 2 storage types, 1 offsite location.
Primary + replica + S3 backup in different region.

## Related Repositories

- Manual HA: https://github.com/mahi-cloud99/postgresql-ha-manual
- Automatic HA: https://github.com/mahi-cloud99/postgresql-ha-patroni
- This repo: https://github.com/mahi-cloud99/postgresql-backup-recovery
