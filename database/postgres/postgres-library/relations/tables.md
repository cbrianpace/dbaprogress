# Tables

## Information

```sql
SELECT  t.relid, t.schemaname, t.relname,
        pg_table_size(t.relid) as table_size_bytes,
        pg_indexes_size(t.relid) as indexes_size_bytes,
        pg_total_relation_size(relid) - pg_relation_size(relid) - pg_indexes_size(relid) AS toast_size_bytes,
        pg_total_relation_size(t.relid) as total_size_bytes,
        c.relpages,           
        t.n_live_tup, t.n_dead_tup,        
        s.row_size_bytes,
        case when t.n_live_tup > 0 
             then t.n_live_tup / c.relpages
        end as rows_per_page,
        c.reltoastrelid,
        c.relhasrules,
        c.relhastriggers,
        c.relrowsecurity,
        c.relfrozenxid,
        c.relacl,
        c.reloptions,
        c.relispartition,
        pg_relation_filepath(t.schemaname||'.'||t.relname) filepath,
        i.total_indexes,
        i.unique_indexes,
        i.single_column_indexes,
        i.multi_column_indexes,
        age(c.relfrozenxid) AS xid_age,
        current_setting('autovacuum_freeze_max_age')::bigint AS freeze_max_age,
        (age(c.relfrozenxid) * 100.0 / current_setting('autovacuum_freeze_max_age')::bigint) AS percent_to_wraparound
FROM    pg_stat_user_tables t
        JOIN pg_class c ON (c.oid=t.relid)
        left JOIN (SELECT starelid, sum(stawidth) as row_size_bytes 
                   FROM pg_statistic 
                   GROUP BY starelid) s ON (s.starelid = t.relid)
        LEFT JOIN (SELECT indrelid, count(1) as total_indexes, 
                          count(1) FILTER (WHERE indisunique) as unique_indexes, 
                          count(1) FILTER (WHERE indnatts = 1) as single_column_indexes, 
                          count(1) FILTER (WHERE indnatts IS DISTINCT FROM 1) as multi_column_indexes 
                   FROM pg_index 
                   GROUP BY indrelid) i ON c.oid = i.indrelid;
```

## Performance / Usage

### DML Acivity

```sql
SELECT schemaname, relname, seq_scan, idx_scan, 
       n_tup_ins, n_tup_upd, n_tup_del, n_tup_hot_upd,
       round(case when n_tup_upd > 0 then (n_tup_hot_upd::numeric/n_tup_upd)*100 else 0 end) percent_hot_upd,
       round(case when n_live_tup > 0 then (n_dead_tup::numeric/n_live_tup)*100 else 0 end) percent_dead_tup,
       round(case when n_live_tup > 0 then (n_tup_upd+n_tup_hot_upd::numeric/n_live_tup)*100 else 0 end) percent_upd
FROM pg_stat_user_tables;
```

### Cache Hit Ratio by Table

```sql
SELECT  relname, 
        case when sum(idx_blks_hit + idx_blks_read) = 0 
             then 0 
             else round(100 * ((sum(idx_blks_hit) - sum(idx_blks_read)) / sum(idx_blks_hit + idx_blks_read))) 
        end as ratio
FROM    pg_statio_user_indexes
WHERE   idx_blks_hit > 0
GROUP BY relname
ORDER BY ratio;
```

## Table Stats

```sql
SELECT  current_database() as dbname, 
        schemaname, 
        relname, 
        case when coalesce(seq_tup_read,0)+coalesce(idx_tup_fetch,0) > 0
            then round(100.0 * (coalesce(seq_tup_read,0)+coalesce(idx_tup_fetch,0))
                 / (coalesce(n_tup_ins,0)+coalesce(n_tup_upd,0)+coalesce(n_tup_del,0)
                    +coalesce(n_tup_hot_upd,0)+coalesce(seq_tup_read,0)+coalesce(idx_tup_fetch,0))
                 ) 
            else 0
        end as pct_read,
        case when seq_tup_read > 0 
            then round(100.0 * coalesce(idx_tup_fetch,0) / (coalesce(seq_tup_read,0) + coalesce(idx_tup_fetch,0))) 
            else 0 
        end AS pct_index_scan,
        coalesce(seq_scan,0) seq_scan, 
        coalesce(idx_scan,0) idx_scan, 
        seq_tup_read, 
        coalesce(idx_tup_fetch,0) idx_tup_fetch, 
        n_tup_ins, 
        n_tup_upd, 
        n_tup_del, 
        n_tup_hot_upd, 
        n_live_tup, 
        n_dead_tup, 
        vacuum_count, 
        autovacuum_count, 
        analyze_count, 
        autoanalyze_count 
FROM 
    pg_catalog.pg_stat_user_tables;
```

### Large Tables not well cached

```sql
SELECT  c.relname,
        pg_size_pretty(pg_table_size(c.oid)) AS total_size,
        pg_size_pretty(count(*) * 8192) AS cached_size,
        round(100.0 * count(*) * 8192 / nullif(pg_table_size(c.oid), 0), 2) AS pct_cached
FROM    pg_buffercache b
        JOIN pg_class c ON b.relfilenode = pg_relation_filenode(c.oid)
        JOIN pg_database d ON b.reldatabase = d.oid AND d.datname = current_database()
GROUP BY c.relname, c.oid
HAVING  pg_table_size(c.oid) > 10 * 1024 * 1024 -- larger than 10 MB
ORDER BY pct_cached ASC
LIMIT 20;
```

## Perent to Emergency Vacuum

```sql
SELECT  n.nspname AS schema_name,
        c.relname AS table_name,
        age(c.relfrozenxid) AS xid_age,
        current_setting('autovacuum_freeze_max_age')::bigint AS freeze_max_age,
        (age(c.relfrozenxid) * 100.0 / current_setting('autovacuum_freeze_max_age')::bigint) AS percent_to_wraparound
FROM    pg_class c
        JOIN pg_namespace n ON c.relnamespace = n.oid
WHERE   c.relkind = 'r'  -- Only tables
        AND n.nspname NOT IN ('pg_catalog', 'information_schema')  -- Exclude system schemas
        AND age(c.relfrozenxid) > (current_setting('autovacuum_freeze_max_age')::bigint * 0.8)  -- Tables within 80% of wraparound
ORDER BY percent_to_wraparound DESC;
```

## Bloat Check

Note:  Requires Custom Objects Per Database

```sql
SELECT current_database() AS dbname, schemaname, objectname, size_bytes,
      (dead_tuple_size_bytes + (free_space_bytes - (relpages - (fillfactor/100) * relpages ) * current_setting('block_size')::bigint ))::bigint AS total_wasted_space_bytes
FROM bloat_stats;
```
