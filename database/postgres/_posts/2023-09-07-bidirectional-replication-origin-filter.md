---
layout: post
read_time: true
show_date: true
title:  Bi-directional Replication and the Origin Filter
date:   2023-09-07 12:00:00 -0400
description: A new feature in Postgres 16 makes bi-directional replication possible.
img: posts/banners/eagle_1280.jpg
tags: [postgres, replication]
author: Brian Pace
---

## Introduction

Postgres is a robust and popular open-source relational database management system known for its advanced features and flexibility.  Among its many capabilities, PostgreSQL offers logical replication, a powerful mechanism for replicating data changes across multiple database instances. In this article, we will delve into the world of logical replication with Postgres and focus on a long-awaited feature in Postgres 16 that enables active/active (bi-directional) replication.

## Understanding Logical Replication

Logical replication is a method of replicating data changes based on the logical contents of the database, rather than at the physical level. It allows you to selectively replicate tables, specific columns, or even specific rows based on defined replication rules. This flexibility makes logical replication ideal for scenarios where you need to replicate only a subset of the data or perform transformations during replication.  

To date, using logical replication for bi-directional replication was difficult and at best not very efficient.  Special processing had to occur, normal tricks with partitioning or custom replication processes, to prevent transaction loop back.  Transaction loop back occurs when a transaction is replicated from the source to the target and then replicated back to the source.  In Postgres 16 there is a feature that solves this problem.  When creating a subscription, the subscriber asks the publisher to ignore transactions that were applied via the replication apply process.  This is possible due to the origin messages in the WAL stream.

## Origin Filter

In the WAL stream are information pieces referred to in the documentation as 'origin messages'.  These messages identify the source of the transaction, whether it was local or from an apply process.  Let's take a look at the following to gain some insight into these messages.

Below is an excerpt from pg_waldump from a local transaction:

```shell
rmgr: Standby     len (rec/tot):     50/    50, tx:          0, lsn: 0/47000028, prev 0/46000A40, desc: RUNNING_XACTS nextXid 900 latestCompletedXid 899 oldestRunningXid 900
rmgr: Heap        len (rec/tot):    114/   114, tx:        900, lsn: 0/47000060, prev 0/47000028, desc: HOT_UPDATE off 17 xmax 900 flags 0x10 ; new off 18 xmax 0, blkref #0: rel 1663/5/24792 blk 0
rmgr: Transaction len (rec/tot):     46/    46, tx:        900, lsn: 0/470000D8, prev 0/47000060, desc: COMMIT 2023-06-20 16:43:03.908882 EDT
```

Now let's Compare it with the COMMIT line from the logical replication apply process:

```shell
rmgr: Heap        len (rec/tot):     54/    54, tx:        901, lsn: 0/47000108, prev 0/470000D8, desc: LOCK off 18: xid 901: flags 0x00 LOCK_ONLY EXCL_LOCK KEYS_UPDATED , blkref #0: rel 1663/5/24792 blk 0
rmgr: Heap        len (rec/tot):    117/   117, tx:        901, lsn: 0/47000140, prev 0/47000108, desc: HOT_UPDATE off 18 xmax 901 flags 0x10 KEYS_UPDATED ; new off 19 xmax 901, blkref #0: rel 1663/5/24792 blk 0
rmgr: Transaction len (rec/tot):     65/    65, tx:        901, lsn: 0/470001B8, prev 0/47000140, desc: COMMIT 2023-06-20 16:43:17.412369 EDT; origin: node 1, lsn 6/A95C2780, at 2023-06-20 16:43:17.412675 EDT
```

Notice the origin message in the COMMIT entry.  This indicates that the transaction originated from 'node 1' at source LSN '6/A95C2780'.  With Postgres 16, setting the 'origin=none' flag on the subscriber instructs the publisher to only send messages that do not have this origin information, indicating it was a transaction performed locally.

The rest of this article will focus on setting up a simple active/active replication solution.  For a glimpse of the many gotchas that come with bi-directional replication, take a look at the article "Active/Active Replication: The Rest of the Story".

## Environment

Start by creating two Postgres 16 instances.  Set the following Postgres parameters to configure the instance for logical replication:

- wal_level = 'logical'
- max_worker_processes = 16

After setting the above parameters, restart Postgres.  For this example, the two Postgres instances will be referred to as pg1 and pg2.  

In pg1, execute the following to configure the sample database objects.

```sql
CREATE SEQUENCE emp_eid_seq
START 1
INCREMENT 2;

CREATE TABLE emp (eid int NOT NULL DEFAULT nextval('emp_eid_seq') primary key,
first_name varchar(40),
last_name varchar(40),
email varchar(100),
hire_dt timestamp);

INSERT INTO emp (FIRST_NAME,LAST_NAME,EMAIL,HIRE_DT) VALUES ('Big','Time','bigtime@example.com','2022-09-25 16:04:47'),
('Shamu','Orca','sorca0@example.com','2022-02-05 12:27:41'),
('Brady','Bunch','bbunch@example.com','2022-05-12 11:04:57'),
('Maggie','May','mmayh@example.com','2022-01-28 17:51:29'),
('Martin','Luther','mluther@example.com','2022-11-12 08:16:30'),
('John','Calvin','jcalvin@example.com','2022-03-07 04:06:56'),
('George','Washington','gwashington@example.com','2022-11-20 22:05:24'),
('Bob','Smith','bsmith@example.com','2022-03-09 00:07:00'),
('Jane','Doe','jdoe@example.com','2022-01-23 05:55:21'),
('Hulk','Hogan','hhogan@example.com','2022-04-22 23:05:23');
```

In pg2, a slightly different script is used to prepare the database objects.

```sql
CREATE SEQUENCE emp_eid_seq
START 2
INCREMENT 2;

CREATE TABLE emp (eid int NOT NULL DEFAULT nextval('emp_eid_seq') primary key,
first_name varchar(40),
last_name varchar(40),
email varchar(100),
hire_dt timestamp);
```

Notice special design considerations are already taking shape.  To avoid primary key conflicts, pg1 generates primary key values with odd numbers and pg2 will use even numbers.  

Last setup piece is the create a user for replication on both systems.

```sql
CREATE ROLE repuser WITH REPLICATION LOGIN PASSWORD 'welcome1';
grant all on all tables in schema public to repuser;
```

### Publication

Using a publish/subscribe model, changes captured in one Postgres instance (publisher) can be replicated to multiple Postgres instance (subscribers).  Using the command below create a publisher on each instance.

pg1:

```sql
CREATE PUBLICATION hrpub1
FOR TABLE emp;
pg2:

CREATE PUBLICATION hrpub2
FOR TABLE emp;
```

The publication name could have been the same for each side, but having different names will help us later on as we measure latency using a custom heartbeat table.

### Subscription

With the publishers ready, the next step is to create the subscribers.  By default, logical replication starts with an initial snapshot on the publisher and copies the data to the subscriber.  Since we are doing bi-directional, we will allow the snapshot from pg1 to pg2, but do not need the reverse copy to happen and will therefore disable the initial copy.

pg1:

```sql
CREATE SUBSCRIPTION hrsub1
  CONNECTION 'host=pg2 port=5432 user=repuser password=welcome1 dbname=postgres'
  PUBLICATION hrpub2
  WITH (origin = none, copy_data = false);
```

pg2:

```sql
CREATE SUBSCRIPTION hrsub2
  CONNECTION 'host=pg1 port=5432 user=repuser password=welcome1 dbname=postgres'
  PUBLICATION hrpub1
  WITH (origin = none, copy_data = true);
```

The key is the origin setting in the subscription (origin = none).  The default for origin is 'any' which will instruct the publisher to send all transactions to the subscriber regardless of the source.  For bi-directional this is bad.  With the setting of 'any', an update performed on pg1 would be replicated to pg2 (so far so good).  That replicated transaction would be captured and send back to pg1, and so forth.  This is what we call transaction loopback.

By setting origin to none, the subscriber will request the publisher to only send changes that do not have an origin and thus ignore replicated transaction.  Now, Postgres is ready for bi-directional logical replication.

After a few seconds, verify that the initial copy of the emp table has occurred between pg1 and pg2.

## Replication Test

With the replication configured, update data on each side and verify replication.

pg1:

```sql
SELECT * FROM emp WHERE eid=1;
UPDATE emp SET first_name='Bugs', last_name='Bunny' WHERE eid=1;
```

pg2:

```sql
SELECT * FROM emp WHERE eid=1;
SELECT * FROM emp WHERE eid=3;
UPDATE emp SET first_name='Road', last_name='Runner' WHERE eid=3;
```

## Glimpse of Things to Come

Setting up bi-directional replication is easy.  Before we can open a change ticket to implement in production, there are many things to consider like monitoring, restrictions, change volume, application behavior, backup and recovery, data reconciliation, etc.  Let's do a quick exercise to demonstrate application behavior that results in data integrity issues.  For the following example open a database connection to both pg1 and pg2.  

Start a transaction in each session using the following and note the value of email and last_name.

```sql
BEGIN;
SELECT * FROM emp WHERE eid=1;
```

In pg1 update the email address of the employee with EID = 1 but do not commit.

```sql
UPDATE emp SET email='bugs.bunny@acme.com' WHERE eid=1;
```

In pg2, update the last name but do not commit.

```sql
UPDATE emp SET last_name='Jones' WHERE eid=1;
```

The expectation after committing is that last_name will be equal to Jones and email will be bugs.bunny@acme.com.  Commit the transaction in pg1 and then in pg2.  What happen?

pg1:

```sql
postgres=# SELECT * FROM emp WHERE eid=1;
 eid | first_name | last_name |          email          |       hire_dt
-----+------------+-----------+-------------------------+---------------------
   1 | Bugs       | Jones     | johndoe@example.com.    | 2022-09-25 16:04:47
(1 row)
```

pg2:

```sql
postgres=# SELECT * FROM emp WHERE eid=1;
 eid | first_name | last_name |        email        |       hire_dt
-----+------------+-----------+---------------------+---------------------
   1 | Bugs       | Bunny     | bugs.bunny@acme.com | 2022-09-25 16:04:47
(1 row)
```

Now both rows are out of sync.  In pg1, the update of the email was lost and in pg2 the update of last_name was lost.  This happens because the entire row is sent over during logical replication and not just the fields that were updated.  In such cases, even eventual consistency is not possible.

## Conclusion

Logical replication with PostgreSQL offers a flexible and powerful solution for replicating data changes across multiple database instances. Its ability to selectively replicate data, scalability features, and high availability options make it a valuable tool for various use cases.  New features in Postgres 16 take an already powerful feature and make it even better.  Bi-directional replication is now within reach using native replication.  However, one must plan and test to maintain data integrity and consistency.
