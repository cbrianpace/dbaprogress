# Transactions

## Locks

### Active Locks

```sql
SELECT  l.locktype,
        d.datname,
        r.relname,
        l.mode AS lock_mode,
        l.granted,
        l.pid,
        a.query,
        a.usename AS username,
        a.client_addr,
        a.wait_event_type,
        a.wait_event
FROM    pg_locks l
        LEFT OUTER JOIN pg_database d ON l.database = d.oid
        LEFT OUTER JOIN pg_class r ON l.relation = r.oid
        LEFT OUTER JOIN pg_stat_activity a ON l.pid = a.pid
WHERE   l.pid != pg_backend_pid()
ORDER BY l.pid;
```

### Blocking Locks

```sql
WITH sos AS (
    SELECT array_cat(array_agg(pid),
           array_agg((pg_blocking_pids(pid))[array_length(pg_blocking_pids(pid),1)])) pids
    FROM pg_locks
    WHERE NOT granted
)
SELECT  a.pid, a.usename, a.datname, a.state, 
        a.wait_event_type || ': ' || a.wait_event AS wait_event, 
        current_timestamp-a.state_change time_in_state,
        current_timestamp-a.xact_start time_in_xact,
        l.relation::regclass relname,
        l.locktype, l.mode, l.page, l.tuple,
        pg_blocking_pids(l.pid) blocking_pids,
        (pg_blocking_pids(l.pid))[array_length(pg_blocking_pids(l.pid),1)] last_session,
        coalesce((pg_blocking_pids(l.pid))[1]||'.'||coalesce(case when locktype='transactionid' then 1 else array_length(pg_blocking_pids(l.pid),1)+1 end,0),a.pid||'.0') lock_depth,
        a.query
FROM    pg_stat_activity a
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
SELECT  lpad('',(coalesce(case when locktype='transactionid' 
                               then 1 
                               else array_length(pg_blocking_pids(l.pid),1)+1 
                          end,0)+1)*5) || a.pid::text pid,
        l.relation::regclass relname, l.locktype, l.mode, l.page, l.tuple,
        coalesce((pg_blocking_pids(l.pid))[1]||'.'||
                    to_char(coalesce(case when locktype='transactionid' 
                                          then 1 
                                          else array_length(pg_blocking_pids(l.pid),1)+1 
                                     end,0),'FM000'),a.pid||'.000'
        ) lock_depth,
       pg_blocking_pids(a.pid) blocking_pids
FROM pg_stat_activity a
     JOIN sos s on (a.pid = any(s.pids))
     LEFT OUTER JOIN pg_locks l on (a.pid = l.pid and not l.granted)
ORDER BY lock_depth;
```

### Lock Summary by Database

```sql
SELECT  pg_database.datname as dbname, tmp.mode, COALESCE(count,0) as count
FROM (VALUES ('accesssharelock'), ('rowsharelock'), ('rowexclusivelock'), ('shareupdateexclusivelock'),
             ('sharelock'), ('sharerowexclusivelock'), ('exclusivelock'), ('accessexclusivelock')) AS tmp(mode) 
    CROSS JOIN pg_catalog.pg_database
    LEFT JOIN (SELECT   database, lower(mode) AS mode,count(*) AS count
               FROM     pg_catalog.pg_locks 
               WHERE database IS NOT NULL
               GROUP BY database, lower(mode)) AS tmp2 ON tmp.mode=tmp2.mode AND pg_database.oid = tmp2.database;
```

## Transaction Summary by Database

```sql
SELECT  s.datname as dbname, 
        xact_commit+xact_rollback as total_xact,
        xact_commit as commits, 
        xact_rollback as rollbacks, 
        conflicts, 
        deadlocks 
FROM    pg_catalog.pg_stat_database s 
        JOIN pg_catalog.pg_database d on d.datname = s.datname 
WHERE   d.datistemplate = false;
```
