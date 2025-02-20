---
layout: post
read_time: true
show_date: true
title:  Bi-directional Replication - Conflict Resolution
date:   2023-10-25 12:00:00 -0400
description: At some point in the life of bi-directional replication some instance will create a conflict during data replication. In this post we will review a simple, but effective, example of identifying conflict and then decide how to resolve it..
img: posts/banners/goats-2189621_640.jpg
tags: [postgres, replication]
author: Brian Pace
---
## Introduction

It will happen.  Despite all of the compensating controls, care, and precautions taken, it will happen.  At some point in the life of bi-directional replication some instance will create a conflict during data replication.   In this post we will review a simple, but effective, example of identifying conflict and then decide how to resolve it.

## Environment Setup

Start by creating two Postgres 16 instances.  Set the following Postgres parameters to configure the instance for logical replication:  

- wal_level = 'logical' 
- max_worker_processes = 16.  

After setting the above parameters, restart Postgres.   For this example, the two Postgres instances will be referred to as pg1 and pg2.

### Create Database Objects

In pg1, execute the following to configure the sample database objects.

```sql
CREATE SEQUENCE emp_eid_seq
START 1 INCREMENT 2;

CREATE TABLE emp (eid int NOT NULL DEFAULT nextval('emp_eid_seq') primary key,
first_name varchar(40),
last_name varchar(40),
email varchar(100),
hire_dt timestamp,
last_update timestamp);

INSERT INTO emp (FIRST_NAME,LAST_NAME,EMAIL,HIRE_DT, LAST_UPDATE) VALUES ('Big','Time','bigtime@example.com','2022-09-25 16:04:47', current_timestamp),('Shamu','Orca','sorca0@example.com','2022-02-05 12:27:41', current_timestamp),('Brady','Bunch','bbunch@example.com','2022-05-12 11:04:57', current_timestamp),('Maggie','May','mmayh@example.com','2022-01-28 17:51:29', current_timestamp),('Martin','Luther','mluther@example.com','2022-11-12 08:16:30', current_timestamp),('John','Calvin','jcalvin@example.com','2022-03-07 04:06:56', current_timestamp),('George','Washington','george@example.com','2022-11-20 22:05:24', current_timestamp),
('Bob','Smith','bsmith@example.com','2022-03-09 00:07:00', current_timestamp),('Jane','Doe','jdoe@example.com','2022-01-23 05:55:21', current_timestamp),('Hulk','Hogan','hhogan@example.com','2022-04-22 23:05:23', current_timestamp);
```

In pg2, a slightly different script is used to prepare the database objects.

```sql
CREATE SEQUENCE emp_eid_seq START 2 INCREMENT 2;

CREATE TABLE emp (eid int NOT NULL DEFAULT nextval('emp_eid_seq') primary key,
first_name varchar(40),
last_name varchar(40),
email varchar(100),
hire_dt timestamp,
last_update timestamp);
```

Last setup piece is the create a user for replication on both systems.

```sql
CREATE ROLE repuser WITH REPLICATION LOGIN PASSWORD 'welcome1';
GRANT all ON all TABLES IN schema public TO repuser;
GRANT all ON all TABLES IN schema public TO repuser;
GRANT all ON DATABASE pg1,pg2 TO repuser;
GRANT pg_create_subscription TO repuser;
```

### Publication

Using a publish/subscribe model, changes captured in one Postgres instance (publisher) can be replicated to multiple Postgres instance (subscribers).  

Using the command below create a publisher on each instance.  pg1:

```sql
CREATE PUBLICATION hrpub1
FOR TABLE emp;
```

Create a publication in pg2:

```sql
CREATE PUBLICATION hrpub2
FOR TABLE emp;
```

### Subscription

With the publishers ready, the next step is to create the subscribers.  Since we are doing bi-directional, we will allow the snapshot from pg1 to pg2, but do not need the reverse copy to happen and will therefore disable the initial copy.  

In database pg1, connected as repuser:

```sql
CREATE SUBSCRIPTION hrsub1
  CONNECTION 'host=pg2 port=5432 user=repuser password=welcome1 dbname=postgres'   
  PUBLICATION hrpub2
  WITH (origin = none, copy_data = false, disable_on_error = true, run_as_owner=true);
```

Now, in database pg2, connected as repuser:

```sql
CREATE SUBSCRIPTION hrsub2
  CONNECTION 'host=pg1 port=5432 user=repuser password=welcome1 dbname=postgres'
  PUBLICATION hrpub1
  WITH (origin = none, copy_data = true, disable_on_error = true, run_as_owner=true);
```

### Conflict Trigger

On both databases we will now introduce a trigger that will only fire if the database user (current_user) is 'repuser'.  Remeber, 'repuser' is the database id used by the apply worker process.  By limiting the trigger to only fire for 'repuser' will ensure that we are not adding unnecessary overhead to the application.

In both pg1 and pg2 databases, deploy the trigger using the following:

```sql
CREATE OR REPLACE FUNCTION emp_conflict_func() RETURNS trigger
   LANGUAGE plpgsql AS
$$
BEGIN
   IF NEW.last_update < OLD.last_update THEN
      RAISE EXCEPTION 'Conflict Detected: New record (%) is older than current record  (%) for eid %', NEW.last_update, OLD.last_update, NEW.eid;
   END IF;

  RETURN NEW;
END;
$$;

CREATE TRIGGER emp_conflict_trg
   BEFORE UPDATE ON emp FOR EACH ROW
   WHEN (current_user = 'repuser')
   EXECUTE PROCEDURE emp_conflict_func();

ALTER TABLE emp enable always trigger emp_conflict_trg;
```

Notice a few things:

1. The function for the trigger is comparing the last_update columns between the 'OLD' record (record currently in the table) with the 'NEW' record (record being replicated).  If the last_update value is for the replicated record is older, an exception is raised (record rejected).
2. The trigger is set to always fire.  By default, triggers will not be fired when rows are manipulated by the apply worker.
3. A WHEN clause has been specified to only fire when the current user is 'repuser'. Since we set disable_on_error for the subscription, the subscription will stop (be disabled) if the trigger raises an exception.  

If we wanted to just ignore the older record, we could have set the trigger to 'RETURN NULL' instead of raising an exception as seen here:

```sql
CREATE OR REPLACE FUNCTION emp_conflict_func() RETURNS trigger
   LANGUAGE plpgsql AS
$$
BEGIN
   IF NEW.last_update < OLD.last_update THEN
      RETURN NULL;
   END IF;

  RETURN NEW;
END;
$$;
```

For this example, let's stick with the first instance of this trigger and raise the exception.

## Create Conflict

With everything in place, let's create a conflict and see what happens.  The first employee record (eid=1) has first_name of 'Big' and last_name 'Time'.  The steps below will modify last_name to two different values.  In pg1, last_name will be changed to 'Space'.  In pg2, last_name will be changed to 'Wheel'.  Do not commit until instructed to in order to properly create the conflict.

**Database pg1:**

```sql
BEGIN;
UPDATE emp SET last_name='Space', last_update=current_timestamp WHERE eid=1;
```

**Database pg2:**

Wait a second or two, then execute the following in pg2.

```sql
BEGIN;
UPDATE emp SET last_name='Wheel', last_update=current_timestamp WHERE eid=1;
COMMIT;
```

**Database pg1:**

Back on pg1, let's now commit the transaction.

```sql
COMMIT;
```

Using the following queries, we can observe that the slot (hrslot1) is inactive (active=f) in pg_replication_slots.  The subscription also shows enabled (subenabled) is false in pg_subscription.  Last, in pg_stat_subscription_stats apply_error_count is equal to 1.

```sql
SELECT slot_name, slot_type, database, active, confirmed_flush_lsn 
FROM pg_replication_slots;

slot_name slot_type database active confirmed_flush_lsn
--------- --------- -------- ------ -------------------
hrslot1   logical   pg1      f      0/B1003BB0
hrslot2   logical   pg2      t      0/B1003F08
(2 rows)

SELECT oid, subname, subenabled FROM pg_subscription;

 oid  subname subenabled
----- ------- ----------
16731 hrsub2  f
16730 hrsub1  t
(2 rows)

SELECT * FROM pg_stat_subscription_stats;

subid subname apply_error_count sync_error_count stats_reset
----- ------- ----------------- ---------------- -----------
16731 hrsub2                  1                0 ¤
16730 hrsub1                  0                0 ¤
(2 rows)
```

A glance at the Postgres log will also show the exception raised by the trigger and the LSN where the apply process errored.

```log
2023-10-25 14:18:53 UTC [57232]: [2-1] user=,db=,app=,client=ERROR:  Conflict Detected: New record (2023-10-25 14:18:41.337145) is older than current record  (2023-10-25 14:18:48.027017) for eid 1

2023-10-25 14:18:53 UTC [57232]: [3-1] user=,db=,app=,client=CONTEXT:  PL/pgSQL function public.emp_conflict_func() line 4 at RAISE processing remote data for replication origin "pg_16731" during message type "UPDATE" for replication target relation "public.emp" in transaction 41899, finished at 0/B1003BB0

2023-10-25 14:18:53 UTC [57232]: [4-1] user=,db=,app=,client=LOG:  subscription "hrsub2" has been disabled because of an error
```

To resolve the conflict, we will enable the subscription and skip the transaction with the conflict.

```sql
ALTER SUBSCRIPTION hrsub2 SKIP (lsn='0/B1003BB0');
ALTER SUBSCRIPTION hrsub2 ENABLE;
```

## Conclusion

This is just a simple example of managing conflict in an bi-directional replication setup.  In any conflict management, care must be taken to ensure the integrity of the business data.
