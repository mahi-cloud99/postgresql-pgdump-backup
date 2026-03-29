# PostgreSQL Backup & Restore — Complete Guide

## Overview

Complete backup and restore practice for PostgreSQL
support engineers. Covers all three backup methods.

## Repository Structure
```
pg-backup-practice/
├── README.md                  ← this file
├── backup.sh                  ← automated backup script
├── 01-pgdump/
│   ├── README.md              ← pg_dump full explanation
│   └── commands.md            ← pg_dump commands reference
├── 02-pgbasebackup/
│   ├── README.md              ← pg_basebackup explanation
│   └── commands.md            ← pg_basebackup commands
└── 03-pitr/
    └── README.md              ← PITR (coming soon)
```

## Three Backup Methods

| Method | Tool | Best For | Restore Speed |
|---|---|---|---|
| Logical | pg_dump | Single table/database | Slow |
| Physical | pg_basebackup | Full server, large DBs | Fast |
| Point in Time | PITR + WAL | Exact minute recovery | Medium |

## The Golden Rule
```
HA Replicas = protection against INFRASTRUCTURE failure
Backup      = protection against DATA failure

You need BOTH!

Replica copies everything including mistakes.
Only backup can go back in time before the mistake.
```

## The 3-2-1 Rule
```
3 copies  → primary + replica + backup
2 types   → server storage + cloud (S3)
1 offsite → backup in different region
```

## Quick Commands
```bash
# Logical backup
pg_dump -U admin -d mydb -F c -f backup.dump
pg_restore -U admin -d mydb -v backup.dump

# Physical backup
pg_basebackup -h localhost -p 5432 \
  -U replicator -D /tmp/backup -F t -z -P -v

# Restore single table
pg_restore -U admin -d mydb \
  -t tablename --clean --if-exists -v backup.dump
```

## Practice Environment

Uses PostgreSQL HA Patroni cluster:
https://github.com/mahi-cloud99/postgresql-ha-patroni

## Related Repositories

- Manual HA: https://github.com/mahi-cloud99/postgresql-ha-manual
- Automatic HA: https://github.com/mahi-cloud99/postgresql-ha-patroni
