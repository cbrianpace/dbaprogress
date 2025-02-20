# Postgres Library

## Functions

### Current WAL LSN

```sql
SELECT pg_current_wal_lsn();
```

### Switch WAL

```sql
SELECT pg_switch_wal();
```

### Reload configuration

```sql
SELECT pg_reload_conf();
```

## generate_series

### Description

Generates a series from start to stop.  Step can be used to control the direction and increments of the series (positive counts up, negative counts down).

### Syntax

```sql
generate_series ( start, stop [, step ] ) 
```

**Example**

```shell
postgres=# SELECT * FROM generate_series(1,10);

  generate_series
-----------------
               1
               2
               3
               4
               5
               6
               7
               8
               9
              10
(10 rows)
```

### Use Cases

Generate series is used a lot to populate a data set with a fixed number of rows.  In the example below, it is used in a forecasting query to project out performance metrics for 1 year in 30 day increments.

```sql
INSERT INTO target_metric_forecast (tid, metric_category, metric_name, metric_key, avg_val, day_count, forecast_dt)
     (SELECT tid, cm.metric_category, cm.metric_name, cm.metric_key,
             ROUND( (t.slope * extract(julian from t.max_timestamp + make_interval(days=>r.n)) + t.intercept)::numeric,2) forecast_avg,
              r.n daycount, t.max_timestamp + make_interval(days=>r.n) forecast_date
        FROM (SELECT s.daynbr n
              FROM generate_series(1,360) s (daynbr)
              ORDER BY s.daynbr) r,
              (SELECT tid, regr_slope(avg_val,jdt) slope, regr_intercept(avg_val,jdt) intercept,
                      count(1) cnt, max(rollup_dt) max_timestamp
               FROM (SELECT o.tid, d.rollup_dt, extract(julian from d.rollup_dt) jdt, d.avg_val
                     FROM  target o
                           JOIN target_metric_summary_day d on (o.tid = d.tid)
                           JOIN (SELECT tid, metric_category, metric_name, 
                                         min(rollup_dt) RollupDateMin, max(rollup_dt) RollupDateMax
                                 FROM target_metric_summary_day                             
                                 GROUP BY tid, metric_category, metric_name) as r on (d.tid=r.tid AND 
                                       d.metric_category = r.metric_category and d.metric_name = r.metric_name)
                             WHERE  o.tid = vtid               
                                    AND d.metric_category = cm.metric_category
                                    AND d.metric_name  = cm.metric_name
                                    AND coalesce(d.metric_key,'null') = coalesce(cm.metric_key,'null')
                                    AND d.rollup_dt >= case when date_trunc('day',current_timestamp - make_interval(days=>vlookbackdays)) < r.RollupDateMin then r.RollupDateMin else date_trunc('day', current_timestamp-make_interval(days=>90)) end
                      ORDER BY o.tid, d.rollup_dt, extract(julian from d.rollup_dt)) as tt
                GROUP BY tid) t 
         WHERE  t.max_timestamp + make_interval(days=>r.n) > date_trunc('day', current_timestamp) 
                 AND mod(r.n,30)=0        
      ); 
      ```
      