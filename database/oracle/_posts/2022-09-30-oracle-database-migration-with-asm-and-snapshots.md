---
layout: post
read_time: true
show_date: true
title:  Data Migration with ASM and Storage Snapshots
date:   2022-09-30 12:00:00 -0400
description: Planning ahead the data migration could be reduced to minutes by leveraging ASM, Transportable Tablespaces and storage snapshots.
img: posts/banners/diskplatter.png
tags: [oracle, asm, migration, snapshots]
author: Brian Pace
---

## Introduction

It is not uncommon for large system migration efforts to use an interim or staging database to perform data transformation tasks. The challenge comes when the transformation is over, and it is now time to get the data from the staging database to the production database.  If the database is large, this effort could take hours if not days.  However, by planning ahead the data migration could be reduced to minutes by leveraging ASM, Transportable Tablespaces and storage snapshots.  In this document a step by step demonstration of how this process could work is shown.

## Environment Overview

In this demonstration, two servers will be used.  The first server, jaxodbstg01 will be the staging server for the data conversion/migrations.  The second server, jaxodbprd01 is the production database server.

On the staging server is a database called HRSTG.  Within this staging database that is hosted on jaxodbstg01, a tablespace that contains all of the database objects to be migrated has been created (MIG_DATA).  This is an important planning point to highlight.  The data that is specific to the environment or that does not need to be migrated is kept in different tablespaces.  There can be more than one tablespace that is transported, however for this example we are going to use one to keep the demonstration simple.

Last, the MIG_DATA tablespace is stored within an ASM Disk Group called DATA02.  This disk group only exists on jaxodbstg01 at this point.

## Environment Setup

Let’s look first at the ASM Disks and Disk Groups that are configured on each of our database servers.

jaxodbstg01 (Staging Server):

```shell
oracle@jaxodbstg01:+ASM$ sqlplus / as sysasm

SQL*Plus: Release 12.2.0.1.0 Production on Fri Jan 25 09:21:22 2019
Copyright (c) 1982, 2016, Oracle.  All rights reserved.
Connected to:
Oracle Database 12c Enterprise Edition Release 12.2.0.1.0 - 64bit Production

SQL> select name, round(free_mb/1024) free_gb, round(total_mb/1024) total_gb, state      
  2  from v$asm_diskgroup;

NAME                              FREE_GB   TOTAL_GB STATE
------------------------------ ---------- ---------- -----------
DATA01                                 18         20 MOUNTED
DATA02                                  4          4 MOUNTED

SQL> select path from v$asm_disk;

PATH
-------------------------------------------------------------------------
/dev/rdsk/c0t600144F09A0466F300005C4A4BB10003d0s6
/dev/rdsk/c0t600144F09A0466F300005C4A4BD10004d0s6 
```

jaxodbprd01 (Production Server):

```shell
oracle@jaxodbprd01:+ASM$ sqlplus / as sysasm

 SQL*Plus: Release 12.2.0.1.0 Production on Fri Jan 25 09:25:11 2019
Copyright (c) 1982, 2016, Oracle.  All rights reserved.
Connected to:
Oracle Database 12c Enterprise Edition Release 12.2.0.1.0 - 64bit Production

SQL> select name, round(free_mb/1024) free_gb, round(total_mb/1024) total_gb, state
  2  from v$asm_diskgroup;

NAME                              FREE_GB   TOTAL_GB STATE
------------------------------ ---------- ---------- -----------
DATA01                                  8         10 MOUNTED

SQL> select path from v$asm_disk;

PATH
--------------------------------------------------------------------------------
/dev/rdsk/c0t600144F09A0466F300005C4A571C0005d0s6
```

Now that our ASM Disk Group DATA02 is in place and there is no conflicting Disk Group defined on our production server we can move on to creating the tablespace in the HRSTG database and loading it with data.


```shell
oracle@jaxodbprd01:hrstg$ sqlplus / as sysdba

SQL*Plus: Release 12.2.0.1.0 Production on Fri Jan 25 09:32:06 2019
Copyright (c) 1982, 2016, Oracle.  All rights reserved.
Connected to:
Oracle Database 12c Enterprise Edition Release 12.2.0.1.0 - 64bit Production

SQL> create tablespace mig_data datafile ‘+DATA02’ size 1G;      

Tablespace created.

SQL> alter user hr quota unlimited on mig_data;      

User altered.

SQL> create table hr.migtable tablespace mig_data as (select * from dba_tables);

Table created.

SQL> create index hr.migtable_idx1 on hr.migtable (owner, table_name) tablespace mig_data;

Index created.

SQL> select count(1) from hr.migtable;      

  COUNT(1)
----------
      1683
```

The environment has now been setup and some data loaded into our migration tablespace.  Next, the migration.

## Migration

The first step is to place the tablespace(s) that will be transported in read only mode.  In this demonstration, that is the MIG_DATA tablespace.

```shell
oracle@jaxodbstg01:hrstg$ sqlplus / as sysdba

 SQL*Plus: Release 12.2.0.1.0 Production on Fri Jan 25 09:32:06 2019
Copyright (c) 1982, 2016, Oracle.  All rights reserved.
Connected to:
Oracle Database 12c Enterprise Edition Release 12.2.0.1.0 - 64bit Production

SQL> alter tablespace mig_data read only;

Tablespace altered.
```

Next, with the tablespace(s) in read only mode, perform the Transportable Tablespace export on the staging database/server.

```shell
oracle@jaxodbstg01:hrstg$ expdp dumpfile=mig_data_tts.dmp directory=DATA_PUMP_DIR transport_tablespaces=mig_data logfile=mig_data_tts.log

Export: Release 12.2.0.1.0 - Production on Fri Jan 25 10:02:47 2019
Copyright (c) 1982, 2017, Oracle and/or its affiliates.  All rights reserved.
Username: / as sysdba
Connected to: Oracle Database 12c Enterprise Edition Release 12.2.0.1.0 - 64bit Production
Starting "SYS"."SYS_EXPORT_TRANSPORTABLE_01":  /******** AS SYSDBA dumpfile=mig_data_tts.dmp directory=DATA_PUMP_DIR transport_tablespaces=mig_data logfile=mig_data_tts.log 
Processing object type TRANSPORTABLE_EXPORT/INDEX_STATISTICS
Processing object type TRANSPORTABLE_EXPORT/TABLE_STATISTICS
Processing object type TRANSPORTABLE_EXPORT/STATISTICS/MARKER
Processing object type TRANSPORTABLE_EXPORT/PLUGTS_BLK
Processing object type TRANSPORTABLE_EXPORT/POST_INSTANCE/PLUGTS_BLK
Processing object type TRANSPORTABLE_EXPORT/TABLE
Processing object type TRANSPORTABLE_EXPORT/INDEX/INDEX
Master table "SYS"."SYS_EXPORT_TRANSPORTABLE_01" successfully loaded/unloaded
******************************************************************************
Dump file set for SYS.SYS_EXPORT_TRANSPORTABLE_01 is:
 /app/oracle/admin/hrstg/dpdump/mig_data_tts.dmp
******************************************************************************
Datafiles required for transportable tablespace MIG_DATA:
 +DATA02/HRSTG/DATAFILE/mig_data.256.998469457
Job "SYS"."SYS_EXPORT_TRANSPORTABLE_01" successfully completed at Fri Jan 25 10:03:46 2019 elapsed 0 00:00:49
```

Before taking the tablespace out of read only mode, have the Storage Administration team take a snapshot of the LUNs that are used for DATA02 Disk Group.  The snapshot disks can then be presented to the production database server (jaxodbprd01 in this example).  

Once the LUNs are presented, have the Unix Administration team change the ownership of the devices to be owned by the grid infrastructure owner and group.

```shell
root@jaxodbprd01:~# ls /dev/rdsk/*600144F09A0466F300005C4AE0360007*s6
/dev/rdsk/c0t600144F09A0466F300005C4AE0360007d0s6

root@jaxodbprd01:~# chown oracle:oinstall /dev/rdsk/c0t600144F09A0466F300005C4AE0360007d0s6
```

Once the ownership change is complete, we may need to modify the ASM_DISKSTRING parameter and then query V$ASM_DISK to force ASM to scan the disks in the discovery string.  After the ASM scans the disks the DATA02 disk group will appear in V$ASM_DISKGROUP and be in the dismounted state.

```shell
oracle@jaxodbprd01:+ASM$ sqlplus / as sysasm

SQL*Plus: Release 12.2.0.1.0 Production on Fri Jan 25 09:25:11 2019
Copyright (c) 1982, 2016, Oracle.  All rights reserved.
Connected to:
Oracle Database 12c Enterprise Edition Release 12.2.0.1.0 - 64bit Production

SQL> show parameter diskstring

NAME                                TYPE        VALUE
------------------------------------ ----------- ------------------------------
asm_diskstring                       string      /dev/rdsk/*

SQL> select path from v$asm_disk;

PATH
--------------------------------------------------------------------------------
/dev/rdsk/c0t600144F09A0466F300005C4AE0360007d0s6
/dev/rdsk/c0t600144F09A0466F300005C4A571C0005d0s6

SQL> select name, round(free_mb/1024) free_gb, round(total_mb/1024) total_gb, state
  2  from v$asm_diskgroup;

NAME                              FREE_GB   TOTAL_GB STATE
------------------------------ ---------- ---------- -----------
DATA01                                  8         10 MOUNTED
DATA02                                  0          0 DISMOUNTED
```

Now the disk group can be mounted, and some simple verifications performed.

```shell
SQL> alter diskgroup data02 mount;

Diskgroup altered.

SQL> select name, round(free_mb/1024) free_gb, round(total_mb/1024) total_gb, state
  2  from v$asm_diskgroup;

NAME                              FREE_GB   TOTAL_GB STATE
------------------------------ ---------- ---------- -----------
DATA01                                  8         10 MOUNTED
DATA02                                  4          4 MOUNTED

oracle@jaxodbprd01:+ASM$ asmcmd

ASMCMD> cd data02/hrstg/datafile

ASMCMD> ls
MIG_DATA.256.998469457
```

With the data in place, the next step is to perform the transportable tablespace metadata import, verify the data and then open the tablespace in read/write mode.

```shell
oracle@jaxodbprd01:hrprd$ impdp transport_datafiles='+DATA02/HRSTG/DATAFILE/MIG_DATA.256.998469457' DUMPFILE=mig_data_tts.dmp DIRECTORY=DATA_PUMP_DIR logfile=mig_data_tts_imp.log

Import: Release 12.2.0.1.0 - Production on Fri Jan 25 10:32:35 2019
Copyright (c) 1982, 2017, Oracle and/or its affiliates.  All rights reserved.
Username: / as sysdba
Connected to: Oracle Database 12c Enterprise Edition Release 12.2.0.1.0 - 64bit Production
Master table "SYS"."SYS_IMPORT_TRANSPORTABLE_01" successfully loaded/unloaded
Source time zone is -05:00 and target time zone is +00:00.
Starting "SYS"."SYS_IMPORT_TRANSPORTABLE_01":  /******** AS SYSDBA transport_datafiles=+DATA02/HRSTG/DATAFILE/MIG_DATA.256.998469457 DUMPFILE=mig_data_tts.dmp DIRECTORY=DATA_PUMP_DIR logfile=mig_data_tts_imp.log 
Processing object type TRANSPORTABLE_EXPORT/PLUGTS_BLK
Processing object type TRANSPORTABLE_EXPORT/TABLE
Processing object type TRANSPORTABLE_EXPORT/INDEX/INDEX
Processing object type TRANSPORTABLE_EXPORT/INDEX_STATISTICS
Processing object type TRANSPORTABLE_EXPORT/TABLE_STATISTICS
Processing object type TRANSPORTABLE_EXPORT/STATISTICS/MARKER
Processing object type TRANSPORTABLE_EXPORT/POST_INSTANCE/PLUGTS_BLK
Job "SYS"."SYS_IMPORT_TRANSPORTABLE_01" successfully completed at Fri Jan 25 10:33:22 2019 elapsed 0 00:00:35

oracle@jaxodbprd01:hrprd$ sqlplus / as sysdba

SQL*Plus: Release 12.2.0.1.0 Production on Fri Jan 25 10:34:45 2019
Copyright (c) 1982, 2016, Oracle.  All rights reserved.
Connected to:
Oracle Database 12c Enterprise Edition Release 12.2.0.1.0 - 64bit Production

SQL> select tablespace_name, status from dba_tablespaces;

TABLESPACE_NAME                STATUS
------------------------------ ---------
SYSTEM                         ONLINE
SYSAUX                         ONLINE
UNDOTBS1                       ONLINE
TEMP                          ONLINE
USERS                          ONLINE
MIG_DATA                       READ ONLY

6 rows selected.

SQL> select count(1) from hr.migtable;

  COUNT(1)
----------
      1683

SQL> alter tablespace mig_data read write;

Tablespace altered.

SQL> select tablespace_name, status from dba_tablespaces;

TABLESPACE_NAME                STATUS
------------------------------ ---------
SYSTEM                         ONLINE
SYSAUX                         ONLINE
UNDOTBS1                       ONLINE
TEMP                           ONLINE
USERS                          ONLINE
MIG_DATA                       ONLINE

6 rows selected.
```

The migrated data and database are now ready for the business to use!

## Post Migration

Currently the business is fully using the database. The only issue is they are running on a LUNs from a storage snapshot.  This has some overhead on the storage array as it has to now track all of the changed blocks coming from the database to the storage.  

The storage administration team has already provisioned a LUN to the server that will be used for the production database. Without any impact to the business or the database, the following demonstrates assigning the LUN to the DATA02 ASM Disk Group and performing a rebalance operation to remove the snapshot LUN.

```shell
oracle@jaxodbprd01:+ASM$ sqlplus / as sysasm

SQL*Plus: Release 12.2.0.1.0 Production on Fri Jan 25 10:51:00 2019
Copyright (c) 1982, 2016, Oracle.  All rights reserved.
Connected to:
Oracle Database 12c Enterprise Edition Release 12.2.0.1.0 - 64bit Production

SQL> select path from v$asm_disk;

PATH
--------------------------------------------------------------------------------
/dev/rdsk/c0t600144F09A0466F300005C4AE80A0008d0s6
/dev/rdsk/c0t600144F09A0466F300005C4A571C0005d0s6
/dev/rdsk/c0t600144F09A0466F300005C4AE0360007d0s6

SQL> select group_number, name from v$asm_diskgroup order by name;

GROUP_NUMBER NAME
------------ ------------------------------
           1 DATA01
           2 DATA02

SQL> select group_number, name, path from v$asm_disk where group_number=2;

GROUP_NUMBER NAME            PATH
------------ --------------- --------------------------------------------------
           2 DATA02_0000     /dev/rdsk/c0t600144F09A0466F300005C4AE0360007d0s6


SQL> alter diskgroup data02 add disk '/dev/rdsk/c0t600144F09A0466F300005C4AE80A0008d0s6';

Diskgroup altered.

SQL> alter diskgroup data02 rebalance power 2 wait;

Diskgroup altered.

SQL> alter diskgroup data02 drop disk 'DATA02_0000';

Diskgroup altered.

SQL> select group_number, name, path from v$asm_disk where group_number=2;

GROUP_NUMBER NAME            PATH
------------ --------------- --------------------------------------------------
           2 DATA02_0000    /dev/rdsk/c0t600144F09A0466F300005C4AE0360007d0s6
           2 DATA02_0001    /dev/rdsk/c0t600144F09A0466F300005C4AE80A0008d0s6

SQL> select operation, state, power, sofar, est_work, est_minutes from v$asm_operation;

OPERA STAT     POWER      SOFAR   EST_WORK EST_MINUTES
----- ---- ---------- ---------- ---------- -----------
REBAL WAIT         1          0          0           0
REBAL RUN          1         12         13          0
REBAL DONE         1          0          0           0

SQL> /

no rows selected

SQL> select group_number, name, path from v$asm_disk where group_number=2;

GROUP_NUMBER NAME            PATH
------------ --------------- --------------------------------------------------
           2 DATA02_0001    /dev/rdsk/c0t600144F09A0466F300005C4AE80A0008d0s6
```

## Conclusion

This is one technic that can be leveraged to rapidly move large amounts of data in a very small window.  Leveraging ASM, Transportable Tablespaces, and Storage Snapshots together will make you the hero in the eyes of the business.

There is still room for improvement in this process. Notice all of the teams and manual steps that had to be followed and executed.  When dealing with storage, these manual steps add some level of risk due to human error.  Leveraging tools that are database centric in nature like Actifio can reduce the number of teams involved and speed up the process even further.
