# Postgres Library

## Replication

### Logical

#### Publications

```sql
select * from pg_publication;
```

#### Publication Tables

```sql
select * from pg_publication_tables;
```

#### Subscriptions

```sql
select * from pg_catalog.pg_subscription ;
```

#### Subscription Stats

```sql
select * from pg_catalog.pg_stat_subscription_stats;
```

#### Replication Lag/Performance Summary

```sql
select r.pid, r.usename, r.application_name, pg_current_wal_lsn(),
       pg_current_wal_lsn()-r.replay_lsn replay_lag_bytes,
       s.total_txns, s.total_bytes 
from pg_stat_replication r
     left outer join pg_stat_replication_slots s 
          on (r.application_name=s.slot_name);
```

### Physical

#### Source: Current WAL LSN

```sql
SELECT pg_current_wal_lsn();
```

#### Target: Last WAL LSN Received and Applied

```sql
SELECT pg_last_wal_receive_lsn(), pg_last_wal_replay_lsn();
```

#### Pause WAL Replay/Apply

```sql
SELECT pg_wal_replay_pause();
```

#### Resume WAL Replay/Apply

```sql
SELECT pg_wal_replay_resume();
```

#### Replication Lag (in bytes)

```sql
-- pgMonitor Exporter (ccp_replication_lag_size)
SELECT 
    client_addr as replica, client_hostname as replica_hostname, client_port as replica_port,
    pg_wal_lsn_diff(sent_lsn, replay_lsn) as bytes
FROM 
    pg_catalog.pg_stat_replication
```

#### Replication Lag (in seconds)

```sql
-- pgMonitor Exporter (ccp_replication_lag)
SELECT
    CASE
      WHEN pg_last_wal_receive_lsn() = pg_last_wal_replay_lsn() THEN 0
      ELSE EXTRACT (EPOCH FROM now() - pg_last_xact_replay_timestamp())::INTEGER
     END AS replay_time,
     EXTRACT (EPOCH FROM now() - pg_last_xact_replay_timestamp())::INTEGER AS received_time
```

#### Replication Slots

```sql
-- pgMonitor Exporter (ccp_replication_slots)
SELECT 
    slot_name, active::int, pg_wal_lsn_diff(pg_current_wal_insert_lsn(), restart_lsn) AS retained_bytes 
FROM 
    pg_catalog.pg_replication_slots;
```

#### Replication Status

Gap between pg_current_wal_lsn and sent_lsn may indicate heavy load on source system.  Gap between sent_lsn and pg_last_wal_receive_lsn on standby may indicate network delay or standby under heavy load.

```sql
SELECT
   pg_is_in_recovery(),pg_is_wal_replay_paused(), pg_last_wal_receive_lsn(), 
    pg_last_wal_replay_lsn(), pg_last_xact_replay_timestamp()
```

```sql
-- Source:
select pid, usesysid, usename, application_name, client_addr, client_hostname, client_port, backend_start, backend_xmin,
       state, pg_current_wal_lsn() as current_wal_lsn, sent_lsn, write_lsn, flush_lsn, replay_lsn, write_lag, flush_lag,
       replay_lag, sync_priority, sync_state, reply_time
from pg_stat_replication;
```

```sql
-- Target:
select * from pg_stat_wal_receiver;
```

#### Replication Lag

Execute on replication target.

```sql
SELECT now() - pg_last_xact_replay_timestamp()
```

## Security

### Table Grants

```sql
SELECT grantee
      ,table_catalog
      ,table_schema
      ,table_name
      ,string_agg(privilege_type, ', ' ORDER BY privilege_type) AS privileges
FROM information_schema.role_table_grants
WHERE grantee != 'postgres' 
      -- grantee NOT IN ('postgres','PUBLIC')
GROUP BY grantee, table_catalog, table_schema, table_name;
```

### Who Can Connect to What Databases?

```sql
--By User:
select pgu.usename as user_name,
       (select string_agg(pgd.datname, ',' order by pgd.datname) 
        from pg_database pgd 
        where has_database_privilege(pgu.usename, pgd.datname, 'CONNECT')) as database_name
from pg_user pgu
order by pgu.usename;
```

```sql
--By Database:
select pgd.datname as database_name,
       (select string_agg(pgu.usename, ',' order by pgu.usename) 
        from pg_user pgu 
        where has_database_privilege(pgu.usename, pgd.datname, 'CONNECT')) as user_name
from pg_database pgd
order by pgd.datname;

### HBA Rules

```sql
select * from pg_catalog.pg_hba_file_rules;
```
## Statements

### pg_stats_statements

```sql
-- pgMonitor Exporter (ccp_pg_stat_statements)
SELECT pg_get_userbyid(s.userid) as role, d.datname AS dbname, s.queryid, 
       btrim(replace(left(s.query, 40), '\n', '')) AS query, s.plans, 
       s.total_plan_time AS total_plan_time_ms, s.min_plan_time AS min_plan_time_ms,
       s.max_plan_time AS max_plan_time_ms, s.mean_plan_time AS mean_plan_time_ms,
       s.stddev_plan_time AS stddev_plan_time_ms, s.calls, s.total_exec_time AS total_exec_time_ms,
       s.min_exec_time AS min_exec_time_ms, s.max_exec_time AS max_exec_time_ms,
       s.mean_exec_time AS mean_exec_time_ms, s.stddev_exec_time AS stddev_exec_time_ms, s.rows,
       s.shared_blks_hit, s.shared_blks_read, s.shared_blks_dirtied, s.shared_blks_written,
       s.local_blks_hit, s.local_blks_read, s.local_blks_dirtied, s.local_blks_written,
       s.temp_blks_read, s.temp_blks_written, s.blk_read_time AS blk_read_time_ms,
       s.blk_write_time AS blk_write_time_ms, s.wal_records, s.wal_fpi, s.wal_bytes
FROM public.pg_stat_statements s
     JOIN pg_catalog.pg_database d ON d.oid = s.dbid;
```

```sql
Select (total_time / 1000 / 60) as total, (total_time/calls) as avg, query from pg_stat_statements order by 1 desc limit 100;
```

### pg_stats_statements Reset

```sql
SELECT pg_stat_statements_reset();
```

### Stat Statements Total

```sql
-- pgMonitor Exporter (ccp_pg_stat_statements_total)
SELECT pg_get_userbyid(s.userid) as role, d.datname AS dbname, sum(s.calls) AS calls_count,
       sum(s.total_exec_time) AS exec_time_ms, avg(s.mean_exec_time) AS mean_exec_time_ms,
       sum(s.rows) AS row_count
FROM public.pg_stat_statements s
     JOIN pg_catalog.pg_database d ON d.oid = s.dbid
GROUP BY 1,2
```

### Top Mean/Avg

```sql
-- pgMonitor Exporter (ccp_pg_stat_statements_top_mean)
SELECT pg_get_userbyid(s.userid) as role, d.datname AS dbname, s.queryid,
       btrim(replace(left(s.query, 40), '\n', '')) AS query, max(s.mean_exec_time) exec_time_ms
FROM public.pg_stat_statements s
     JOIN pg_catalog.pg_database d ON d.oid = s.dbid
GROUP BY 1,2,3,4
ORDER BY 5 DESC
LIMIT #PG_STAT_STATEMENTS_LIMIT#
```

### Top Max

```sql
-- pgMonitor Exporter (ccp_pg_stat_statements_top_max)
SELECT pg_get_userbyid(s.userid) as role, d.datname AS dbname, s.queryid,
       btrim(replace(left(s.query, 40), '\n', '')) AS query, s.max_exec_time AS exec_time_ms
FROM public.pg_stat_statements s
     JOIN pg_catalog.pg_database d ON d.oid = s.dbid
ORDER BY 5 DESC
LIMIT #PG_STAT_STATEMENTS_LIMIT#;
```

### Top Total

```sql
-- Exporter (ccp_pg_stat_statements_top_total)
SELECT pg_get_userbyid(s.userid) as role, d.datname AS dbname, s.queryid,
       btrim(replace(left(s.query, 40), '\n', '')) AS query, s.total_exec_time exec_time_ms
FROM public.pg_stat_statements s
     JOIN pg_catalog.pg_database d ON d.oid = s.dbid
ORDER BY 5 DESC
LIMIT #PG_STAT_STATEMENTS_LIMIT#
```

## Tables

### General Size

```sql
SELECT *, pg_size_pretty(total_bytes) AS total
    , pg_size_pretty(index_bytes) AS index
    , pg_size_pretty(toast_bytes) AS toast
    , pg_size_pretty(table_bytes) AS table
  FROM (
  SELECT *, total_bytes-index_bytes-coalesce(toast_bytes,0) AS table_bytes FROM (
      SELECT c.oid,nspname AS table_schema, relname AS table_name
              , c.reltuples AS row_estimate
              , pg_total_relation_size(c.oid) AS total_bytes
              , pg_indexes_size(c.oid) AS index_bytes
              , pg_total_relation_size(reltoastrelid) AS toast_bytes
          FROM pg_class c
          LEFT JOIN pg_namespace n ON n.oid = c.relnamespace
          WHERE relkind = 'r'
  ) a
) a;
```

### Check for Bloat

```sql
SELECT  objectname, pg_size_pretty(size_bytes) as object_size,     
        pg_size_pretty(free_space_bytes) as reusable_space, 
        pg_size_pretty(dead_tuple_size_bytes) dead_tuple_space, 
        free_percent
FROM bloat_stats;
```

### Column Stats

```sql
select * from pg_stats where tablename='mytable';
```

### Size (unpartitioned)

```sql
WITH RECURSIVE pg_inherit(inhrelid, inhparent) AS
    (select inhrelid, inhparent
    FROM pg_inherits
    UNION
    SELECT child.inhrelid, parent.inhparent
    FROM pg_inherit child, pg_inherits parent
    WHERE child.inhparent = parent.inhrelid),
pg_inherit_short AS (SELECT * FROM pg_inherit WHERE inhparent NOT IN (SELECT inhrelid FROM pg_inherit))
SELECT table_schema
    , TABLE_NAME
    , row_estimate
    , pg_size_pretty(total_bytes) AS total
    , pg_size_pretty(index_bytes) AS INDEX
    , pg_size_pretty(toast_bytes) AS toast
    , pg_size_pretty(table_bytes) AS TABLE
    , round(total_bytes::float8 / sum(total_bytes) OVER ()) AS total_size_share
  FROM (
    SELECT *, total_bytes-index_bytes-COALESCE(toast_bytes,0) AS table_bytes
    FROM (
         SELECT c.oid
              , nspname AS table_schema
              , relname AS TABLE_NAME
              , SUM(c.reltuples) OVER (partition BY parent) AS row_estimate
              , SUM(pg_total_relation_size(c.oid)) OVER (partition BY parent) AS total_bytes
              , SUM(pg_indexes_size(c.oid)) OVER (partition BY parent) AS index_bytes
              , SUM(pg_total_relation_size(reltoastrelid)) OVER (partition BY parent) AS toast_bytes
              , parent
          FROM (
                SELECT pg_class.oid
                    , reltuples
                    , relname
                    , relnamespace
                    , pg_class.reltoastrelid
                    , COALESCE(inhparent, pg_class.oid) parent
                FROM pg_class
                    LEFT JOIN pg_inherit_short ON inhrelid = oid
                WHERE relkind IN ('r', 'p')
             ) c
             LEFT JOIN pg_namespace n ON n.oid = c.relnamespace
  ) a
  WHERE oid = parent
) a
ORDER BY total_bytes DESC;
```

### Size (partitioned)

```sql
WITH RECURSIVEpg_inherit(inhrelid, inhparent) AS
    (selectinhrelid, inhparent
    FROMpg_inherits
    UNION
    SELECTchild.inhrelid, parent.inhparent
    FROMpg_inherit child, pg_inherits parent
    WHEREchild.inhparent = parent.inhrelid),
pg_inherit_short AS(SELECT* FROMpg_inherit WHEREinhparent NOTIN(SELECTinhrelid FROMpg_inherit))
SELECTparent::regclass
    , coalesce(spcname, 'default') pg_tablespace_name
    , row_estimate
    , pg_size_pretty(total_bytes) AStotal
    , pg_size_pretty(index_bytes) ASINDEX
    , pg_size_pretty(toast_bytes) AStoast
    , pg_size_pretty(table_bytes) ASTABLE
    , round(100* total_bytes::float8/ sum(total_bytes) OVER()) ASPERCENT
  FROM(
    SELECT*, total_bytes-index_bytes-COALESCE(toast_bytes,0) AStable_bytes
    FROM(
         SELECTparent
              , reltablespace
              , SUM(c.reltuples) ASrow_estimate
              , SUM(pg_total_relation_size(c.oid)) AStotal_bytes
              , SUM(pg_indexes_size(c.oid)) ASindex_bytes
              , SUM(pg_total_relation_size(reltoastrelid)) AStoast_bytes
          FROM(
                SELECTpg_class.oid
                    , reltuples
                    , relname
                    , relnamespace
                    , reltablespace reltablespace
                    , pg_class.reltoastrelid
                    , COALESCE(inhparent, pg_class.oid) parent
                FROMpg_class
                    LEFTJOINpg_inherit_short ONinhrelid = oid
                WHERErelkind IN('r', 'p')
             ) c
            GROUPBYparent, reltablespace
  ) a
) a LEFTJOINpg_tablespace ON(pg_tablespace.oid= reltablespace)
ORDERBYtotal_bytes DESC;
```

### Table Size

```sql
-- pgMonitor Exporter (ccp_table_size)
SELECT 
    current_database() as dbname, n.nspname as schemaname, c.relname, 
    pg_total_relation_size(c.oid) as size_bytes 
FROM 
    pg_catalog.pg_class c 
    JOIN pg_catalog.pg_namespace n ON c.relnamespace = n.oid 
WHERE 
    NOT pg_is_other_temp_schema(n.oid) AND relkind IN ('r', 'm', 'f');
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

### Percent to Emergency Vacuum & Wraparound

```sql
WITH table_option (oid, val) AS (
	SELECT oid, unnest(reloptions)
	FROM pg_class WHERE reloptions::text ~ 'autovacuum_freeze_max_age'
), autovac_option AS (
	SELECT oid, val
	FROM table_option WHERE val LIKE 'autovacuum_freeze_max_age%'
)
SELECT n.nspname, c.relname, c.relpages,
       CASE WHEN c.reloptions::text ~ 'autovacuum_enabled=false' THEN true ELSE false END AS autovacuum_disabled,
       coalesce(CASE WHEN ao.val IS NOT NULL THEN substring(ao.val,position('=' in ao.val)+1) 
                     ELSE null 
                END, current_setting('autovacuum_freeze_max_age'))::numeric autovaccum_freeze_max_age,
       round(((greatest(age(c.relfrozenxid), age(t.relfrozenxid))::numeric) / 
            coalesce(CASE WHEN ao.val IS NOT NULL THEN  substring(ao.val,position('=' in ao.val)+1) 
                          ELSE null 
                     END, current_setting('autovacuum_freeze_max_age'))::numeric)*100) pct_emergency_vac,      
       round(((greatest(age(c.relfrozenxid), age(t.relfrozenxid))::numeric) / 2146483647)*100)  pct_wraparound,
       c.relfrozenxid table_relfrozenxid, 
       age(c.relfrozenxid) table_relfrozenxid_age,
       t.relfrozenxid toast_relfrozenxid,       
       age(t.relfrozenxid) toast_relfrozenxid_age       
FROM pg_class c 
     JOIN pg_namespace n ON n.oid = c.relnamespace
     LEFT OUTER JOIN pg_class t ON c.reltoastrelid = t.oid
     LEFT OUTER JOIN autovac_option ao ON ao.oid = c.oid
WHERE c.relkind in ('r', 'm', 'f')
ORDER BY pct_emergency_vac DESC;


SELECT c.oid::regclass
    , age(c.relfrozenxid)
    , pg_size_pretty(pg_total_relation_size(c.oid))
    , current_setting('autovacuum_freeze_max_age')
    , round((age(c.relfrozenxid)::numeric / current_setting('autovacuum_freeze_max_age')::numeric)*100) pct_to_emer_vacuum
    , round((age(c.relfrozenxid)::numeric / 2146483647 )*100) pct_to_wraparound
FROM pg_class c
JOIN pg_namespace n on c.relnamespace = n.oid
WHERE relkind IN ('r', 't', 'm')
and c.oid in (76759, 76756)
--AND n.nspname NOT IN ('pg_toast')
ORDER BY 2 DESC LIMIT 100;
```

### Vacuum Progress

```sql
select a.pid, pc.datname, relid, heap_blks_scanned, heap_blks_total, phase, round((heap_blks_scanned::numeric/heap_blks_total::numeric)*100) pct_done,
       extract(epoch from current_timestamp - xact_start) elasped_time_seconds,
       round(heap_blks_scanned / extract(epoch from current_timestamp - xact_start)) blks_per_second,
       round(((heap_blks_total - heap_blks_scanned) / (heap_blks_scanned / extract(epoch from current_timestamp - xact_start)))) est_time_remaining 
from pg_catalog.pg_stat_progress_vacuum pc
     join pg_stat_activity a on (a.pid=pc.pid);
```

### Percent Toward Exceeding Maximum for Auto Incremental Columns

```sql
SELECT
    seqs.relname AS sequence,
    format_type(s.seqtypid, NULL::integer) sequence_datatype,
    CONCAT(tbls.relname, '.', attrs.attname) AS owned_by,
    format_type(attrs.atttypid, atttypmod) AS column_datatype,
    pg_sequence_last_value(seqs.oid::regclass) AS last_sequence_value,
    TO_CHAR((CASE WHEN format_type(s.seqtypid, NULL::integer) = 'smallint' THEN(pg_sequence_last_value(seqs.relname::regclass) / 32767::float)WHEN format_type(s.seqtypid, NULL::integer) = 'integer' THEN(pg_sequence_last_value(seqs.relname::regclass) / 2147483647::float)WHEN format_type(s.seqtypid, NULL::integer) = 'bigint' THEN(pg_sequence_last_value(seqs.relname::regclass) / 9223372036854775807::float)END) * 100, 'fm9999999999999999999990D00%') AS sequence_percent,
    TO_CHAR((CASE WHEN format_type(attrs.atttypid, NULL::integer) = 'smallint' THEN(pg_sequence_last_value(seqs.relname::regclass) / 32767::float)WHEN format_type(attrs.atttypid, NULL::integer) = 'integer' THEN(pg_sequence_last_value(seqs.relname::regclass) / 2147483647::float)WHEN format_type(attrs.atttypid, NULL::integer) = 'bigint' THEN(pg_sequence_last_value(seqs.relname::regclass) / 9223372036854775807::float)END) * 100, 'fm9999999999999999999990D00%') AS column_percent
FROM
    pg_depend d
    JOIN pg_class AS seqs ON seqs.relkind = 'S'AND seqs.oid = d.objid
    JOIN pg_class AS tbls ON tbls.relkind = 'r'AND tbls.oid = d.refobjid
    JOIN pg_attribute AS attrs ON attrs.attrelid = d.refobjid
        AND attrs.attnum = d.refobjsubid
    JOIN pg_sequence s ON s.seqrelid = seqs.oid
WHERE
    d.deptype = 'a'AND d.classid = 1259;
```

## WAL

### Activity

```sql
-- pgMonitor Exporter (ccp_wal_activity)
SELECT 
    last_5_min_size_bytes,
    (SELECT COALESCE(sum(size),0) FROM pg_catalog.pg_ls_waldir()) AS total_size_bytes
FROM 
    (SELECT COALESCE(sum(size),0) AS last_5_min_size_bytes 
     FROM pg_catalog.pg_ls_waldir() 
     WHERE modification > CURRENT_TIMESTAMP - '5 minutes'::interval) x
```

### Current WAL LSN

```sql
SELECT pg_current_wal_lsn();
```

### Last WAL Received/Replayed (Replication Target)

```sql
SELECT pg_last_wal_receive_lsn() last_received, pg_last_wal_replay_lsn() last_replayed
```

### Bytes Between Two LSNs

```sql
select '30/77000060'::pg_lsn - '2C/7D000000'::pg_lsn size_bytes;
```

### Useful PG_WALINSPECT Extension Queries

```sql
select * from pg_get_wal_records_info('275/20000000','275/200018F0');

select * from pg_get_wal_record_info('275/20001850');

select * from pg_get_wal_block_info('275/20000000','275/200018F0', true);

select * from pg_get_wal_stats('275/20000000','275/200018F0');

select n.nspname,
      case 
         when c.relkind = 'r' then 'table'
         when c.relkind = 'i' then 'index'
         when c.relkind = 'S' then 'sequence'
         when c.relkind = 't' then 'toast'
         when c.relkind = 'v' then 'view'
         when c.relkind = 'm' then 'materialized view'
         when c.relkind = 'c' then 'composite type'
         when c.relkind = 'f' then 'foreign table'
         when c.relkind = 'p' then 'partitioned table'
         when c.relkind = 'I' then 'partitioned index'
         else 'other'
       end object_type,
       c.relname, w.start_lsn, w.end_lsn, w.prev_lsn, w.relblocknumber,
       w.xid, w.resource_manager, w.record_type, 
       w.record_length, w.main_data_length,
       w.block_data_length, w.block_fpi_length, w.block_fpi_info,
       w.description, w.block_data, w.block_fpi_data
from pg_get_wal_block_info('275/20000000','275/200018F0', true) w
     join pg_class c on (c.relfilenode = w.relfilenode)
     join pg_namespace n on (n.oid = c.relnamespace)
     join pg_type t on (t.oid = c.reltype);
```