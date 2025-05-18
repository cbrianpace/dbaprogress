# Postgres Performance Scripts

## Activity

### Session Summary

```sql
SELECT 
    pid,datname, usename, application_name,     
    state,
    wait_event_type || ': ' || wait_event AS wait_event, 
    pg_blocking_pids(pid) AS blocking_pids,
    current_timestamp-state_change time_in_state,
    current_timestamp-xact_start time_in_xact,
    to_char(state_change, 'YYYY-MM-DD HH24:MI:SS TZ') AS state_change, 
    to_char(query_start, 'YYYY-MM-DD HH24:MI:SS TZ') AS query_start,
    to_char(xact_start, 'YYYY-MM-DD HH24:MI:SS TZ') AS xact_start,
    to_char(backend_start, 'YYYY-MM-DD HH24:MI:SS TZ') AS backend_start,
    backend_type, 
    query
FROM
    pg_stat_activity
WHERE usename is not null
ORDER BY state, wait_event;
```

### State Summary

```sql
 SELECT state, wait_event_type || ': ' || wait_event AS wait_event, 
       count(1) cnt, 
	   min(current_timestamp-state_change) min_time_in_state,
	   avg(current_timestamp-state_change) avg_time_in_state,
	   max(current_timestamp-state_change) max_time_in_state
FROM pg_stat_activity
GROUP BY state, wait_event_type || ': ' || wait_event
ORDER BY 1,2;
```

#### User/Stage Summary

```sql
SELECT usename, state, count(1) cnt
FROM pg_stat_activity
WHERE usename is not null
GROUP BY usename, state
ORDER BY usename, state;
```

### bgWriter Stats

```sql
-- pgMonitor Exporter (ccp_stat_bgwriter)
SELECT 
    checkpoints_timed, checkpoints_req, checkpoint_write_time, checkpoint_sync_time, 
    buffers_checkpoint, buffers_clean, maxwritten_clean, buffers_backend, buffers_backend_fsync, 
    buffers_alloc, stats_reset 
FROM 
    pg_catalog.pg_stat_bgwriter;
```

### Block IO

```sql
SELECT
    (SELECT sum(blks_read) as "Read" FROM pg_stat_database),
    (SELECT sum(blks_hit) as "Hits" FROM pg_stat_database)
```

### Blocking Locks

```sql
WITH sos AS (
	SELECT array_cat(array_agg(pid),
           array_agg((pg_blocking_pids(pid))[array_length(pg_blocking_pids(pid),1)])) pids
	FROM pg_locks
	WHERE NOT granted
)
SELECT a.pid, a.usename, a.datname, a.state, 
	   a.wait_event_type || ': ' || a.wait_event AS wait_event, 
       current_timestamp-a.state_change time_in_state,
       current_timestamp-a.xact_start time_in_xact,
       l.relation::regclass relname,
       l.locktype, l.mode, l.page, l.tuple,
       pg_blocking_pids(l.pid) blocking_pids,
       (pg_blocking_pids(l.pid))[array_length(pg_blocking_pids(l.pid),1)] last_session,
       coalesce((pg_blocking_pids(l.pid))[1]||'.'||coalesce(case when locktype='transactionid' then 1 else array_length(pg_blocking_pids(l.pid),1)+1 end,0),a.pid||'.0') lock_depth,
       a.query
FROM pg_stat_activity a
     JOIN sos s on (a.pid = any(s.pids))
     LEFT OUTER JOIN pg_locks l on (a.pid = l.pid and not l.granted)
ORDER BY lock_depth;
```

### Blocking Locks Tree

```sql
WITH sos AS (
	SELECT array_cat(array_agg(pid),
           array_agg((pg_blocking_pids(pid))[array_length(pg_blocking_pids(pid),1)])) pids
	FROM pg_locks
	WHERE NOT granted
)
SELECT lpad('',(coalesce(case when locktype='transactionid' then 1 else array_length(pg_blocking_pids(l.pid),1)+1 end,0)+1)*5) || a.pid::text pid,
       l.relation::regclass relname, l.locktype, l.mode, l.page, l.tuple,
       coalesce((pg_blocking_pids(l.pid))[1]||'.'||to_char(coalesce(case when locktype='transactionid' then 1 else array_length(pg_blocking_pids(l.pid),1)+1 end,0),'FM000'),a.pid||'.000') lock_depth,
       pg_blocking_pids(a.pid) blocking_pids
FROM pg_stat_activity a
     JOIN sos s on (a.pid = any(s.pids))
     LEFT OUTER JOIN pg_locks l on (a.pid = l.pid and not l.granted)
ORDER BY lock_depth;
```

### Database Stats

```sql
-- pgMonitor Exporter (ccp_stat_database)
SELECT 
    s.datname as dbname, xact_commit, xact_rollback, blks_read, blks_hit, tup_returned, 
    tup_fetched, tup_inserted, tup_updated, tup_deleted, conflicts, temp_files, temp_bytes, deadlocks 
FROM 
    pg_catalog.pg_stat_database s JOIN pg_catalog.pg_database d on d.datname = s.datname 
WHERE 
    d.datistemplate = false
```

### Index Stats

```sql
-- Hit Ratio
SELECT 'index hit rate' as name,
(sum(idx_blks_hit) - sum(idx_blks_read)) / sum(idx_blks_hit + idx_blks_read) as ratio
From pg_statio_user_indexes
Union all
Select 'cache hit rate' as name,
Case sum(idx_blks_hit) when 0 then 'NaN'::numeric
Else to_char((sum(idx_blks_hit) - sum(idx_blks_read)) / sum(idx_blks_hit + idx_blks_read),'99.99')::numeric end as ration
From pg_statio_user_indexes;
```

```sql
-- Index Hit Rate
Select relname, 100* idx_scan / (seq_scan + idx_scan), n_live_tup from pg_stat_user_tables order by n_live_typ desc;
```

### Locks

#### Active Locks

```sql
SELECT 
    l.locktype,
    d.datname AS database,
    r.relname AS relation,
    l.mode AS lock_mode,
    l.granted,
    l.pid,
    a.query,
    a.usename AS username,
    a.client_addr,
    a.wait_event_type,
    a.wait_event
FROM 
    pg_locks l
    LEFT OUTER JOIN pg_database d ON l.database = d.oid
    LEFT OUTER JOIN pg_class r ON l.relation = r.oid
    LEFT OUTER JOIN pg_stat_activity a ON l.pid = a.pid
WHERE 
    l.pid != pg_backend_pid()
ORDER BY 
    l.pid;
```

#### Lock Summary by Database

```sql
SELECT 
    pg_database.datname as dbname, tmp.mode, COALESCE(count,0) as count
FROM
(
  VALUES ('accesssharelock'),
         ('rowsharelock'),
         ('rowexclusivelock'),
        ('shareupdateexclusivelock'),
         ('sharelock'),
        ('sharerowexclusivelock'),
         ('exclusivelock'),
         ('accessexclusivelock')
) AS tmp(mode) CROSS JOIN pg_catalog.pg_database
LEFT JOIN
  (SELECT database, lower(mode) AS mode,count(*) AS count
   FROM pg_catalog.pg_locks WHERE database IS NOT NULL
   GROUP BY database, lower(mode)
) AS tmp2
ON tmp.mode=tmp2.mode and pg_database.oid = tmp2.database;
```

### Session Info (General)

```sql
select datname, pid, leader_pid, backend_type,
       usename, application_name,
       state, wait_event_type, wait_event,
       trunc(extract(epoch from current_timestamp-backend_start)) session_age,
       trunc(extract(epoch from current_timestamp-xact_start)) transaction_age,
       trunc(extract(epoch from current_timestamp-query_start)) query_age,
       trunc(extract(epoch from current_timestamp-state_change)) state_age,
       query,
       client_addr, client_hostname
from pg_stat_activity
order by state, usename;
```

### Table Stats

```sql
-- pgMonitor Exporter (ccp_stat_user_tables)
SELECT 
    current_database() as dbname, schemaname, relname, seq_scan, seq_tup_read, 
    idx_scan, idx_tup_fetch, n_tup_ins, n_tup_upd, n_tup_del, n_tup_hot_upd, n_live_tup, n_dead_tup, 
    vacuum_count, autovacuum_count, analyze_count, autoanalyze_count 
FROM 
    pg_catalog.pg_stat_user_tables;
```

### Transactions

```sql
SELECT
    (SELECT sum(xact_commit) + sum(xact_rollback) AS "Total" FROM pg_stat_database),
    (SELECT sum(xact_commit) AS "Commit" FROM pg_stat_database),
    (SELECT sum(xact_rollback) AS "Rollback" FROM pg_stat_database)
```
