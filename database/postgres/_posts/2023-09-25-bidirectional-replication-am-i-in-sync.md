---
layout: post
read_time: true
show_date: true
title:  Bi-directional Replication - Am I in-sync?
date:   2023-09-13 12:00:00 -0400
description: Row counts are rarely enough to verify replication.  Here is an easy and efficient way to compare data.
img: posts/banners/illusion_1280.jpg
tags: [postgres, replication]
author: Brian Pace
---
## Introduction

Years of administering databases employing logical replication solutions like Shareplex or GoldenGate have given rise to the necessity of comparing or reconciling tables across distinct schemas or databases. While using row counts can serve as a rudimentary check for soundness, this approach falls short of ensuring data congruence. On the flip side, transferring substantial tables across the network and subsequently iterating through each row and field to establish comparisons exacts an enormous toll on resources.

Fortunately, there's a remedy enabled by key features that transform PostgreSQL into more than a mere database – a comprehensive data platform. In this article, I unveil a solution that conducts comparisons (even on active tables) while circumventing the need to transfer substantial data volumes between databases.

## Creating Environments

Simplifying the environment to enable practice even with limited resources, we will employ a solitary PostgreSQL cluster that accommodates two databases: hrprd and hrrpt. These databases are interconnected using the PostgreSQL Foreign Data Wrapper. The present simulation simulates a scenario where a production database (hrprd) is paired with a reporting database (hrrpt). It's important to note that the source and target databases need not necessarily reside within the same PostgreSQL cluster.

### Production Setup (hrprd)

The steps to create the simulated production database is simple: create the database, create the `postgres_fdw` extension, create the `employee` table and lastly populate the `employee` table with three rows of data.

```sql
postgres=# create database hrprd;
CREATE DATABASE

postgres=# \c hrprd
You are now connected to database "hrprd" as user "postgres".

hrprd=# create extension postgres_fdw;
CREATE EXTENSION

hrprd=# create table employee (id int, first_name varchar(50), last_name varchar(50), department varchar(20));
CREATE TABLE

hrprd=# insert into employee (id, first_name, last_name, department) values (1,'John','Smith','explorer'),(2,'George','Washington','government'),(3,'Thomas','Edison','inventor');
INSERT 0 3
```

### Reporting Setup (hrrpt)

The steps are then repeated to create the simulated reporting database.

```sql
postgres=# create database hrrpt;
CREATE DATABASE

postgres=# \c hrrpt
You are now connected to database "hrrpt" as user "postgres".

hrrpt=# create extension postgres_fdw;
CREATE EXTENSION

hrrpt=# create table employee (id int, first_name varchar(50), last_name varchar(50), department varchar(20));
CREATE TABLE

hrrpt=# insert into employee (id, first_name, last_name, department) values (1,'John','Smith','explorer'),(2,'George','Washington','government'),(3,'Thomas','Edison','inventor');
INSERT 0 3
```

At this point, the configuration is finalized, and the data within the employee table aligns seamlessly across both databases.

## Data Compare

The compare will be performed from the reporting database (`hrrpt`).  A temporary table named `data_compare` is created to store three pieces of information:

- `source_name` column that identifies where the data came from (`hrprd` or `hrrpt` in this example).
- `id` column that will store the value(s) of the primary key from the table. 
- `hash_value` column that stores the hash value of all the non-key fields in the table.

If the table has a composite key, the `id` column would be populated by joining the values into a single string.  The hash occurs on the source side and only the hashed value is used for the comparison, greatly reducing network traffic, transfer time, etc.

### Setup Data Compare

Create the `data_compare` table in both the production (`hrprd`) and target (`hrrpt`) databases.

```sql
hrrpt=# \c hrprd
You are now connected to database "hrprd" as user "postgres".

hrprd=# CREATE TABLE data_compare 
        (source_name VARCHAR(140), 
      id VARCHAR(1000), 
      hash_value varchar(100) 
        );
CREATE TABLE

hrprd=# \c hrrpt
You are now connected to database "hrrpt" as user "postgres".

hrrpt=# CREATE TABLE data_compare 
        (source_name VARCHAR(140), 
      id VARCHAR(1000), 
      hash_value varchar(100) 
        );
CREATE TABLE
```

An `INSERT` statement will be executed on both the source and target to populate the `data_compare` table
and then the contents of the tables compared to identify differences.  To reduce time and transfer for
multiple compare passes, the `data_compare` table contents can be transferred via the foreign table
or `pg_dump`, etc.

The following steps were used to create the foreign table.

```sql
hrrpt=# CREATE SERVER hrprd FOREIGN DATA WRAPPER postgres_fdw OPTIONS (host 'localhost', dbname 'hrprd', port '5432');
CREATE SERVER

hrrpt=# CREATE USER MAPPING FOR current_user SERVER hrprd options (user 'postgres', password 'welcome1');
CREATE USER MAPPING

CREATE FOREIGN TABLE hrprd_data_compare (source_name varchar(140), id varchar(1000), hash_value varchar(100)) SERVER hrprd OPTIONS (table_name 'data_compare');
```

### Perform Initial Compare

Populate the `data_compare` table in both the source (`hrprd`) and target (`hrrpt`) databases.

```sql
hrprd=# INSERT INTO data_compare (source_name, id, hash_value)
  (SELECT 'hrprd' source_name, id::text, md5(concat_ws('|',first_name, last_name, department)) hash_value FROM employee e);
INSERT 0 3

hrrpt=# INSERT INTO data_compare (source_name, id, hash_value)
  (SELECT 'hrrpt' source_name, id::text, md5(concat_ws('|',first_name, last_name, department)) hash_value FROM employee e);
INSERT 0 3
```

At this point we know that the data is exactly the same so let's look at the SQL that is used to perform the actual comparison.

```sql
hrrpt=# SELECT COALESCE(s.id,t.id) id, 
              s.hash_value source_hash_value, t.hash_value target_hash_value,
              CASE WHEN s.hash_value = t.hash_value THEN 'equal'
                    WHEN s.id IS NULL THEN 'row not on source'
                    WHEN t.id IS NULL THEN 'row not on target'
                    ELSE 'difference'
              END compare_result
        FROM hrprd_data_compare s
            FULL JOIN data_compare t ON s.id=t.id;


 id |        source_hash_value         |        target_hash_value         |  compare_result   
----+----------------------------------+----------------------------------+-------------------
 1  | 681c37a127083d90164a9f04b5f92759 | 681c37a127083d90164a9f04b5f92759 | equal
 2  | 6e181f686815319daa07c5e0e1ddcd27 | 6e181f686815319daa07c5e0e1ddcd27 | equal
 3  | 4d4eba0d792cb227d247a3b0f9f66979 | 4d4eba0d792cb227d247a3b0f9f66979 | equal
(3 rows)
```

The `compare_result` confirms that two sets of data are equal.  An alternate compare SQL is included at the end of this article to show various ways the data can be compared when the two `data_compare` tables are combined.

### Create an Out-Of-Sync Condition and Compare

At this stage, three rows exist in the table and the data matches. To create the out of sync, the following changes will be performed:

- In `hrprd`, add CS Lewis with id 4, Charles Babbage with id 5, Blaise Pascal with id 6.
- In `hrrpt`, add Charles Babbage with id 4, CS Lewis with id 5, Kenny Rogers with id 7.

Notice that the ids for CS Lewis and Charles Babbage have been swapped and a unique record added to each database (Blaise Pascal to `hrprd` and Kenny Rogers to `hrrpt`).  The compare should show that 3 rows match, 2 rows have differences and 2 rows are in one database but not the other.

Implement the changes to source (`hrprd`).

```sql
hrprd=# INSERT INTO employee (id, first_name, last_name, department) 
        VALUES (4,'CS','Lewis','author'),(5,'Charles','Babbage','math'),(6,'Blaise','Pascal','math');

hrprd=# SELECT * FROM employee ORDER BY id;
 id | first_name | last_name  | department 
----+------------+------------+------------
  1 | John       | Smith      | explorer
  2 | George     | Washington | government
  3 | Thomas     | Edison     | inventor
  4 | CS         | Lewis      | author
  5 | Charles    | Babbage    | math
  6 | Blaise     | Pascal     | math
(6 rows)
```

Now the changes to the target (`hrrpt`).

```sql
hrrpt=# INSERT INTO employee (id, first_name, last_name, department) 
        VALUES (5,'CS','Lewis','author'),(4,'Charles','Babbage','math'),(7,'Kenny','Rogers','music');

hrrpt=# SELECT * FROM employee ORDER BY id;
 id | first_name | last_name  | department 
----+------------+------------+------------
  1 | John       | Smith      | explorer
  2 | George     | Washington | government
  3 | Thomas     | Edison     | inventor
  4 | Charles    | Babbage    | math
  5 | CS         | Lewis      | author
  7 | Kenny      | Rogers     | music
(6 rows)
```

To summarize the current state:

- Three rows that match (id=1, 2, 3)
- Two rows that do not match (id=4, id=5)
- Two rows that exist in one but not the other (id=6, id=7)

Let's now clear the `data_compare` tables and perform the compare again.

```sql
postgres=# \c hrprd
```

You are now connected to database "hrprd" as user "postgres".

```sql
hrprd=# DELETE FROM data_compare;
DELETE 3

hrprd=# INSERT INTO data_compare (source_name, id, hash_value)
  (SELECT 'hrprd' source_name, id::text id, md5(textin(record_out(e))) FROM employee e);
INSERT 0 6

hrprd=# \c hrrpt
```

You are now connected to database "hrrpt" as user "postgres".

```sql
hrrpt=# DELETE FROM data_compare;
DELETE 3

hrrpt=# INSERT INTO data_compare (source_name, id, hash_value)
  (SELECT 'hrrpt' source_name, id::text id, md5(textin(record_out(e))) FROM employee e);
INSERT 0 6

hrrpt=# SELECT COALESCE(s.id,t.id) id, 
              s.hash_value source_hash_value, t.hash_value target_hash_value,
              CASE WHEN s.hash_value = t.hash_value THEN 'equal'
                    WHEN s.id IS NULL THEN 'row not on source'
                    WHEN t.id IS NULL THEN 'row not on target'
                    ELSE 'difference'
              END compare_result
        FROM hrprd_data_compare s
            FULL JOIN data_compare t ON s.id=t.id;


 id |        source_hash_value         |        target_hash_value         |  compare_result   
----+----------------------------------+----------------------------------+-------------------
 1  | 681c37a127083d90164a9f04b5f92759 | 681c37a127083d90164a9f04b5f92759 | equal
 2  | 6e181f686815319daa07c5e0e1ddcd27 | 6e181f686815319daa07c5e0e1ddcd27 | equal
 3  | 4d4eba0d792cb227d247a3b0f9f66979 | 4d4eba0d792cb227d247a3b0f9f66979 | equal
 4  | bbee9d6cccbeac4e9125ec78507c4eb7 | 57acef6ed228a52b8c42f0a6c155e62b | difference
 5  | 57acef6ed228a52b8c42f0a6c155e62b | bbee9d6cccbeac4e9125ec78507c4eb7 | difference
 6  | 047742fb256df0b78cebc3fbbc3ca4ad |                                  | row not on target
 7  |                                  | 66e5e35673780bd392d2f81d589fbb52 | row not on source
 (7 rows)
```

The above output indicates that rows with id = 1 thru 3 exists in both databases and the content of the rows match.  Rows with id 4 and 5 exists in each database but the contents of the row is different.  Going a step further, one could see that the hash values are the same between the two different rows but associated to the wrong id. Row with id 6 only exist on the target (`hrrpt`) while the row with id 7 only exists on the source (`hrprd`).  In total, there are 4 rows that are out of sync.

With the rows identified, proper steps can be performed to sync the appropriate rows.  Last thought, imagine for a moment that logical replication was in place between the two databases and changes were pending on the target due to lag.  The INSERT into the `data_compare` could be performed only on the rows flagged as out of sync to verify just those rows once replication lag is gone.

## Conclusion

Comparing data can be a monumental task.  However, this little trick has come in handy over the years when expensive data compare software packages were not an option.  There is still room for some creativity with the compare SQL to meet the exact needs of the compare.  For example, only show rows that are missing from one side or the other.

Alternate Compare SQL:

```sql
SELECT id, hash_value,
       count(src1) src1, 
       count(src2) src2
 FROM 
     ( SELECT a.*, 
              1 src1, 
              null src2 
        FROM data_compare a
        WHERE source_name='hrprd'
        UNION ALL
        SELECT b.*, 
               null src1, 
               2 src2 
        FROM data_compare b
        WHERE source_name='hrrpt'
    ) c
 GROUP BY id, hash_value
 HAVING count(src1) <> count(src2);
```
