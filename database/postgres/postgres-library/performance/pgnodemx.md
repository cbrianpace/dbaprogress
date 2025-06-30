# pgNodeMX Queries

## CPU

```sql
SELECT monitor.kdapi_scalar_bigint('cpu_request') as request, monitor.kdapi_scalar_bigint('cpu_limit') as limit;
```

```sql
SELECT monitor.cgroup_scalar_bigint('cpu.cfs_period_us') as period_us, 
        case 
          when monitor.cgroup_scalar_bigint('cpu.cfs_quota_us') < 0 then 0 
           else monitor.cgroup_scalar_bigint('cpu.cfs_quota_us') 
        end as quota_us;
```

```sql
SELECT monitor.cgroup_scalar_bigint('cpuacct.usage') as usage, clock_timestamp() as usage_ts;
```

```sql
WITH d(key, val) AS
        (SELECT key, val FROM monitor.cgroup_setof_kv('cpu.stat'))
SELECT
    (SELECT val FROM d WHERE key='nr_periods') AS nr_periods,
    (SELECT val FROM d WHERE key='nr_throttled') AS nr_throttled,
    (SELECT val FROM d WHERE key='throttled_time') AS throttled_time, clock_timestamp() as snap_ts;
```

## Disk Activity

```sql
SELECT 
    mount_point,sectors_read,sectors_written
FROM 
    monitor.proc_mountinfo() m
    JOIN monitor.proc_diskstats() d USING (major_number, minor_number)
WHERE 
    m.mount_point IN ('/pgdata', '/pgwal') OR m.mount_point like '/tablespaces/%';  
```

## Disk Utilization

```sql
SELECT 
   mount_point, round(((total_bytes-available_bytes)/total_bytes)*100) pct_used,
   fs_type, total_bytes, available_bytes, total_file_nodes, 
   free_file_nodes
FROM 
    monitor.proc_mountinfo() m
    JOIN monitor.fsinfo(m.mount_point) f USING (major_number, minor_number)
WHERE 
    m.mount_point IN ('/pgdata', '/pgwal') 
    OR m.mount_point like '/tablespaces/%';  
```

## Memory

```sql
WITH d(key, val) as
        (SELECT key, val 
         FROM monitor.cgroup_setof_kv('memory.stat'))
SELECT
    monitor.kdapi_scalar_bigint('mem_request') as request,
    case 
      when monitor.cgroup_scalar_bigint('memory.limit_in_bytes') = 9223372036854771712 then 0 
      else monitor.cgroup_scalar_bigint('memory.limit_in_bytes') 
    end as limit,
   (SELECT val FROM d WHERE key='cache') as cache,
   (SELECT val FROM d WHERE key='rss') as rss,
   (SELECT val FROM d WHERE key='shmem') as shmem,
   (SELECT val FROM d WHERE key='mapped_file') as mapped_file,
   (SELECT val FROM d WHERE key='dirty') as dirty,
   (SELECT val FROM d WHERE key='active_anon') as active_anon,
   (SELECT val FROM d WHERE key='inactive_anon') as inactive_anon,
   (SELECT val FROM d WHERE key='active_file') as active_file,
   (SELECT val FROM d WHERE key='inactive_file') as inactive_file,
  monitor.cgroup_scalar_bigint('memory.usage_in_bytes') as usage_in_bytes,
  monitor.cgroup_scalar_bigint('memory.kmem.usage_in_bytes') as kmem_usage_in_bytes;
```

## Network

```sql
SELECT 
    interface,tx_bytes,tx_packets, rx_bytes,rx_packets 
FROM
    monitor.proc_network_stats();
```

## Process Count

```sql
SELECT monitor.cgroup_process_count() as count;
```
