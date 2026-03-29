# pg_dump — Logical Backup & Restore

## What is pg_dump?

pg_dump connects to PostgreSQL and exports data as SQL 
statements into a file. Like a smart copy machine that 
reads all tables, indexes, constraints and writes them 
to one file.
```
pg_dump reads:
- Every table structure (CREATE TABLE)
- Every row of data (INSERT INTO)
- Every index (CREATE INDEX)
- Every constraint (FOREIGN KEY etc)

Writes everything into ONE backup file.
```

## When to use pg_dump
```
✅ Restore just one table
✅ Restore one database
✅ Copy database to test environment
✅ Small databases under 10GB
✅ Need human readable backup
✅ Before running risky migrations
```

## Backup Formats

| Format | Flag | Size | Readable | Single Table |
|---|---|---|---|---|
| Custom | `-F c` | Small | No | Yes ✅ |
| Plain SQL | `-F p` | Large | Yes ✅ | No |
| Directory | `-F d` | Medium | No | Yes ✅ |
| Tar | `-F t` | Medium | No | Yes ✅ |

**Always use custom format (`-F c`) in production!**

## Commands

### Backup single database
```bash
pg_dump -U admin -d mydb -F c -f backup.dump
```

### Backup single table only
```bash
pg_dump -U admin -d mydb -t tablename -F c -f table.dump
```

### Backup all databases + users + roles
```bash
pg_dumpall -U admin -f all_databases.sql
```

### List contents of backup
```bash
pg_restore --list backup.dump
```

## Restore Commands

### Restore full database
```bash
pg_restore -U admin -d mydb -v backup.dump
```

### Restore single table
```bash
pg_restore -U admin -d mydb \
  -t tablename \
  --clean \
  --if-exists \
  -v backup.dump
```

### Restore all databases
```bash
psql -U admin -f all_databases.sql
```

## Important Flags

| Flag | Meaning |
|---|---|
| `-U` | Username |
| `-d` | Database name |
| `-F c` | Custom format (recommended) |
| `-F p` | Plain SQL format |
| `-f` | Output file |
| `-v` | Verbose output |
| `-t` | Single table only |
| `--clean` | Drop before restore |
| `--if-exists` | Skip error if not exists |

## Common Error — Duplicate Key
```
ERROR: relation already exists
ERROR: duplicate key value violates unique constraint

Why: Table already exists when you try to restore
Fix: Add --clean --if-exists flags

pg_restore -U admin -d mydb \
  --clean --if-exists \
  -v backup.dump
```

## Sequence Fix After Restore
```sql
-- If you get "null value in id" error after restore
-- The sequence counter was reset

-- Find sequence name
\d tablename

-- Fix it
SELECT setval('sequence_name', (SELECT MAX(id) FROM tablename));
```

## Real Production Scenarios

### Scenario 1 — Developer deleted table
```bash
pg_restore -U admin -d mydb \
  -t deleted_table \
  --clean --if-exists -v backup.dump
```

### Scenario 2 — Copy to test environment
```bash
pg_dump -U admin -d proddb -F c -f prod.dump
pg_restore -U admin -d testdb -v prod.dump
```

### Scenario 3 — Before risky migration
```bash
# Always backup before major changes!
pg_dump -U admin -d mydb -F c \
  -f "pre_migration_$(date +%Y%m%d_%H%M%S).dump"
```

## pg_dump vs pg_dumpall

| Feature | pg_dump | pg_dumpall |
|---|---|---|
| Backs up | Single database | ALL databases |
| Includes users/roles | No ❌ | Yes ✅ |
| Includes tablespaces | No ❌ | Yes ✅ |
| Use for | Regular backups | Full server backup |

## Interview Questions

**Q: What backup format do you use in production?**
Custom format (-F c). Compressed, supports parallel
restore, and allows restoring single tables. Only use
plain format when I need to read or edit backup manually.

**Q: How do you restore a single table?**
pg_restore -t tablename --clean --if-exists -v backup.dump
The --clean flag drops existing table first to avoid
duplicate key errors.

**Q: What is difference between pg_dump and pg_dumpall?**
pg_dump backs up single database. pg_dumpall backs up
all databases plus global objects like roles and
tablespaces. Always use pg_dumpall for full server
backup so user accounts are also saved.
