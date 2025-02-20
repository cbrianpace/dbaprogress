# Database

## Archive Command Status

```sql
-- pgMonitor Exporter (ccp_archive_command_status)
SELECT CASE 
         WHEN EXTRACT(epoch from (last_failed_time - last_archived_time)) IS NULL THEN 0
         WHEN EXTRACT(epoch from (last_failed_time - last_archived_time)) < 0 THEN 0
         ELSE EXTRACT(epoch from (last_failed_time - last_archived_time)) 
        END AS seconds_since_last_fail,
        EXTRACT(epoch from (CURRENT_TIMESTAMP - last_archived_time)) AS seconds_since_last_archive,
        archived_count,
        failed_count
FROM pg_catalog.pg_stat_archiver
```

## Bloat Check

Note:  Requires Custom Objects Per Database

```sql
-- pgMonitor Exporter (ccp_bloat_check)
SELECT current_database() AS dbname, schemaname, objectname, size_bytes,
      (dead_tuple_size_bytes + (free_space_bytes - (relpages - (fillfactor/100) * relpages ) * current_setting('block_size')::bigint ))::bigint AS total_wasted_space_bytes
FROM bloat_stats
```

## Checkpoint Settings

```sql
-- pgMonitor Exporter (ccp_settings_gauge)
SELECT (SELECT setting::int 
        FROM pg_catalog.pg_settings 
        WHERE name = 'checkpoint_timeout') as checkpoint_timeout, 
       (SELECT setting::float 
        FROM pg_catalog.pg_settings 
        WHERE name = 'checkpoint_completion_target') as checkpoint_completion_target,
       (SELECT 8192*setting::bigint as bytes 
        FROM pg_catalog.pg_settings 
        WHERE name = 'shared_buffers') as shared_buffers
```

## Checksum Failure

```sql
-- pgMonitor Exporter (ccp_data_checksum_failure)
SELECT datname AS dbname, checksum_failures AS count,
       coalesce(extract(epoch from (now()-checksum_last_failure)), 0) AS time_since_last_failure_seconds
FROM pg_catalog.pg_stat_database;
```

## Checksum Settings

```sql
-- pgMonitor Exporter (ccp_pg_settings_checksum)
SELECT monitor.pg_settings_checksum() AS status
```

## Configuration Settings (database level/cluster level)

```sql
SELECT coalesce(role.rolname, 'database wide') as role, 
       coalesce(db.datname, 'cluster wide') as database, 
       setconfig as what_changed
FROM pg_db_role_setting role_setting
LEFT JOIN pg_roles role ON role.oid = role_setting.setrole
LEFT JOIN pg_database db ON db.oid = role_setting.setdatabase;
```

## Database Info

```sql
SELECT pg_current_logfile(), pg_conf_load_time(), pg_postmaster_start_time(),    
      pg_current_snapshot(), version()
```

## Database Settings

```sql
-- Non-Default Settings
SELECT name, current_setting(name) 
FROM pg_settings 
WHERE source <> 'default' AND setting is distinct from boot_val;
```

```sql
-- All Settings
SELECT name, setting 
FROM pg_settings
```

## Database Size

```sql
-- pgMonitor Exporter (ccp_database_size)
SELECT datname as dbname, pg_database_size(datname) as bytes 
FROM pg_catalog.pg_database 
WHERE datistemplate = false
```

## In Recovery?

```sql
-- pgMonitor Exporter (ccp_is_in_recovery)
SELECT CASE WHEN pg_is_in_recovery = true THEN 1 ELSE 2 END AS status 
FROM pg_is_in_recovery();
```

## Postgres Version

```sql
-- pgMonitor Exporter (ccp_postgresql_version)
SELECT current_setting('server_version_num')::int AS current
```

## Postmaster Runtime

```sql
-- pgMonitor Exporter (ccp_postmaster_runtime)
SELECT extract('epoch' from pg_postmaster_start_time) as start_time_seconds 
FROM pg_catalog.pg_postmaster_start_time()
```

## Postmaster Uptime

```sql
-- pgMonitor Exporter (ccp_postmaster_uptime)
SELECT extract(epoch FROM (now() - pg_postmaster_start_time() )) AS seconds;
```

## Sequence Exhaustion

```sql
-- pgMonitor Exporter (ccp_sequence_exhaustion) *Uses ccp_monitoring view*
SELECT count 
FROM monitor.sequence_exhaustion(75)
```

## Settings Pending Restart

```sql
-- pgMonitor Exporter (ccp_settings_pending_restart)
SELECT count(*) AS count 
FROM pg_catalog.pg_settings WHERE pending_restart = true
```

## Shared Buffer Usage

```sql
SELECT c.relname
  , pg_size_pretty(count(*) * 8192) as buffered
  , round(100.0 * count(*) / ( SELECT setting FROM pg_settings WHERE name='shared_buffers')::integer,1) AS buffers_percent
  , round(100.0 * count(*) * 8192 / pg_relation_size(c.oid),1) AS percent_of_relation
FROM pg_class c
INNER JOIN pg_buffercache b ON b.relfilenode = c.relfilenode
INNER JOIN pg_database d ON (b.reldatabase = d.oid AND d.datname = current_database())
WHERE pg_relation_size(c.oid) > 0
GROUP BY c.oid, c.relname
ORDER BY 3 DESC
LIMIT 10;
```

```sql
SELECT pg_size_pretty(count(*) * 8192) as ideal_shared_buffers
FROM pg_buffercache b
WHERE usagecount >= 3;
```

## Percent to Emergency Vacuum

```sql
SELECT datname
    , age(datfrozenxid)
    , current_setting('autovacuum_freeze_max_age')
    , round((age(datfrozenxid)::numeric / current_setting('autovacuum_freeze_max_age')::numeric)*100) pct_to_emer_vacuum
FROM pg_database
ORDER BY 2 DESC;
```

## Transaction Wraparound

```sql
-- pgMonitor Exporter (ccp_transaction_wraparound)
WITH max_age AS 
       (SELECT 2146483647 as max_old_xid, 
               setting AS autovacuum_freeze_max_age 
        FROM pg_catalog.pg_settings 
        WHERE name = 'autovacuum_freeze_max_age'), 
     per_database_stats AS 
       (SELECT datname, m.max_old_xid::int, m.autovacuum_freeze_max_age::int,
               age(d.datfrozenxid) AS oldest_current_xid 
        FROM pg_catalog.pg_database d 
             JOIN max_age m ON (true) 
        WHERE d.datallowconn)
SELECT max(oldest_current_xid) AS oldest_current_xid, 
      max(ROUND(100*(oldest_current_xid/max_old_xid::float))) AS percent_towards_wraparound,      
      max(ROUND(100*(oldest_current_xid/autovacuum_freeze_max_age::float))) AS percent_towards_emergency_autovac 
FROM per_database_stats;
```
