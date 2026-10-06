cat > ~/Desktop/week-6-database-assignment/README.md << 'EOF'
# Week 6 — PostgreSQL Backups, PITR, and Replication Lab

## Overview

This lab exercised the full backup and disaster-recovery toolkit:

1. **Logical backup** — `pg_dump` custom-format backup + verified restore
2. **WAL archiving** — enabled continuous archiving + base backup
3. **Point-in-Time Recovery (PITR)** — recovered from a `DELETE` to a precise moment
4. **Streaming standby** — built a replica with `pg_basebackup -R`
5. **Replication health** — verified live streaming with `pg_stat_replication`

## Step 1 — Logical Backup

- Backup created successfully
- Restored to `bootcamp_check`
- Verified row count: **2,000,000** orders

## Step 2 — WAL Archiving

Config additions in `/etc/postgresql/16/main/postgresql.conf`:

- Restarted PostgreSQL
- Triggered a WAL switch with `SELECT pg_switch_wal();`
- Confirmed WAL file appeared in `~/backups/wal/`
- Base backup: `pg_basebackup -D ~/backups/base -Ft -z -Xs -P`

## Step 3 — Point-in-Time Recovery

Timeline of events:

| Time | Action |
|---|---|
| 21:49:23 | Recorded T0 with SELECT now() |
| 21:49:23 | DELETE FROM orders — simulated disaster |
| 21:49:23 | Verified 0 rows remained |
| 21:51:50 | Stopped PostgreSQL |
| — | Replaced data directory with base backup |
| — | Set recovery_target_time to 2026-10-06 21:49:20+03 |
| — | Created recovery.signal |
| 22:02:39 | Started recovery — PostgreSQL replayed WAL |
| 22:02:50 | Recovery paused before the DELETE transaction |
| — | Verified 2,000,000 rows restored |
| — | pg_wal_replay_resume() to make it live |

## Step 4 — Streaming Standby

- Created replicator role with REPLICATION privilege
- Configured pg_hba.conf to allow replication connections
- Built standby: pg_basebackup -h 127.0.0.1 -U replicator -D ~/standby -R -P
- The -R flag wrote standby.signal and primary_conninfo automatically

## Step 5 — Replication Health

Created a second PostgreSQL cluster on port 5433 and started it as a standby.

Query on primary:

    SELECT application_name, state,
           pg_wal_lsn_diff(sent_lsn, replay_lsn) AS lag_bytes
    FROM pg_stat_replication;

Result:

    application_name |   state   | lag_bytes
    -----------------+-----------+----------
    16/standby       | streaming |         0

Live replication test:
- Inserted into repl_test on primary
- Read the same row from standby (port 5433)

## Key Takeaways

1. Tested restores are the only trustworthy backups
2. WAL archiving + base backup enables PITR to any moment
3. pg_basebackup -R makes standby setup nearly automatic
4. pg_stat_replication is the primary tool for monitoring replication lag
5. RPO/RTO drive infrastructure choices — sync vs async, hot vs cold standby

## Files

- screenshots/step1_backup.png — backup and restore verification
- screenshots/step2_wal_archiving.png — WAL archive + base backup
- screenshots/step3_pitr.png — PITR recovery log
- screenshots/step4_standby.png — standby base backup
- screenshots/step5_replication.png — replication status + live test
