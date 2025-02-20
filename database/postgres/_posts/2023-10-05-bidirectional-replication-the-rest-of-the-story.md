---
layout: post
read_time: true
show_date: true
title:  Bi-directional Replication - The Rest of the Story
date:   2023-10-05 12:00:00 -0400
description: Can bi-directional replication be achieved from a technical standpoint? Without a doubt! The foundation for such replication lies in logical replication. While the initial setup might seem straightforward, the real challenges arise when it comes to sustaining the environment and guaranteeing consistency with data and performance.
img: posts/banners/automobile-3368094_640.jpg
tags: [postgres, replication]
author: Brian Pace
---

## Introduction

Throughout my three decades of experience in dealing with enterprise databases, the concept of bi-directional (active/active) replication has remained a steady subject of conversation. Can bi-directional replication be achieved from a technical standpoint? Without a doubt! The foundation for such replication lies in logical replication. While the initial setup might seem straightforward, the real challenges arise when it comes to sustaining the environment and guaranteeing consistency with data and performance.

A challenging aspect within bi-directional replication involves mitigating the occurrence of what's termed as a 'transaction loop back'. This phenomenon transpires when a replicated transaction is captured when replayed on the target and subsequently transmitted back to the source as a fresh modification. This, in turn, could be captured anew and relayed to the target, initiating a continuous cycle. Postgres 16 introduces a novel feature aimed at curtailing this loop back occurrence. For an in-depth exploration of this feature, refer to the article titled 'Bi-directional Replication: Postgres is ready, are you?'

## What Problem is Being Addressed?

The initial step in any discussion about bi-directional replication involves assessing the actual necessity. To accurately gauge this requirement, it's crucial to precisely outline the issue at hand. This definition of the problem will essentially mold the entire bi-directional undertaking. It's not uncommon for there to exist a disparity between the perceived requirements of the business or architects and the genuine needs. Based on my experience, there's often an inclination to employ a technical remedy to rectify an operational challenge.

## Impact of Active/Active

Active/active replication changes everything from database object design, object modification processes, and even processing of large batch jobs. Bi-directional replication, or active/active, 
requires logical replication. Logical replication, at some stage of the process, translates the change vectors in transaction logs into SQL statements that are executed on the target.  This is true when using tools such as Shareplex, GoldenGate, etc.  

The good news is that the Postgres apply process does not execute individual SQL statements when applying changes.  Instead it can manipulate the rows directly.  That is good for speed.  It also means that 'per statement' triggers will never fire.  This native feature makes it faster than some of the third-party tools mentioned, but at the end of the day the processing of rows for large data manipulation activities could introduce latency.  Is there a tradeoff?  There is always a tradeoff.  Since Postgres apply is replacing the entire row, it introduces more opportunity for conflicts.  Some of the third party tools, using SQL to apply,  would only manipulate modified columns thus reducing opportunity for conflicts.

Think of the scenario where a single DELETE statement is executed on the source and deletes 1,000,000 rows. The DELETE statement completes in 10 seconds and is committed. What appears in the transaction log is an individual change description for the 1,000,000 rows that were deleted. This translates into 1,000,000 changes to be processed by the replication tool (SQL for third-party, or individual row changes for native).  Assume for a moment that each delete statement takes 0.1 millisecond. What took 10 seconds on the source will take 100 seconds on the target database. Now the target system at this specific point in time is over 1 minute behind your source. Is that ok with your application and business?

The implementation of logical replication also exerts an influence on the alteration of database objects. It necessitates a meticulous process to facilitate database object changes during application releases. In certain cases, this might even entail executing the release in phases, spanning an extended time frame. To illustrate, envision the addition of a new column to a table, complete with a predefined default value. The question arises: how does interim replication manage the data mapping to this field, particularly when the source hasn't yet incorporated the new field?  For a table with 100,000,000 rows, how much lag will be introduced in order to set the default value on existing rows?

By scrutinizing these instances, one can begin to grasp the profound scope of impact and meticulous planning that must be undertaken when contemplating a configuration based on logical replication.

## Data Conflict Resolution

For auditors, an immediate question emerges when confronted with any active/active configuration: How can the assurance of synchronization between both sides be guaranteed? In most instances, the response traditionally relies on row counts. Nonetheless, teams have often gained a hard-earned understanding that placing exclusive reliance on row counts does not ensure the consistency of data. To genuinely confirm data integrity, a sophisticated data comparison solution bridging the source and target systems becomes imperative.

Given the ongoing occurrence of transactions, transient disparities between the two systems might arise. These disparities warrant a secondary assessment to ascertain whether they are rectified by in-flight transactions or if they signify an authentic data anomaly. When data irregularity is identified, its resolution entails the engagement of an individual well-versed in the intricacies of business data and processes. This investigative and resolution process can extend over several hours. Consequently, the deployment of a data auditing solution alongside any logical replication (active/active) mechanism becomes indispensable to effectively tackle these challenges.

## Latency Struggles

Over the course of a bi-directional configuration's lifespan, instances of latency can emerge that surpass established service level agreements. Such occurrences could stem from infrastructure anomalies or unanticipated modifications to the database, leading to replication bottlenecks. Regardless of the trigger, what transpires in the event of latency? Are queries and application activity impeded? If indeed constrained, who is overseeing the monitoring process, and how will automated traffic redirection be orchestrated?

As time progresses, the latency might escalate to a point where the only feasible remedial measure is a re-synchronization of the environment. This undertaking demands a meticulously devised process and a well-calculated timing strategy, which must be an integral component of the initial active/active implementation project.

## Conclusion

Bringing an bi-directional configuration to life and sustaining its operation is invariably more straightforward said than done. Commence any bi-directional initiative by ensuring a thorough grasp of the problem at hand. Once the problem is well understood, embark on research to identify the optimal solution. If active/active proves to be the right approach, allocate ample time for meticulous planning and rigorous testing to navigate the various scenarios elucidated within this post.
