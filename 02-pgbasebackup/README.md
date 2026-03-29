# pg_basebackup — Physical Backup & Restore

## What is pg_basebackup?

pg_basebackup creates a binary copy of the entire 
PostgreSQL data directory. Like taking a photo of 
the entire database server.
```
pg_dump     = photocopying individual pages of a book
pg_basebackup = scanning the entire book as one image
```

## Why we need BOTH HA Replicas AND Backups

### HA Replicas protect against INFRASTRUCTURE failure
```
Server hardware dies → replica takes over
Network failure      → another node handles traffic
PostgreSQL crash     → Patroni promotes replica
```

### Backups protect against DATA failure
```
Developer runs: DELETE FROM orders; (no WHERE clause!)
  Primary:  0 orders ❌
  Replica1: 0 orders ❌ (copied the delete!)
  Replica2: 0 orders ❌ (copied the delete!)

Replica is a MIRROR — copies everything including mistakes!
Only backup from BEFORE mistake can restore data.
```

### What if ALL servers crash simultaneously?
```
Real scenarios:
  - Data center fire
  - Power outage
  - Natural disaster
  - Cloud region outage

Without offsite backup → data GONE FOREVER ❌
With S3 backup        → restore in 1 hour ✅
```

## The 3-2-1 Backup Rule
```
3 → Keep 3 copies of data
2 → Store on 2 different storage types
1 → Keep 1 copy offsite (different location)

Example:
  Copy 1: Primary (Mumbai server)
  Copy 2: Replica (Chennai server)
  Copy 3: Backup on AWS S3 (different region)
```

## Protection Matrix

| Scenario | HA Only | Backup Only | HA + Backup |
|---|---|---|---|
| One server crashes | ✅ | ✅ | ✅ |
| All servers crash | ❌ | ✅ | ✅ |
| Accidental DELETE | ❌ | ✅ | ✅ |
| DROP TABLE mistake | ❌ | ✅ | ✅ |
| Ransomware | ❌ | ✅ | ✅ |
| Data center fire | ❌ | ✅* | ✅* |
| Restore to yesterday | ❌ | ✅ | ✅ |

*Only if backup stored offsite

## When to use pg_basebackup
```
✅ Entire server disaster recovery
✅ Setting up new replicas
✅ Large databases over 50GB
✅ Need Point in Time Recovery (PITR)
✅ Faster restore than pg_dump
```

## Speed Comparison (500GB database)
```
pg_dump restore:       4-6 hours ❌
pg_basebackup restore: 30-45 min ✅

Why faster?
pg_dump:        re-executes every INSERT, rebuilds indexes
pg_basebackup:  copies binary files directly
```

## Backup Command
```bash
pg_basebackup \
  -h localhost \     # host to connect
  -p 5432 \          # port number
  -U replicator \    # MUST use replication user
  -D /tmp/backup \   # destination directory
  -F t \             # tar format
  -z \               # compress with gzip
  -P \               # show progress
  -v                 # verbose output
```

## Why replicator user?
```
pg_basebackup uses replication protocol internally.
Only users with REPLICATION privilege can use it.

Without REPLICATION privilege:
  FATAL: must be superuser or replication role ❌

Create with:
  CREATE USER replicator WITH REPLICATION PASSWORD '...';
```

## Backup Output Files

| File | Contents | Required? |
|---|---|---|
| `base.tar.gz` | Entire data directory | Yes ✅ |
| `pg_wal.tar.gz` | WAL during backup | Yes ✅ |
| `backup_manifest` | File checksums | Optional |

## Why both files needed?
```
Backup takes 30 minutes for large database.
Database keeps running during backup.
New transactions happen continuously.

base.tar.gz alone    = INCONSISTENT ❌
base.tar.gz + pg_wal = CONSISTENT ✅

pg_wal captures all changes during backup.
Restore: extract base → apply WAL → consistent!
```

## Complete Restore Steps

### Step 1 - Stop the node
```bash
docker stop pg-node3
```

### Step 2 - Wipe existing data
```bash
docker run --rm \
  -v pg-patroni-cluster_pg_node3_data:/data \
  alpine \
  sh -c "find /data -mindepth 1 -delete"
```

### Step 3 - Restore backup
```bash
docker run --rm \
  -v pg-patroni-cluster_pg_node3_data:/data \
  -v ~/Desktop/pg-backups/physical:/backup \
  postgres:15 \
  bash -c "cd /data && tar -xzf /backup/base.tar.gz"
```

### Step 4 - Fix permissions (CRITICAL!)
```bash
docker run --rm \
  -v pg-patroni-cluster_pg_node3_data:/data \
  postgres:15 \
  bash -c "chown -R postgres:postgres /data && chmod 700 /data"
```

Why fix permissions?
```
tar extracts files as root
PostgreSQL REFUSES to start if owned by root
Requires: owner=postgres, permission=700

700 = owner can read+write+execute
      group: no access
      others: no access
```

### Step 5 - Start and verify
```bash
docker start pg-node3
sleep 20
patronictl -c /etc/patroni.yml list
```

## Production Backup Schedule (CRON)
```
Never backup manually in production!
CRON automates everything.

0 2 * * *   → pg_dumpall nightly at 2 AM
0 0 * * 0   → pg_basebackup weekly Sunday
*/5 * * * * → WAL archiving every 5 min (PITR)
```

## Interview Questions

**Q: Why need backups if we have HA replicas?**
Replicas mirror everything including mistakes. Accidental
DELETE replicated to all replicas instantly. Only backup
from before mistake can restore data. HA protects
infrastructure, backup protects data. Need both.

**Q: What if all servers crash simultaneously?**
Store backups offsite on S3 in different region.
Even if entire data center burns, S3 backup is safe.
Restore to new server from S3 in 1-2 hours.

**Q: Why faster than pg_dump?**
pg_dump re-executes every INSERT and rebuilds all indexes.
pg_basebackup copies binary files directly. For 500GB:
pg_dump=4hrs vs pg_basebackup=30min.

**Q: What is the 3-2-1 backup rule?**
3 copies, 2 storage types, 1 offsite. Primary + replica
+ S3 backup in different region.
