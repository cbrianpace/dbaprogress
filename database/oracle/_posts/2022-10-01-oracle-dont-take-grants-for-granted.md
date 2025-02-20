---
layout: post
read_time: true
show_date: true
title:  Don't Take Grants for Granted
date:   2022-10-01 12:00:00 -0400
description: Think that performing a grant is harmless?  Think again.
img: posts/reference/grantsforgranted.png
tags: [oracle, performance, security]
author: Brian Pace
---

![Concurrency Wait Spike](./assets/img/posts/reference/grantsforgranted.png)

## Impact of Direct Grants

While monitoring a critical database there was a large concurrency spike that resulted in an application performance impact. Investigation revealed that a direct GRANT performed on a single table was the root cause.  Could a simple GRANT cause a performance impact?  One would not think so, at least until we analyze what is actually going on.

The concurrency spike pictured above was Library Cache Locks. The direct GRANT caused all of the dependent SQL statements in the library cache to be invalidated. Therefore, every statement had to be parsed which ultimately caused the contention.  

## Illustration

The following is an example of what happened during this spike that you can use in a lab environment to investigate for yourself using the HR example schema provided by Oracle. The following SQL statement is executed (SQL ID 3jn29k7301m49).  After execution, details are pulled from V$SQL.

```sql
SQL>   select * from hr.employees;

SQL>   select sql_id, first_load_time, executions, invalidations, parse_calls
     from v$sql
     where sql_id = ‘3jn29k7301m49’;

SQL_ID            FIRST_LOAD_TIME      EXECUTIONS  INVALIDATIONS  PARSE_CALLS
--------------    ------------------- ----------   -------------  -----------
3jn29k7301m49     2016-10-04/16:18:54           1              0            1
```
 
Before performing the direct GRANT, pull information from DBA_OBJECTS on the HR.EMPLOYEES table.  Notice the LAST_DDL_TIME before the GRANT.

```sql
SQL>   select object_name, object_type, status, last_ddl_time
     from dba_objects
     where object_name ‘EMPLOYEES’
           and owner=’HR’;

OBJECT_NAME   OBJECT_TYPE  STATUS  LAST_DDL_TIME
-----------   -----------  ------  --------------------
EMPLOYEES     TABLE        VALID   07-JUL-2014 06:56:26
Now perform a GRANT on the HR.EMPLOYEES table.

SQL>   grant select on hr.employees to testuser;

Grant succeeded.
```

Execute the query against DBA_OBJECTS again.  Notice the LAST_DDL_TIME after the grant has changed to the time of the grant. The GRANT modifies the LAST_DDL_TIME in DBA_OBJECTS (actually underlying tables) which triggers an invalidation of all SQL statements (including PL/SQL objects) in the Library Cache that are based on the impacted table.  This invalidation then causes each cached SQL statement (and PL/SQL object) to be parsed during the next execution.  It is the parsing that creates the bottleneck.  In the use case pictured above, it was large PL/SQL objects used for Virtual Private Database (VPD).

```sql
SQL>   select object_name, object_type, status, last_ddl_time
     from dba_objects
     where object_name ‘EMPLOYEES’
           and owner=’HR’;

OBJECT_NAME   OBJECT_TYPE  STATUS  LAST_DDL_TIME
-----------   -----------  ------  --------------------
EMPLOYEES     TABLE        VALID   04-OCT-2016 16:21:02
```

Looking back at V$SQL data shows that indeed the statement has been invalided and parsed.

```sql
SQL>   select sql_id, first_load_time, executions, invalidations, parse_calls
     from v$sql
     where sql_id = ‘3jn29k7301m49’;

SQL_ID            FIRST_LOAD_TIME      EXECUTIONS  INVALIDATIONS  PARSE_CALLS
--------------    ------------------- ----------  -------------  -----------
3jn29k7301m49     2016-10-04/16:18:54           2              1            1
```

## Conclusion

Do not take GRANTs for granted.  They are not harmless actions when they are direct grants.  Performing direct GRANTs on a busy database should be done with extreme caution.  When possible and as a standard, perform grants to roles.  This will avoid invalidations.

If you have strange spikes, take a look if a GRANT performed at the very beginning of the spike caused a snowball effect.
