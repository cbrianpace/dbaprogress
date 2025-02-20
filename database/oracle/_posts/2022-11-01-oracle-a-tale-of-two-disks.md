---
layout: post
read_time: true
show_date: true
title:  A Tale of Disks and Disk Groups
date:   2022-11-01 12:00:00 -0400
description: Row counts are rarely enough to verify replication.  Here is an easy and efficient way to compare data.
img: posts/banners/diskplatter.png
tags: [oracle, performance, asm, disk]
author: Brian Pace
---
## Introduction

The topic of how many disks, disk allocations, file separation, etc. have been around for years between DBAs, Storage Admins, Architects, etc. There is disagreement within these groups as well, not just between them.  Which one is right?  Well, it depends on what the goal is.   The DBAs are often in favor of high performance and low risk configurations while storage administrators are trying to control cost.  Many of the marketing papers for ASM make it seem that all of these topics are no longer an issue.  Implement ASM, and it will ‘take care of everything’.  While ASM does add a lot of value and is worth implementing even on single instances, it has just changed the context of these age-old topics.  

Before digging into this topic, let me be clear that this does not apply to Exadata.  Exadata is a special case that needs to be addressed in a totally different context.

## Best Practices

To settle these questions, technical people often turn to best practice white papers for the answers.  The challenge with these papers is that the definition of ‘best’ is rarely given.  In fact, the answer will vary based on what team contributed to each section.  

Take Oracle’s own best practice white paper (810394.1).  Looking closely through this document, you will see the old arguments still exist today, and even in the same paper.    For example, in the section ‘Storage Considerations’ it is noted, ‘It is recommended to maintain no more than 2 ASM disk groups…’. There is obviously a good reason that Oracle makes this statement.  The challenge is that the rationale behind the decision nor the context is given (most of the time it is for the sake of simplicity, although there is a slight system overhead piece to the puzzle).  Next, look at the very next section in the paper.  Under ‘Clusterware and Grid Infrastructure Configuration Considerations’ section, the best practice for OCR/Voting disks reads, ‘When storing the OCR and Voting Disk within ASM in 11gR2 and higher it is recommended to maintain a separate diskgroup for OCR and Voting Disk (not in the same disk group which stores database related files).’   Within this single best practice white paper, it is recommended to only have 2 ASM Disk Groups, yet in the next section the instruction is to have a third just for OCR and Voting Disk.

Is there a conflict here?  No, they are both statements based on certain objectives and a specific context.  Since the rationale is not provided, we have to take data like this and filter it through experience, case studies, and other recommendations from Oracle.  Another good place to look is how Oracle builds their ‘appliances’.  In the case of these two examples, we see that most Exadata and SuperCluster builds do have three disk groups (DATA, RECO, and DBFS/GRID).

## Number of Disk Groups and LUNs/Disks

How many disk groups do you need?  From the best practice document, the assumption is we need at least 3 disk groups for a RAC installation.  First, the DATA disk group for the database files.  Second, the RECO/FRA for the archive logs and recovery areas.  Third, GRID disk group for the OCR and Voting Disks.  Oracle goes on to state that each disk group should have 4 LUNs/disks.  Tests results, whitepapers, experience etc. shows the number of LUNs has a direct impact on the achievable IOPS due to device queues/host queue depths/etc.  In fact, most storage vendors will encourage you to do 4 LUNs per ‘array’ (which has a different definition from vendor to vendor).  EMC goes a step further for XtremeIO recommending 1 LUN per component (for the recovery disk group).  Like all things there are tradeoffs and ultimate what needs to be considered is the cost versus reward.   More LUNs means higher potential IOPS (from an O/S perspective).  It also means higher overhead on the database node (from ASM processes monitoring health and rebalance needs).  If the goal is to achieve high IOPS, then the 10-15% CPU overhead of ASM would be worth the increased through-put to the application/database.   The recommendation of 4 is a good starting point and is often sufficient for most workloads.

Before we start ordering 4 LUNs for every disk group, we need to consider what the disk group is being used for.  For example, when using a disk group for the voting disk using high redundancy you want an odd number of disks (5 is the recommendation here unless using external redundancy then only 1 LUN is recommended).  So it is easy to see how all of these considerations must be heavily weighed before committing to a single direction.

## Disk Group for Redo Logs?

With today’s technology, do you still need to multiplex?  Back in the day the goal was to have separate mount points (isolated disk) for redo, is this still required?  Depending on who you ask, you will get different answers.  You will also get different answers based on the technology stack (NAS, SAN, Engineered System).  With Oracle Engineered Systems, there is not a concern over the placement of Redo Logs since the Exadata Storage Cell is intelligent and automatically prioritizes log writer (LGWR) activity.  Once outside the world of Oracle engineered systems, that intelligence does not exist (at least at the same level).  Most storage arrays cannot tell the difference between a sequential write going to a temporary segment and LGWR writing to redo logs.  However, there is the ability to specify quality of service levels by the host/LUN within the storage array.  By giving the LUNs used for redo logs a higher priority will reduce log writer-based waits.  

Leaving the performance aspect for a moment, another reason for the REDOx disk group is data protection.  Loss of a redo log group could mean loss of data and even database outages.  Most of the ‘storage outages’ are not caused by the physical storage at all.  Instead they are caused by human error (stepping on LUNs) and software bugs (like the ASM bug that corrupts the entire disk group, corrupt blocks, etc.).  By placing the redo logs in separate disk groups, we reduce, but do not eliminate, these risks and further reduce the risk of data loss.  The question then is, why not just place the multiplexed redo log in the RECO disk group? This is a good question and has two answers.  First, the goal of the RECO disk group should be to use LUNs from cheaper (which means slow) storage in order to manage cost.  Second, as software defined storage becomes the new norm (iSCSI and dNFS), you need to be specific about how the storage is configured (log bias and recordsize/volblocksize).  Setting these incorrectly will result in noticeable performance differences.

## Conclusion

To summarize, the most flexible and cost-effective storage solution would look something like the following:

Disk Group|Storage Medal Rating*|ASM Redundancy/RAID|Contents
DATA|GOLD|HIGH / 5|Control file 1 and 2, Data files, Temp files
RECO|BRONZE|NORMAL / Mirror|Archive Logs, Flash Recovery Area, Backups
GRID|GOLD|HIGH / None|OCR, Voting Disks
REDOx|GOLD|NORMAL / Mirror**|Redo Logs
 
* Gold = High Performance, Silver = Medium, Bronze = Inexpensive Low Performance
** Since multiplexing is used, a good argument could be made for not using any RAID on the storage side.  However, most storage solutions today have little overhead for RAID as all writes go to cache.

As new storage technology comes out, these topics will continue to be discussed at many levels.  For example, now that solid state is becoming more popular one would think that just having two disk groups would be the answer once and for all.  However, if you read EMC’s paper on Oracle Best Practices for XtremIO the recommendation is still similar to the above table (http://www.emc.com/collateral/white-papers/h13497-oracle-best-practices-xtremio-wp.pdf). The one exception is they make no recommendation of putting RECO (FRADG as they call it) on slower and cheaper storage (but they have not benefited from that recommendation, so they do not make it in the paper as a best practice).
