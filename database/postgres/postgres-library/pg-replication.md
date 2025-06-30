# Replication Monitoring

## Logical Replication

### Latency (Logical)

```sql
SELECT  subname,
        pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), received_lsn)) AS byte_lag,
        (CURRENT_TIMESTAMP - last_msg_receipt_time) AS time_lag
FROM    pg_stat_subscription;
```

```sql
SELECT s.subid, s.subname, s.received_lsn, s.latest_end_lsn,
       ros.remote_lsn,
       pg_wal_lsn_diff(s.received_lsn, ros.remote_lsn) as received_lag_bytes,
       pg_wal_lsn_diff(s.latest_end_lsn, ros.remote_lsn) as latest_end_lag_bytes,
       current_timestamp - last_msg_receipt_time as last_receipt_lag_time,
       current_timestamp - latest_end_time as latest_end_lag_time
FROM pg_stat_subscription s
     JOIN pg_replication_origin_status ros ON (concat('pg_',s.subid::text) = ros.external_id)
ORDER BY s.subname;
```

## Physical Replication

### Latency (Physical)

```sql
SELECT
    CASE
      WHEN pg_last_wal_receive_lsn() = pg_last_wal_replay_lsn() THEN 0
      ELSE EXTRACT (EPOCH FROM now() - pg_last_xact_replay_timestamp())::INTEGER
     END AS replay_time,
     EXTRACT (EPOCH FROM now() - pg_last_xact_replay_timestamp())::INTEGER AS received_time;
```

```sql
SELECT 
    client_addr as replica, client_hostname as replica_hostname, client_port as replica_port,
    pg_wal_lsn_diff(sent_lsn, replay_lsn) as bytes
FROM 
    pg_catalog.pg_stat_replication;
```

### WAL Receiver

```sql
SELECT * 
FROM   pg_stat_wal_receiver;
```

## Replication Slots

```sql
SELECT  slot_name, 
        active::int, 
        pg_wal_lsn_diff(pg_current_wal_insert_lsn(), restart_lsn) AS retained_bytes,
        pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), confirmed_flush_lsn)) AS lag
FROM    pg_catalog.pg_replication_slots;
```

## Subscription

```sql
SELECT
    subname,
    pid,
    relid,
    received_lsn,
    last_msg_send_time,
    last_msg_receipt_time,
    latest_end_lsn,
    latest_end_time,
    pg_size_pretty(pg_wal_lsn_diff(received_lsn, latest_end_lsn)) AS lag
FROM pg_stat_subscription;
```

```sql
SELECT s.oid, s.subname, s.subenabled, s.subfailover, s.subslotname, s.subpublications,
       sr.srrelid::regclass AS table_name, sr.srsubstate,
       sss.apply_error_count, sss.sync_error_count,
       ss.pid, ss.worker_type, ss.received_lsn, ss.last_msg_receipt_time,
       ss.latest_end_lsn, ss.latest_end_time
FROM pg_subscription s
     JOIN pg_subscription_rel sr ON (s.oid = sr.srsubid)
     JOIN pg_stat_subscription_stats sss ON (s.oid = sss.subid)
     JOIN pg_stat_subscription ss ON (s.oid = ss.subid);
```
