# IO

## Background Writer

The following is for versions older than postgres 17.

```sql
SELECT  checkpoints_timed, 
        checkpoints_req, 
        checkpoint_write_time, 
        checkpoint_sync_time, 
        buffers_checkpoint, 
        buffers_clean, 
        maxwritten_clean, 
        buffers_backend, 
        buffers_backend_fsync, 
        buffers_alloc, 
        stats_reset 
FROM    pg_catalog.pg_stat_bgwriter;
```

## Shared Buffer Utilization

The following requires extension pg_buffercache.  The output will show what objects are consuming shared_buffers and how many blocks are dirty.

```sql
SELECT  c.relkind AS type,
        c.relname AS relation,
        pg_size_pretty(count(*) * 8192) AS buffered,
        round(100.0 * count(*) / total_buffers.buffers, 2) AS percent_of_cache,
        round(100.0 * (count(*) * 8192)::numeric / pg_table_size(c.oid), 2) AS percent_of_relation,
        sum(case when isdirty then 1 else 0 end) dirty,
        sum(case when isdirty then 1 else 0 end)*8192 as dirty_size_bytes    
FROM    pg_buffercache b
        JOIN pg_class c ON b.relfilenode = pg_relation_filenode(c.oid)
        JOIN pg_database d ON b.reldatabase = d.oid AND d.datname = current_database(),
        (SELECT count(*) AS buffers FROM pg_buffercache) AS total_buffers
GROUP BY c.relkind, c.relname, c.oid, total_buffers.buffers
ORDER BY count(*) DESC
LIMIT 20;
```

## Database Stats

```sql
SELECT  s.datname as dbname, 
        blks_read, 
        blks_hit, 
        case when blks_hit=0 then 0 else round(((blks_hit+blks_read)/blks_hit)*100) end as cache_hit_ratio,
        tup_returned, 
        tup_fetched, 
        tup_inserted, 
        tup_updated, 
        tup_deleted, 
        case when tup_fetched+tup_returned=0 then 0 else round(((tup_inserted+tup_updated+tup_deleted+tup_fetched+tup_returned)/(tup_fetched+tup_returned))*100) end as read_ratio,
        round(100.0 * tup_fetched / nullif(tup_fetched + tup_returned,0), 2) AS pct_index_scan,
        round(100.0 * tup_returned / nullif(tup_fetched + tup_returned,0), 2) AS pct_seq_or_indexonly_scan,
        temp_files, 
        temp_bytes
FROM    pg_catalog.pg_stat_database s 
        JOIN pg_catalog.pg_database d on d.datname = s.datname 
WHERE   d.datistemplate = false;
```

## Index Stats

### Cache Hit Ratio Summary

```sql
SELECT  round(100 * ((sum(idx_blks_hit) - sum(idx_blks_read)) / sum(idx_blks_hit + idx_blks_read))) as ratio
FROM    pg_statio_user_indexes;
```
