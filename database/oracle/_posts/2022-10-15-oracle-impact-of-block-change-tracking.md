---
layout: post
read_time: true
show_date: true
title:  Impact of Block Change Tracking
date:   2023-11-01 12:00:00 -0400
description: Enabling block change tracking will improve the performance of incremental backups, but at what cost?
img: posts/banners/onesandzeros.jpg
tags: [oracle, backup, performance]
author: Brian Pace
---
## Introduction

Enabling block change tracking will improve the performance of incremental backups, but at what cost?  It is common to hear concerns about turning on a feature that is currently disabled.  After all, most outages occur due to a change in the environment.

The good news is most applications have no measurable impact by enabling Oracle Block Change Tracking.   Databases with data warehouse type workloads are the most common application type that has a measurable impact.  The impact occurs during load processes where hundreds of processes are loading gigabytes of data concurrently.

In this post the goal is to highlight where contention could occur, how to recognize the contention, and possible solutions to resolve it.

## Application Impact

If database performance is impacted, it will occur during the population of data into the CTWR buffer.  If there is free space in this buffer the process to populate information about the changed blocks is extremely fast.  If there is insufficient free space then the process must wait until CTWR updates the block change tracking file and frees up space. The wait event “block change tracking buffer space” indicates the number occurrences and amount of time spent waiting for free space.

When the amount of time spent waiting for “block change tracking buffer space” is a significant percentage of the total waits in the database or for a specific transaction then it is time to consider tuning or disabling block change tracking.

## Possible Solutions

To reduce the waits for “block change tracking buffer space” there are three options.  It is recommended to evaluate these options in the order they are presented.  The first key to fixing this issue is to determine if it is the problem or symptom.

First, check to see if the IO from CTWR to the block change tracking file is optimal. If there is resource contention (busy disk, high cpu utilization) and CTWR is not able to complete its work in a timely manner, then the “block change tracking buffer space” event is a symptom and not the root cause.  Focus your tuning effort on the direct causes for the resource contention.

Second, verify the Large Pool is large enough for the CTWR buffer and that the CTWR buffer is large enough to support the demand.  If necessary, the CTWR buffer can be increased by adjusting “_bct_public_dba_buffer_size”.  Note that a change to this parameter may require adjustment to the Large Pool Size and “_bct_public_dba_buffer_allocation_max” parameter.  Before adjusting any hidden parameter, it is best to engage Oracle Support for guidance. To help size the buffer, Oracle has provided the following query:

```sql
select dba_buffer_count_public*dba_entry_count_public*dba_entry_size*2 from X$KRCSTAT;
```

Last, disable block change tracking.  Although it is very rare, there are some application workloads (mostly large ETL workloads) that cannot tolerate the overhead of enabling block change tracking.  

## Conclusion

Enabling Oracle Block Change Tracking is recommended to improve the performance of incremental backups.  By RMAN not having to scan every block in the database, the backup window will be smaller and infrastructure utilization reduced (CPU, IO).  There is always certain risk to making any change, but in the context of most applications enabling block change tracking will be transparent.
