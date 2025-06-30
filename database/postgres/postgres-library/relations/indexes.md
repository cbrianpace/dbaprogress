# Indexes

## Information

```sql
SELECT  t.schemaname,
        t.tablename,
        psai.indexrelname as index_name,
        c.reltuples::bigint as num_rows,
        pg_relation_size(c.oid) as table_size_bytes,
        pg_relation_size(i.indexrelid) as index_size_bytes,
        CASE WHEN i.indisunique THEN true ELSE false END  as isunique,
        psai.last_idx_scan as last_idx_scan,
        current_timestamp - psai.last_idx_scan as time_since_last_use,
        psai.idx_scan as number_of_scans,
        psai.idx_tup_read as tuples_read,
        psai.idx_tup_fetch as tuples_fetched
FROM    pg_tables t
        LEFT JOIN pg_class c ON t.tablename = c.relname
        LEFT JOIN pg_index i ON c.oid = i.indrelid
        LEFT JOIN pg_stat_all_indexes psai ON i.indexrelid = psai.indexrelid
WHERE   t.schemaname NOT IN ('pg_catalog', 'information_schema')
ORDER BY 1, 2;
```

## Creation Progress Monitoring

```sql
select t.relname, index_relid, command, phase, blocks_done, blocks_total, 
       case when blocks_total > 0 then round(100*(blocks_done::numeric/blocks_total::numeric)) else 0 end pct_done 
from pg_class t
     join pg_catalog.pg_stat_progress_create_index i on (t.oid = i.relid) ;
```

## Duplicate

```sql
SELECT pg_size_pretty(sum(pg_relation_size(idx))::bigint) as size,
       (array_agg(idx))[1] as idx1, (array_agg(idx))[2] as idx2,
       (array_agg(idx))[3] as idx3, (array_agg(idx))[4] as idx4
FROM (
    SELECT indexrelid::regclass as idx, (indrelid::text ||E'\n'|| indclass::text ||E'\n'|| indkey::text ||E'\n'||
                                        coalesce(indexprs::text,'')||E'\n' || coalesce(indpred::text,'')) as key
    FROM pg_index) sub
GROUP BY key HAVING count(*)>1
ORDER BY sum(pg_relation_size(idx)) DESC;
```

## Size

```sql
SELECT pg_size_pretty (pg_indexes_size('<table name>'));
```

## Stats

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

## Unused

```sql
SELECT  i.relid, i.schemaname, i.relname, i.indexrelname, pg_relation_size(i.relid) index_size_bytes
FROM    pg_stat_user_indexes i
WHERE   coalesce(idx_scan,0) = 0
ORDER BY index_size_bytes DESC;
```

## Blot

```sql
WITH index_bloat AS (
    SELECT
        schemaname,
        tablename,
        indexname,
        relpages,
        reltuples,
        pg_relation_size(quote_ident(schemaname) || '.' || quote_ident(indexname)) AS index_size,
        CASE 
            WHEN relpages > 0 THEN (relpages - reltuples / NULLIF((bs / 1024), 0)) * bs
            ELSE 0
        END AS bloat_size,
        CASE 
            WHEN relpages > 0 THEN 
                100 * (relpages - reltuples / NULLIF((bs / 1024), 0)) / NULLIF(relpages, 0)
            ELSE 0
        END AS bloat_ratio
    FROM (
        SELECT
            c.oid,
            n.nspname AS schemaname,
            c.relname AS tablename,
            i.relname AS indexname,
            i.reltuples,
            i.relpages,
            current_setting('block_size')::int AS bs
        FROM pg_class c
        JOIN pg_namespace n ON n.oid = c.relnamespace
        JOIN pg_index ix ON ix.indrelid = c.oid
        JOIN pg_class i ON i.oid = ix.indexrelid
        WHERE c.relkind = 'r'
    ) sub
)
SELECT * FROM index_bloat
--WHERE schemaname='public' and tablename='yyyyyy' 
ORDER BY bloat_ratio DESC;
```
