# Sessions

## Session Info

### Current Session 

```sql
SELECT current_database(), current_role, current_schema(), 
       current_schemas(true), pg_backend_pid(), pg_current_xact_id(),
       inet_client_addr(), inet_server_addr()  ;
```

### Session Status

```sql
SELECT datname, usename, 
       sum(case when state='active' then 1 else 0 end) as active,
       sum(case when state='idle' then 1 else 0 end) as idle,
       sum(case when state is null then 1 else 0 end) as background,
       count(1) as total
 FROM pg_catalog.pg_stat_activity
 GROUP BY datname, usename;
```

```sql
SELECT sum(case when state='active' then 1 else 0 end) as active,
       sum(case when state='idle' then 1 else 0 end) as idle,
       sum(case when state is null then 1 else 0 end) as background,
       count(1) as total
FROM pg_catalog.pg_stat_activity;
```

```sql
SELECT ((total - idle) - idle_in_txn) as active, total, idle, idle_in_txn,
       (SELECT coalesce(extract(epoch from (max(now() - state_change))),0) 
        FROM pg_catalog.pg_stat_activity 
        WHERE state = 'idle in transaction') as max_idle_in_txn_time,
       (SELECT coalesce(extract(epoch from (max(now() - query_start))),0) 
        FROM pg_catalog.pg_stat_activity 
        WHERE backend_type = 'client backend' and state <> 'idle' ) as max_query_time,
       (SELECT coalesce(extract(epoch from (max(now() - query_start))),0) 
        FROM pg_catalog.pg_stat_activity 
        WHERE backend_type = 'client backend' and wait_event_type = 'Lock' ) as max_blocked_query_time,
        max_connections
FROM (SELECT count(*) as total,
             coalesce(sum(case when state = 'idle' then 1 else 0 end),0) as idle,
             coalesce(sum(case when state = 'idle in transaction' then 1 else 0 end),0) as idle_in_txn 
      FROM pg_catalog.pg_stat_activity) x
           JOIN (SELECT setting::float AS max_connections 
                 FROM pg_settings 
                 WHERE name = 'max_connections') xx ON (true);
```

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

## Cancel Session

```sql
SELECT pg_cancel_backend(<pid>);
```

## Disconnect Session

```sql
SELECT pg_terminate_backend(<pid>);
```

```sql
SELECT pg_terminate_backend(pid) 
FROM pg_stat_activity 
WHERE usename = '<role>';
```

## Cancel All Long Running SQL

```sql
SELECT
  pid,
  now() - pg_stat_activity.query_start AS duration,
  query,
  state, pg_cancel_backend(pid) as cancelled
FROM pg_stat_activity
WHERE (now() - pg_stat_activity.query_start) > interval '5 minutes' and backend_type='client backend' and state='active';
```

## Session Info with Client Details

```sql
SELECT datname, a.pid as pid, usename, client_addr, client_port,                           
       round(extract(epoch from (now() - xact_start))) as age,                           
       wait_event_type IS NOT DISTINCT FROM 'Lock' AS waiting,
       NULLIF(array_to_string(ARRAY(SELECT unnest(pg_blocking_pids(a.pid)) ORDER BY 1), ','), '') as locked_by,
       CASE WHEN state = 'idle in transaction' THEN
                 CASE WHEN xact_start != state_change THEN
                       'idle in transaction ' || CAST(                                                 abs(round(extract(epoch from (now() - state_change)))) AS text)                                           
                      ELSE 'idle in transaction'
                 END
             WHEN state = 'active' THEN query                                   
             ELSE state
       END AS query
FROM pg_stat_activity a
WHERE a.pid != pg_backend_pid() AND a.datname IS NOT NULL                      GROUP BY 1,2,3,4,5,6,7,9
```