# Index Queries

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

```sql
-- Index Hit Rate
Select relname, 100* idx_scan / (seq_scan + idx_scan), n_live_tup from pg_stat_user_tables order by n_live_typ desc;
```

## Summary

```sql
SELECT
    pg_class.relname,
   pg_size_pretty(pg_class.reltuples::bigint)            AS size,
    pg_class.reltuples                                    AS num_rows,
    COUNT(*)                                             AS total_indexes,
    COUNT(*) FILTER ( WHERE indisunique)                  AS unique_indexes,
    COUNT(*) FILTER ( WHERE indnatts = 1 )                AS single_column_indexes,
    COUNT(*) FILTER ( WHERE indnatts IS DISTINCT FROM 1 ) AS multi_column_indexes
FROM
    pg_namespace
    LEFT JOIN pg_class ON pg_namespace.oid = pg_class.relnamespace
    LEFT JOIN pg_index ON pg_class.oid = pg_index.indrelid
WHERE
    pg_namespace.nspname = 'public' 
    AND pg_class.relkind = 'r'
    # AND pg_class.relname = 'mytable'
GROUP BY pg_class.relname, pg_class.reltuples
ORDER BY pg_class.reltuples DESC;
```

## Unused

```sql
SELECT *, pg_size_pretty(pg_relation_size(indexrelname))
 FROM pg_stat_all_indexes 
 WHERE schemaname = 'public' 
 ORDER BY pg_relation_size(indexrelname) DESC, idx_scan ASC;
```

## Usage

```sql
SELECT
    t.schemaname,
    t.tablename,
    c.reltuples::bigint                            AS num_rows,
   pg_size_pretty(pg_relation_size(c.oid))        AS table_size,
    psai.indexrelname                              AS index_name,
   pg_size_pretty(pg_relation_size(i.indexrelid)) AS index_size,
    CASE WHEN i.indisunique THEN 'Y' ELSE 'N' END  AS "unique",
    psai.idx_scan                                  AS number_of_scans,
    psai.idx_tup_read                              AS tuples_read,
    psai.idx_tup_fetch                             AS tuples_fetched
FROM
    pg_tables t
    LEFT JOIN pg_class c ON t.tablename = c.relname
    LEFT JOIN pg_index i ON c.oid = i.indrelid
    LEFT JOIN pg_stat_all_indexes psai ON i.indexrelid = psai.indexrelid
WHERE
    t.schemaname NOT IN ('pg_catalog', 'information_schema')
ORDER BY 1, 2;
```

```sql
SELECT
    t.schemaname,
    t.tablename,
    indexname,
    c.reltuples AS num_rows,
    pg_size_pretty(pg_relation_size(quote_ident(t.schemaname)::text || '.' || quote_ident(t.tablename)::text)) AS table_size,
    pg_size_pretty(pg_relation_size(quote_ident(t.schemaname)::text || '.' || quote_ident(indexrelname)::text)) AS index_size,
    CASE WHEN indisunique THEN 'Y'
        ELSE 'N'
    END AS UNIQUE,
    number_of_scans,
    tuples_read,
    tuples_fetched
FROM pg_tables t
LEFT OUTER JOIN pg_class c ON t.tablename = c.relname
LEFT OUTER JOIN (
    SELECT
        c.relname AS ctablename,
        ipg.relname AS indexname,
        x.indnatts AS number_of_columns,
        idx_scan AS number_of_scans,
        idx_tup_read AS tuples_read,
        idx_tup_fetch AS tuples_fetched,
        indexrelname,
        indisunique,
        schemaname
    FROM pg_index x
    JOIN pg_class c ON c.oid = x.indrelid
    JOIN pg_class ipg ON ipg.oid = x.indexrelid
    JOIN pg_stat_all_indexes psai ON x.indexrelid = psai.indexrelid
) AS foo ON t.tablename = foo.ctablename AND t.schemaname = foo.schemaname
WHERE t.schemaname NOT IN ('pg_catalog', 'information_schema')
ORDER BY 1,2;
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
WHERE schemaname='xxxx' and tablename='yyyyyy' 
ORDER BY bloat_ratio DESC;
```
