---
layout: post
read_time: true
show_date: true
title:  The Missing Guide for pgBouncer Metrics
date:   2024-11-22 12:00:00 -0400
description: A step toward understanding and using pgBouncer metrics.
img: posts/banners/lost-1605501_1280.jpg
tags: [postgres, pgbouncer]
author: Brian Pace
category: postgres
---
## Introduction

Several pgBouncer metrics are critically important.  However, there is often a lack of understanding as to what these metrics mean and why they are important.  To empower you with the necessary knowledge, I want to demonstrate these metrics in action.  To help with this, a load generation will create a controlled load on the database (and pgBouncer).  During the test, various SHOW commands will display useful metrics.  We will dig into these metrics to understand them better.

## Metric Overview

The metrics below show the starting point.  At this point, pgBouncer service is running and no connections exists.

```sql
pgbouncer=# show mem;
     name     | size | used | free | memtotal
--------------+------+------+------+----------
 user_cache   | 1264 |    3 |   47 |    63200
 db_cache     |  208 |    1 |   77 |    16224
 pool_cache   |  480 |    1 |   49 |    24000
 server_cache |  560 |    0 |    0 |        0
 client_cache |  560 |    1 |   49 |    28000
 iobuf_cache  | 4112 |    1 |   49 |   205600
(6 rows)

pgbouncer=# show lists;
     list      | items
---------------+-------
 databases     |     1
 users         |     2
 pools         |     1
 free_clients  |    49
 used_clients  |     1
 login_clients |     0
 free_servers  |     0
 used_servers  |     0
 dns_names     |     0
 dns_zones     |     0
 dns_queries   |     0
 dns_pending   |     0
(12 rows)

pgbouncer=# show pools;
 database  |   user    | cl_active | cl_waiting | cl_cancel_req | sv_active | sv_idle | sv_used | sv_tested | sv_login | maxwait | maxwait_us | pool_mode
-----------+-----------+-----------+------------+---------------+-----------+---------+---------+-----------+----------+---------+------------+-----------
 pgbouncer | pgbouncer |         1 |          0 |             0 |         0 |       0 |       0 |         0 |        0 |       0 |          0 | statement
(1 row)
```

### SHOW MEM

The SHOW MEM command displays informal about internal memory structures used by pgBouncer.  Each cache represent memory allocated to track related details.  For example, the client_cache tracks information for incoming connections to pgBouncer.  The server_cache tracks information about connections to the Postgres database.  When a client submits a SQL statement, a free server performs the action in the database.  The client_cache and server_cache record information about this association.  This association is also visible in the SHOW CLIENTS command output.

At our starting point, server_cache is 0 for both used and free as no database connections exists.  The client_cache shows 1 used and 49 free.  The one connection used is for our pgbouncer admin console connection.  The two metrics for the server_cache and client_cache do not represent hard limits.  Instead, the caches will increase as needed in sets of 50.

### SHOW LISTS

The SHOW LIST command has some obvious metrics and some not so obvious metrics.  First the obvious.  The 'databases' metric shows the number of databases that currently have a connection.  The important note here is that 'databases' is from pgBouncer perspective.  For example, in the pgBouncer configuration there are two defined connections (mydb_rw and mydb_ro).  Both of these point to the same database in Postgres (mydb).   With a connection made to each, the 'databases' metric will show 2 databases. 

The 'pools' metric shows the total number of active connection pools in pgBouncer.  Remember that a pool is a set of connections based on a unique combination of database name and user name.  At our starting point, only 1 pool exists. That pool is for the connection to pgBouncer itself to execute the SHOW commands.

The 'x_clients' metrics are tracking incoming client connections (cache used).  To understand 'free_clients' and 'used_clients' think back to the SHOW MEM command and the client_cache metric.  Of the current client_cache allocated size, 'free_clients' represents how many are available for new connections.  The 'used_clients' reflects the number of caches used by established clients.  This is not relative to any hard limit as indicated earlier but to the current memory allocation.  The 'login_clients' metric reflects the number of clients in the authentication process to the Postgres database.

The 'free_servers', the 'used_servers' metrics show the usage of the  server_cache.  The 'free_servers' indicates the number of free cache entries available.  The 'used_servers' represents the number of used entries in the cache.  These are not related to any hard limit like max_connections.  Instead they are  relative to the current allocated size of the server_cache.

### SHOW POOLS

The metrics of most concern come from the SHOW POOLS command.  In the above output, there is only one pool that is active and that is for our connection to pgBouncer itself.

Below is a description for each of the metrics and why it is important:

**database & user:**
    Remember that pgBouncer creates pools based database and user.  In our opening example there is one pool for the database pgbouncer that is connected to via the pgbouncer user.  The database is the name as defined in the databases section or databases.ini for pgBouncer and not the actual database.

**cl_active:**
    Number of active connections to pgBouncer from the applications or users.  Active here does not mean that the connection is doing anything just that a connection has been made to pgBouncer.

**cl_waiting:**
    A connection has issued a SQL statement that is waiting to be assigned to a free server.
    Small spikes here is normal.  Small being less than 10 or any number if the pool is newly created.  What is an indication of an issue is if the number is sustained over several samples.  This indicates there are not enough servers (connections to the actual database) to satisfy the demand.

**sv_active:**
    Number of server connections (connections to the actual database) that are actively assigned to a client connection to perform database work.

**sv_idle:**
    Number of server connections that have not been used for a period of time.  By default, pgBouncer does not use all of the servers in a round robin.  This is due to performance reasons.  Instead the first connection is always used first if it is available.  If not, then the second.  If the second is not available, then the third, etc.  Therefore servers that are at the end of that list may not be utilized and will fall into the idle state.

**sv_used:**
    These are server connections that have been idle for so long that the keep alive command needs to be executed to ensure they are still valid connections.
    Sustained values here may indicate an oversized pool or a pool that bloats during peak times and then goes idle.

**sv_login:**
    Server connection that is in the authentication phase with the database.

**maxwait_us:**
    In microseconds, the longest a client that shows up under the cl_waiting metric has been waiting for a connection.
    From an application perspective, this is the amount of time that has been added to the perceived response time of the submitted query.  For example, if there is a SQL statement that took 1ms in the database but the client waited 14ms for an available connection, then to the application the response time from the database is 15ms.

## Metric at Work

### pgBouncer Settings

To start the test the following key parameters have been configured:

- pool_mode = transaction
- default_pool_size = 10
- min_pool_size = 5

The Java application generating the load has JVM based connection pool of 30 configured.  The min/max are the same to avoid the Java connection pool from connecting and disconnecting.

### Start of 10 User Load

After starting the application, executing SHOW POOLS a few times to see how the pools is performing.  Below is the output.

```sql
pgbouncer=# show pools;
 database  |   user    | cl_active | cl_waiting | cl_cancel_req | sv_active | sv_idle | sv_used | sv_tested | sv_login | maxwait | maxwait_us |  pool_mode
-----------+-----------+-----------+------------+---------------+-----------+---------+---------+-----------+----------+---------+------------+-------------
 hippo     | postgres  |        21 |          9 |             0 |         1 |       0 |       0 |         0 |        0 |       0 |      19942 | transaction
 
pgbouncer=# show pools;
 database  |   user    | cl_active | cl_waiting | cl_cancel_req | sv_active | sv_idle | sv_used | sv_tested | sv_login | maxwait | maxwait_us |  pool_mode
-----------+-----------+-----------+------------+---------------+-----------+---------+---------+-----------+----------+---------+------------+-------------
 hippo     | postgres  |        23 |          7 |             0 |         2 |       0 |       0 |         0 |        0 |       0 |       5145 | transaction

pgbouncer=# show pools;
 database  |   user    | cl_active | cl_waiting | cl_cancel_req | sv_active | sv_idle | sv_used | sv_tested | sv_login | maxwait | maxwait_us |  pool_mode
-----------+-----------+-----------+------------+---------------+-----------+---------+---------+-----------+----------+---------+------------+-------------
 hippo     | postgres  |        28 |          2 |             0 |         6 |       0 |       0 |         0 |        0 |       0 |        819 | transaction

  database  |   user    | cl_active | cl_waiting | cl_cancel_req | sv_active | sv_idle | sv_used | sv_tested | sv_login | maxwait | maxwait_us |  pool_mode
-----------+-----------+-----------+------------+---------------+-----------+---------+---------+-----------+----------+---------+------------+-------------
 hippo     | postgres  |        30 |          0 |             0 |         9 |       1 |       0 |         0 |        0 |       0 |          0 | transaction
```

At first there are several client connections waiting (cl_waiting = 9) and only 1 database connection (sv_active=1).  The initial wait event indicates that the oldest client in the queue has been waiting 19ms (19.943 microseconds).  At first this is very bad until we understand what is taking place.  As demand is sustained, pgBouncer increases the size of the pool.  The last sample shows there are 10 total connections to the database (sv_idle + sv_active) with zero waiting.

![Application Start](./assets/img/posts/reference/app-start-responsetime.png)

With our simulated load of 10 active users, the pgBouncer pool is now adequate.  Not a lot of idle servers and very little queuing.  The application reports the starting response time around 20ms.  That reduced to an average of 2ms as the pool grew to the needed size.

### Increase Load to 20 Concurrent Users

The below output from SHOW POOLS shows the pool is now back to 0 clients connected (cl_active).  Restarting the load generation tool with 20 concurrent users, the pool again went through that ramp up period as we saw in our initial load.  Once all 10 servers are started  (as defined by default_pool_size set to 10), there is still see a sustained number of clients waiting (cl_waiting = 8).  The sv_active metric shows all servers are busy.  The longest client in the queue has been waiting for 3.4ms (3,441 microseconds).  This sustained queuing indicates the pool is not sized properly for the workload.

The min_pool_size is set to 5.  This instructs pgBouncer to maintian a minimal of 5 connections/servers.  The cl_active metric shows 0 clients connected with 5 idle connections/servers (sv_idle).  This is helpful as it can help the reduce the 'ramp up' period when the application restarts.

```sql
pgbouncer=# show pools;
 database  |   user    | cl_active | cl_waiting | cl_cancel_req | sv_active | sv_idle | sv_used | sv_tested | sv_login | maxwait | maxwait_us |  pool_mode
-----------+-----------+-----------+------------+---------------+-----------+---------+---------+-----------+----------+---------+------------+-------------
 hippo     | postgres  |         0 |          0 |             0 |         0 |       5 |       0 |         0 |        0 |       0 |          0 | transaction

pgbouncer=# show pools;
 database  |   user    | cl_active | cl_waiting | cl_cancel_req | sv_active | sv_idle | sv_used | sv_tested | sv_login | maxwait | maxwait_us |  pool_mode
-----------+-----------+-----------+------------+---------------+-----------+---------+---------+-----------+----------+---------+------------+-------------
 hippo     | pgbouncer |         0 |          0 |             0 |         0 |       0 |       1 |         0 |        0 |       0 |          0 | transaction
 hippo     | postgres  |        22 |          8 |             0 |        10 |       0 |       0 |         0 |        0 |       0 |       3441 | transaction
```

After increasing the pool size (default_pool_size) to 20 and executing a RELOAD,  the queuing has stopped.  The connection pool increased servers to 20 while processing the backlog of work.  Now there are 3 connections/servers that are idle (sv_idle).  The connection is properly sized for the workload.

```sql
pgbouncer=# reload;
RELOAD

pgbouncer=# show pools;
 database  |   user    | cl_active | cl_waiting | cl_cancel_req | sv_active | sv_idle | sv_used | sv_tested | sv_login | maxwait | maxwait_us |  pool_mode
-----------+-----------+-----------+------------+---------------+-----------+---------+---------+-----------+----------+---------+------------+-------------
 hippo     | postgres  |        24 |          6 |             0 |        11 |       0 |       0 |         0 |        0 |       0 |       1549 | transaction

pgbouncer=# show pools;
 database  |   user    | cl_active | cl_waiting | cl_cancel_req | sv_active | sv_idle | sv_used | sv_tested | sv_login | maxwait | maxwait_us |  pool_mode
-----------+-----------+-----------+------------+---------------+-----------+---------+---------+-----------+----------+---------+------------+-------------
 hippo     | pgbouncer |         0 |          0 |             0 |         0 |       0 |       1 |         0 |        0 |       0 |          0 | transaction
 hippo     | postgres  |        30 |          0 |             0 |        17 |       3 |       0 |         0 |        0 |       0 |          0 | transaction
```

Viewing the chart below, you can see the initial ramp up period where response time was almost 100ms.  As the pool ramped up the response time dropped to 7ms and stayed at 7ms until we resized the pool to 20 database connections.  With the pool properly sized response time returned to the expected 2ms.

![Application Rampup](./assets/img/posts/reference/app-20concurrent-users-responsetime.png)

## Conclusion

Using pgBouncer has many advantages.  If not properly configured it can also cause many challenges.  With this new understanding of key metrics you can now ensure that pgBouncer is ready for your most demanding workloads.
