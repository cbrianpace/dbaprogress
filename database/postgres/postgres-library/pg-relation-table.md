# Table Queries

```sql
SELECT schemaname, relname, seq_scan, idx_scan, 
       n_live_tup, n_dead_tup,
       n_tup_ins, n_tup_upd, n_tup_del, n_tup_hot_upd,
       round(case when n_tup_upd > 0 then (n_tup_hot_upd::numeric/n_tup_upd)*100 else 0 end) percent_hot_upd,
       round(case when n_live_tup > 0 then (n_dead_tup::numeric/n_live_tup)*100 else 0 end) percent_dead_tup,
       round(case when n_live_tup > 0 then (n_tup_hot_upd::numeric/n_live_tup)*100 else 0 end) percent_hot_upd
FROM pg_stat_user_tables
WHERE schemaname='x' 
      AND relname='y';
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
WHERE t.schemaname IN ('public')
ORDER BY 1,2;
```

```sql
SELECT
    pg_namespace.nspname,
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
    pg_class.relkind = 'r'
    AND pg_namespace.nspname not in ('pg_catalog', 'information_schema')
    -- AND pg_namespace.nspname = 'public'
    -- AND pg_class.relname = 'mytable'
GROUP BY pg_namespace.nspname, pg_class.relname, pg_class.reltuples
ORDER BY pg_namespace.nspname, pg_class.relname, pg_class.reltuples;
```
