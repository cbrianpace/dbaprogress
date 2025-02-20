---
layout: post
read_time: true
show_date: true
title:  Recovering Data from a Split-Brain Incident
date:   2023-11-01 12:00:00 -0400
description: Operating with asynchronous replication can introduce data loss risks in particular failover scenarios. In this article, we will investigate the indicators that may point to a split-brain incident and delve into potential measures for investigation and recovery.
img: posts/banners/oranges_640.jpg
tags: [postgres, replication]
author: Brian Pace
---

## Introduction

The best practice when deploying Patroni alongside PostgreSQL, and potentially for standalone PostgreSQL setups with replicas, strongly advocates the utilization of synchronous replication. Synchronous replication stands as the most robust method for minimizing data loss during unexpected outages and failover events. However, there may be situations where enabling synchronous replication is not practical or where it was initially configured with an asynchronous approach. Irrespective of the situation, operating with asynchronous replication can introduce data loss risks in particular failover scenarios. In this article, we will investigate the indicators that may point to a split-brain incident and delve into potential measures for investigation and recovery.

## Environment Setup

The setup involves a Postgres High Availability (HA) Cluster comprising two nodes and is under the management of Patroni. The current timeline in use is timeline 3, and the following SQL commands have been executed to establish this environment.

```shell
$ patronictl list

+ Cluster: rhino (7280616456049559375) -------+----+-----------+
| Member     | Host       | Role    | State   | TL | Lag in MB |
+------------+------------+---------+---------+----+-----------+
| orlhippo01 | orlhippo01 | Leader  | running |  3 |           |
| orlhippo02 | orlhippo02 | Replica | running |  3 |         0 |
+------------+------------+---------+---------+----+-----------+

psql <<EOF
CREATE SEQUENCE emp_eid_seq
START 100
INCREMENT 1;

CREATE TABLE emp (eid int NOT NULL DEFAULT nextval('emp_eid_seq') primary key,
first_name varchar(40),
last_name varchar(40),
last_update timestamp);

INSERT INTO emp (FIRST_NAME,LAST_NAME,LAST_UPDATE) 
VALUES ('Mark', 'Smith', current_timestamp),
       ('John', 'Smith', current_timestamp),
       ('Jan', 'Doe', current_timestamp),
       ('Kathy', 'Doe', current_timestamp);

select pg_switch_wal();

EOF
```

The emp table now contains 4 rows as shown below.

```sql
postgres=# select * from emp;
 eid | first_name | last_name |        last_update
-----+------------+-----------+----------------------------
 100 | Mark       | Smith     | 2023-10-04 10:45:46.469488
 101 | John       | Smith     | 2023-10-04 10:45:46.469488
 102 | Jan        | Doe       | 2023-10-04 10:45:46.469488
 103 | Kathy      | Doe       | 2023-10-04 10:45:46.469488
(4 rows)
```

## Manually Creating a Split-Brain Incident

To illustrate this in a way that can be easily replicated, we will follow a manual process to deliberately create a split-brain incident. First, we will place Patroni in maintenance mode by using the command `patronictl pause rhino`. Following that, we will promote the replica instance with the command `psql -c "select pg_promote()"`. These commands will be executed in the specified sequence below. Once both of these commands are executed, two instances will be open for read/write activity, resulting in a split-brain scenario. Finally, to intentionally introduce data divergence, we will execute some SQL statements on both nodes.

Execute on Primary (orlhippo01):

```sql
INSERT INTO emp (FIRST_NAME,LAST_NAME,LAST_UPDATE) 
VALUES ('Matt', 'Jones', current_timestamp);
```

To activate Patroni's maintenance mode and promote the replica, you should run the following commands on the replica host (orlhippo02):

```shell
patronictl pause rhino
psql -c "select pg_promote()"
```

Execute on Original Primary (orlhippo01):

```sql
delete from emp where eid=100;
update emp set first_name='Jill' where eid=103;
```

Execute on the New Primary (orlhippo02):

```sql
delete from emp where eid=103;
INSERT INTO emp (FIRST_NAME,LAST_NAME,LAST_UPDATE) 
VALUES ('Brian', 'Pace', current_timestamp);
update emp set first_name='John' where eid=103;
```

With the steps outlined above completed, we currently have two leaders with data in the 'emp' table that are not synchronized. Prior to deactivating Patroni's maintenance mode, it's imperative to disable Patroni's utilization of pg_rewind. This setting is managed by the 'use_pg_rewind' parameter found in both the 'bootstrap' and 'postgres' sections of the Patroni configuration file(s). Execute the following commands to exit Patroni's maintenance mode:

```shell
patronictl resume rhino
```

## Symptoms

Usually, the investigative process initiates following an alert or notification regarding replication lag or the unavailability of the replica. The displayed output from Patroni represents a common symptom that serves as an alert for a problem potentially linked to a split-brain scenario. In the instance below, the former primary server (orlhippo01) is attempting to restart as a replica but encounters failure.

```shell
$ patronictl list
+ Cluster: rhino (7280616456049559375) ------------+----+-----------+
| Member     | Host       | Role    | State        | TL | Lag in MB |
+------------+------------+---------+--------------+----+-----------+
| orlhippo01 | orlhippo01 | Replica | start failed |    |   unknown |
| orlhippo02 | orlhippo02 | Leader  | running      |  4 |           |
+------------+------------+---------+--------------+----+-----------+
```

Another symptom emerges directly from the PostgreSQL logs, and in the case of the error message provided below, it strongly suggests the likelihood of a split-brain incident. The crucial piece of information here is the comparison between the most recent checkpoint (LSN 0/2F000028) and the point at which the current leader branched off to create the new timeline (LSN 0/2E0002A8). As per this message, it appears that the old primary server produced nearly 16MB of Write-Ahead Log (WAL) data before it was shut down.

```shell
$ cat $PGDATA/log/postgres-Wed.log
...
2023-10-04 11:07:42.203 EDT [2559286]: [5-1] user=,db=,client= FATAL:  requested timeline 4 is not a child of this server's history
2023-10-04 11:07:42.203 EDT [2559286]: [6-1] user=,db=,client= DETAIL:  Latest checkpoint is at 0/2F000028 on timeline 3, but in the history of the requested timeline, the server forked off from that timeline at 0/2E0002A8.
2023-10-04 11:07:42.204 EDT [2559281]: [8-1] user=,db=,client= LOG:  startup process (PID 2559286) exited with exit code 1
2023-10-04 11:07:42.204 EDT [2559281]: [9-1] user=,db=,client= LOG:  aborting startup due to startup process failure
2023-10-04 11:07:42.205 EDT [2559281]: [10-1] user=,db=,client= LOG:  database system is shut down
...
```

By utilizing the LSNs extracted from the PostgreSQL log, you can execute the following query to determine the amount of Write-Ahead Log (WAL) generation that transpired between the promotion of the new leader and the shutdown of the previous one. It's important to note that this calculation assumes that each WAL segment was entirely filled, although this may or may not be the case:

```sql
postgres=# select '0/2F000028'::pg_lsn-'0/2E0002A8'::pg_lsn bytes;
  bytes
----------
 16776576
(1 row)
```

## Investigation

As the investigation commences, one critical piece of evidence stands out—the Log Sequence Number (LSN) at which the new timeline (4) was created. This LSN, as extracted from the PostgreSQL log, is '0/2E0002A8.' Consequently, any changes recorded after this point in the Write-Ahead Log (WAL) segment(s) are absent from the new primary server (orlhippo02).

In the event this log entry was not accessible, we could retrieve the same data by examining the '00000004.history' file on the new primary, as illustrated below. This LSN, which we can refer to as the "failover LSN," will serve as a filtering criterion in the subsequent steps of our investigation.

```shell
$ cat 00000004.history
1 0/2C0000A0  no recovery target specified
2 0/2D0000A0  no recovery target specified
3 0/2E0002A8  no recovery target specified
```

### Starting Original Primary as Standby

The former primary server is presently in a failed state, with Patroni repeatedly attempting to restore it. In the course of our investigation, it is necessary to have the database accessible in read-only mode (standby mode). By initiating the database in standby mode, we can prevent the generation of an additional timeline and avoid tasks such as autovacuum related clean-up activities. To initiate this process, we will first place Patroni in maintenance mode and subsequently shut down the Patroni service.

Execute on Original Primary (orlhippo01):

```shell
$ patronictl pause rhino
Success: cluster management is paused

$ sudo systemctl stop patroni@rhino
```

Prior to manually starting the database, several adjustments must be applied to the 'postgresql.conf' file:

- Commented out primary_conninfo
- Set archive_command='true'
- Set recovery_target_timeline='3'

After adjusting the parameters, the following command is employed to start PostgreSQL as a standby server. It's important to note that the 'standby.signal' file should already be present in the 'PGDATA' directory for this process to proceed:

```shell
pg_ctl start -D /app/pgdata/rhino.15
```

With the old primary now open, an easier solution may be to leverage a data compare solution as documented in this blog post.  However, as an education exercise let's continue with our example.

### Identify Recorded Changes in WAL

Recall that during the symptom data collection phase, we mentioned the possibility of a split-brain incident. Depending on how PostgreSQL is shut down, some of the data written to the Write-Ahead Log (WAL) segment might consist mainly of metadata related to checkpoints and other administrative information. To discern the actual content within these WAL segments, we execute pg_waldump with the 'stats' option. This action provides a summarized overview of the contents and helps us determine whether further investigation is warranted.

```shell
$ pg_waldump  --start=0/2E0002A8 --timeline=3 --stats=record
WAL statistics between 0/2E0002A8 and 0/2F0000A0:
Type                                           N      (%)          Record size      (%)             FPI size      (%)        Combined size      (%)
----                                           -      ---          -----------      ---             --------      ---        -------------      ---
XLOG/CHECKPOINT_SHUTDOWN                       1 ( 14.29)                  114 ( 29.23)                    0 (  0.00)                  114 ( 29.23)
XLOG/SWITCH                                    1 ( 14.29)                   24 (  6.15)                    0 (  0.00)                   24 (  6.15)
Transaction/COMMIT                             2 ( 28.57)                   68 ( 17.44)                    0 (  0.00)                   68 ( 17.44)
Standby/RUNNING_XACTS                          1 ( 14.29)                   50 ( 12.82)                    0 (  0.00)                   50 ( 12.82)
Heap/DELETE                                    1 ( 14.29)                   54 ( 13.85)                    0 (  0.00)                   54 ( 13.85)
Heap/HOT_UPDATE                                1 ( 14.29)                   80 ( 20.51)                    0 (  0.00)                   80 ( 20.51)
                                        --------                      --------                      --------                      --------
Total                                          7                           390 [100.00%]                   0 [0.00%]                   390 [100%]
```

The information provided above suggests the presence of two committed transactions (Transaction/COMMIT) within the records. Additionally, it includes one DELETE operation (Heap/DELETE) and one UPDATE operation (Heap/HOT_UPDATE). However, the available data does not offer sufficient insight to determine whether the DELETE and UPDATE actions originated from the same transaction. Therefore, to gain a more comprehensive understanding of these activities, further investigation and utilization of pg_waldump will be necessary to gather additional details.

```shell
$ pg_waldump  --start=0/2E0002A8 --timeline=3
rmgr: Heap        len (rec/tot):     54/    54, tx:        858, lsn: 0/2E0002A8, prev 0/2E000270, desc: DELETE off 1 flags 0x00 KEYS_UPDATED , blkref #0: rel 1663/5/16907 blk 0
rmgr: Transaction len (rec/tot):     34/    34, tx:        858, lsn: 0/2E0002E0, prev 0/2E0002A8, desc: COMMIT 2023-10-04 10:49:31.092400 EDT
rmgr: Heap        len (rec/tot):     80/    80, tx:        859, lsn: 0/2E000308, prev 0/2E0002E0, desc: HOT_UPDATE off 4 xmax 859 flags 0x60 ; new off 6 xmax 0, blkref #0: rel 1663/5/16907 blk 0
rmgr: Transaction len (rec/tot):     34/    34, tx:        859, lsn: 0/2E000358, prev 0/2E000308, desc: COMMIT 2023-10-04 10:49:32.951403 EDT
rmgr: Standby     len (rec/tot):     50/    50, tx:          0, lsn: 0/2E000380, prev 0/2E000358, desc: RUNNING_XACTS nextXid 860 latestCompletedXid 859 oldestRunningXid 860
rmgr: XLOG        len (rec/tot):     24/    24, tx:          0, lsn: 0/2E0003B8, prev 0/2E000380, desc: SWITCH
rmgr: XLOG        len (rec/tot):    114/   114, tx:          0, lsn: 0/2F000028, prev 0/2E0003B8, desc: CHECKPOINT_SHUTDOWN redo 0/2F000028; tli 3; prev tli 3; fpw true; xid 0:860; oid 16918; multi 1; offset 0; oldest xid 717 in DB 1; oldest multi 1 in DB 1; oldest/newest commit timestamp xid: 0/0; oldest running xid 0; shutdown
```

Fortunately, there wasn't a substantial amount of activity post-failover. In the provided output, we can observe two transactions (tx 858 and 859) clearly. Transaction 858 executed a delete operation on the row located at line pointer 1 within 'block 0' of the relation identified as '1663/5/16907' (representing the table). Transaction 859, on the other hand, carried out an update on the row found at line pointer 4 within 'block 0' of the same table.

To delve deeper into these objects, we can translate '1663/5/16907' into the actual table name. The first number, 1663, designates the tablespace, the second number, 5, corresponds to the database, and the last number, 16907, represents the OID (object identifier) for the actual relation (table). The following queries assist in mapping this information to the real table name.

```sql
postgres=# select * from pg_tablespace where oid=1663;
 oid  |  spcname   | spcowner | spcacl | spcoptions
------+------------+----------+--------+------------
 1663 | pg_default |       10 |        |
(1 rows)

postgres=# select oid, datname from pg_database where oid=5;
  oid  | datname
-------+----------
     5 | postgres
(1 rows)

postgres=# select oid, relname from pg_class where oid=16907;
  oid  | relname
-------+---------
 16907 | emp
(1 row)

postgres=# select ordinal_position, column_name, is_nullable , data_type
           from information_schema.columns
           where table_name='emp'
           order by ordinal_position;
 ordinal_position | column_name | is_nullable |          data_type
------------------+-------------+-------------+---------------------------
                1 | eid         | NO          | integer
                2 | first_name  | YES         | character varying
                3 | last_name   | YES         | character varying
                4 | last_update | YES         | timestamp without timezone
(4 rows)
```

By utilizing the information extracted from the WAL dump, we were able to construct the following timeline that illustrates the sequence of the two transactions:

- 2023-10-04 10:49:31.092400 EDT
    Raw WAL Dump:
     rmgr: Heap        len (rec/tot):     54/    54, tx:        858, lsn: 0/2E0002A8, prev 0/2E000270, desc: DELETE off 1 flags 0x00 KEYS_UPDATED , blkref #0: rel 1663/5/16907 blk 0
     rmgr: Transaction len (rec/tot):     34/    34, tx:        858, lsn: 0/2E0002E0, prev 0/2E0002A8, desc: COMMIT 2023-10-04 10:49:31.092400 EDT

    Summarized:
      DELETE executed against the row at ctid (0,1).  This was constructed using the block number (blk 0) and the line position (off 1).

- 2023-10-04 10:49:32.951403 EDT
    Raw WAL Dump:
      rmgr: Heap        len (rec/tot):     80/    80, tx:        859, lsn: 0/2E000308, prev 0/2E0002E0, desc: HOT_UPDATE off 4 xmax 859 flags 0x60 ; new off 6 xmax 0, blkref #0: rel 1663/5/16907 blk 0
      rmgr: Transaction len (rec/tot):     34/    34, tx:        859, lsn: 0/2E000358, prev 0/2E000308, desc: COMMIT 2023-10-04 10:49:32.951403 EDT

     Summarized:
      UPDATE (hot) executed against the row at ctid (0,4).  This was a hot update which is important to tracking down the final state of the row.


### Identify Actual Rows

Now that we have gathered information regarding the table and 'ctid' pointers, the subsequent action involves utilizing the 'pageinspect' extension to examine the raw data within the database blocks, often referred to as pages. For the sake of simplicity, in this example, both transactions were performed on the 'emp' table and altered block 0. The following output demonstrates the heap page items as visualized through the 'pageinspect' tool.

```sql
postgres=# select lp, t_xmin, t_xmax, t_ctid, t_attrs from heap_page_item_attrs(get_raw_page('emp',0),'emp');
 lp | t_xmin | t_xmax | t_ctid |                                 t_attrs
----+--------+--------+--------+-------------------------------------------------------------------------
  1 |    854 |    858 | (0,1)  | {"\\x64000000","\\x0b4d61726b","\\x0d536d697468","\\x7014817ae0a90200"}
  2 |    854 |      0 | (0,2)  | {"\\x65000000","\\x0b4a6f686e","\\x0d536d697468","\\x7014817ae0a90200"}
  3 |    854 |      0 | (0,3)  | {"\\x66000000","\\x094a616e","\\x09446f65","\\x7014817ae0a90200"}
  4 |    854 |    859 | (0,6)  | {"\\x67000000","\\x0d4b61746879","\\x09446f65","\\x7014817ae0a90200"}
  5 |    857 |      0 | (0,5)  | {"\\x68000000","\\x0b4d617474","\\x0d4a6f6e6573","\\xefcf6b86e0a90200"}
  6 |    859 |      0 | (0,6)  | {"\\x67000000","\\x0b4a696c6c","\\x09446f65","\\x7014817ae0a90200"}
(6 rows)
```

Based on the information provided above, we can confirm that 't_ctid' (0,1) was indeed deleted in the initial transaction. The presence of a non-zero value in 't_xmax' corroborates this well-established fact. The values in the 't_attrs' column represent the raw hexadecimal data. The translation of this data relies on the data type and the presence of null values within any of the columns. Additionally, it's worth noting that certain fields, not displayed in the previous information, may be required if there were modifications to the table structure during the lifespan of the rows within the block, such as column additions or removals.

To gain further insight into the data, the following shell command was executed to decode the 'eid' and 'first_name' columns, which are the first and second columns in the table.

```shell
$ echo 'eid: ' $((16#64));echo 'first_name: ' $(echo 4d61726b | xxd -r -p)
eid:  100
first_name:  Mark
```

Alright, so here's the sequence of events: 'eid' 100 was deleted at 2023-10-04 10:49:31.092400 EDT, and this deletion was successfully committed.

Now, let's delve into the UPDATE process, which requires a bit more explanation. Since this was a hot update, it implies that the modified row was inserted into the same data block, and the 't_ctid' of line pointer 4 was altered to reference the updated row's location, specifically at 't_ctid' (0,6).  In the output above, observe that 'lp' 4 points to (0,6) and carries a 't_xmax' value of 589, which corresponds to the transaction ID of the update operation.

Moving down to 'lp' 6, you'll notice that it has a 't_xmax' value of zero and points to itself. By gathering information from both of these locations, we can retrieve the values before and after the update for the 'first_name' field using the following command.

```shell
$ echo 'eid: ' $((16#67)); echo  'first_name_before: ' $(echo 4b61746879 | xxd -r -p); echo 'first_name_after: ' $(echo 4a696c6c | xxd -r -p)
eid:  103
first_name_before:  Kathy
first_name_after:  Jill
```

Transaction 589 modified the 'first_name' value for 'eid' 103, changing it from 'Kathy' to 'Jill.' However, before we proceed to the new primary server to enact these modifications, it's imperative that we first conduct a thorough check for any potential conflicts.

### Check for Conflicts

By utilizing pg_waldump on the newly designated primary server, we will scan for alterations made to block 0 within the 'emp' table. From the output provided below, it becomes evident that there was one INSERT and one DELETE operation executed subsequent to the LSN at which timeline 4 was created. As there was activity detected, we proceed to extract the specifics by employing pg_waldump and subsequently displaying the page's content using pageinspect.

```shell
$ pg_waldump  --start=0/2E0002A8 --timeline=4 --block=0 --relation=1663/5/16907 --stats=record
WAL statistics between 0/2E0002A8 and 0/2E000D28:
Type                                           N      (%)          Record size      (%)             FPI size      (%)        Combined size      (%)
----                                           -      ---          -----------      ---             --------      ---        -------------      ---
Heap/INSERT                                    1 ( 50.00)                   79 ( 56.43)                    0 (  0.00)                   79 ( 24.16)
Heap/DELETE                                    1 ( 50.00)                   61 ( 43.57)                  187 (100.00)                  248 ( 75.84)
                                        --------                      --------                      --------                      --------
Total                                          2                           140 [42.81%]                  187 [57.19%]                  327 [100%]

$ pg_waldump  --start=0/2E0002A8 --timeline=4 --block=0 --relation=1663/5/16907
rmgr: Heap        len (rec/tot):     61/   248, tx:        858, lsn: 0/2E000A80, prev 0/2E0007D0, desc: DELETE off 4 flags 0x00 KEYS_UPDATED , blkref #0: rel 1663/5/16907 blk 0 FPW
rmgr: Heap        len (rec/tot):     79/    79, tx:        859, lsn: 0/2E000CD8, prev 0/2E000C70, desc: INSERT off 6 flags 0x00, blkref #0: rel 1663/5/16907 blk 0

$ psql -c "select lp, t_xmin, t_xmax, t_ctid, t_attrs from heap_page_item_attrs(get_raw_page('emp',0),'emp')"

 lp | t_xmin | t_xmax | t_ctid |                                 t_attrs
----+--------+--------+--------+-------------------------------------------------------------------------
  1 |    854 |      0 | (0,1)  | {"\\x64000000","\\x0b4d61726b","\\x0d536d697468","\\x7014817ae0a90200"}
  2 |    854 |      0 | (0,2)  | {"\\x65000000","\\x0b4a6f686e","\\x0d536d697468","\\x7014817ae0a90200"}
  3 |    854 |      0 | (0,3)  | {"\\x66000000","\\x094a616e","\\x09446f65","\\x7014817ae0a90200"}
  4 |    854 |    858 | (0,4)  | {"\\x67000000","\\x0d4b61746879","\\x09446f65","\\x7014817ae0a90200"}
  5 |    857 |      0 | (0,5)  | {"\\x68000000","\\x0b4d617474","\\x0d4a6f6e6573","\\xefcf6b86e0a90200"}
  6 |    859 |      0 | (0,6)  | {"\\x89000000","\\x0d427269616e","\\x0b50616365","\\xb2c59d88e0a90200"}
(6 rows)
```

Based on the information provided above, we can draw the following conclusions and take any required actions to ensure that the data in the 'emp' table is up to date:

- EID 100 is still in database as a current record (0,1) with no t_xmax.
      -   Corrective Action: DELETE FROM emp where eid=100;
- EID 103 has been deleted.
      -   If record no longer needed then nothing to do.
      -   If record updated is still needed then insert record.
          -   Corrective Action:
              -   On old primary:
                      ```sql
                      postgres=# copy (select * from emp where eid=103) to stdout delimiter ',' csv  header;
                      eid,first_name,last_name,last_update
                      103,Jill,Doe,2023-10-04 10:45:46.469488
                      ```
              -   On new primary:
                      ```sql
                      postgres=# copy emp from stdin delimiter ',' csv;
                      Enter data to be copied followed by a newline.
                      End with a backslash and a period on a line by itself, or an EOF signal.
                      >> 103,Jill,Doe,2023-10-04 10:45:46.469488
                      >> \.
                      COPY 1
                      postgres=# select * from emp where eid=103;
                      eid | first_name | last_name |        last_update
                      -----+------------+-----------+----------------------------
                      103 | Jill       | Doe       | 2023-10-04 10:45:46.469488
                      (1 row)
                      ```

## Conclusion

In a production environment, it's highly advisable to implement synchronous replication! Given the current state of network speeds, the associated overhead remains comfortably within the acceptable bounds of most service level agreements for the majority of applications. However, if asynchronous replication becomes a necessity, it's worth contemplating setting the wal_level to 'logical.' This adjustment will enable us to employ a logical decoder to capture modifications from the Write-Ahead Log (WAL).  This will be more efficient than the steps above (and may be the next blog post).

Moreover, consider adding the pageinspect and pg_walinspect extensions. These extensions can furnish us with additional forensic tools, should the need for such tools ever arise.
