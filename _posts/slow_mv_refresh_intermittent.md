---
published: false
layout: post
title: "Why is my materialized view refresh quick sometimes and slow at other times"
tags: Materialized View Performance Refresh
categories: Performance
date: "2016-01-12 09:46"
---
Our BI application team reported an issue where a particular job within their batch schedule varied in elapsed time from 10-15mins on some occasions to between 1-2hrs on other occasions.

As always, I asked them to break down the job into it's constituent parts so I can start to get a deeper understanding of where the problem might be.

At a high level the job did this:

1. Load new data into tables
2. DBMS_STATS.GATHER_TABLE_STATS on a 115 partition table
3. alter session set "_mv_refresh_costing"='rule_pt_pd_fa_co'
4. alter session enable parallel dml
5. DBMS_MVIEW.REFRESH on a partitioned MVIEW with parameters of: method=>'?' and ATOMIC_REFRESH=>false
6. DBMS_STATS.GATHER_TABLE_STATS on the MVIEW
7. alter session disable parallel dml

*Footnote: point 3 struck me as interesting as I've never come across that underscore parameter before and presumably there is a good reason for it being included. Made a note to find out about this*

The first thing I did was to start digging around in ASH, comparing a "good" (10-15min) run with a "bad" (1-2hr) run to see if that could shed any light on the difference in execution time. Of course I could and should trace a good and bad run, but this is a quick way to see what I can find out without having to wait for the next run, raise a change (yes!) to trace the session etc.

This is a useful piece of SQL to run against DBA_HIST_ACTIVE_SESS_HISTORY to show how long each SQL_ID was executing between a given date range - in this case I ran it for both the "good" and "bad" batch runs:

```
set lines 200 pages 1000
col sql_exec_start for a25
col LAST_SAMPLE_TIME for a25
col DUR_MINS for a30
col SQL_TEXT for a50

select a.sql_id
       ,to_char(min_time,'DD-MON-YY HH24:MI:SS') sql_exec_start
       ,to_char(max_time,'DD-MON-YY HH24:MI:SS') last_sample_time
       ,max_time-min_time dur_mins
       ,dbms_lob.substr(sql_text,50) sql_text
from (      
select sql_id
      ,min(sql_exec_start) min_time
      ,max(sample_time) max_time
      ,count(*) num_samples
from  dba_hist_active_sess_history
where sample_time between to_date('12-JAN-16 04:32:05','DD-MON-YY HH24:MI:SS') and to_date('12-JAN-16 05:45:56','DD-MON-YY HH24:MI:SS')
and   session_type='FOREGROUND'
--filter here by session_id, module, action, consumer_group_id etc
and consumer_group_id=925847 --our batch consumer group 
group by sql_id) a, dba_hist_sqltext b
where a.sql_id=b.sql_id(+)
order by 2;
```

Unfortunately we didn't have any MODULE or ACTION set for this particular job by DBMS_APPLICATION_INFO so the results brought back all of the batch jobs running at the time. With a bit of manual filtering of the results on SQL_TEXT based on my understanding of the process flow, the following were the results I was interested in:

###good

```
SQL_ID        SQL_EXEC_START            LAST_SAMPLE_TIME          DUR_MINS                       SQL_TEXT
------------- ------------------------- ------------------------- ------------------------------ --------------------------------------------------
fkndgf5rbd7sa 12-JAN-16 13:36:45        12-JAN-16 13:38:39        +000000000 00:01:54.075        /* MV_REFRESH (MRG) */ MERGE INTO "XXXXXX"."AAAAAA
d7mdgsbrqy4bd 12-JAN-16 13:38:40        12-JAN-16 13:38:50        +000000000 00:00:10.039        /* MV_REFRESH (INS) */ INSERT /*+ APPEND BYPASS_RE
```

###bad

```
SQL_ID        SQL_EXEC_START            LAST_SAMPLE_TIME          DUR_MINS                       SQL_TEXT
------------- ------------------------- ------------------------- ------------------------------ --------------------------------------------------
621azpj5fhuan 12-JAN-16 04:35:07        12-JAN-16 05:30:02        +000000000 00:54:55.001        BEGIN
                                                                                                 DBMS_MVIEW.REFRESH('XXXX.WC_GL_BALANCE_A'
0zpm3xdh8b2aa 12-JAN-16 04:35:28        12-JAN-16 05:17:57        +000000000 00:42:29.655        /* MV_REFRESH (INS) */INSERT /*+ BYPASS_RECURSIVE_
```

So on the good run the MView refresh was using a MERGE INTO then an INSERT and it took 01:54, and on the bad run it was only doing an INSERT taking 42:29. Quite a difference eh.

Now, I'm no expert in MView refreshes, but what I do know that they can be refreshed completely (COMPLETE) or incrementally (FAST). So the initial theory here is that is what is happening - on the good runs Oracle is able to do a fast refresh whereas on the bad runs Oracle is doing a complete refresh.

I need to:

1. See if there is a way to prove that it is doing a FAST on the good runs and a COMPLETE on the others
2. Understand why it is doing this

But before I do that, I need to go and do some research about the different methods of refreshing MViews.

Oracle can refresh a materialized view using either a fast, complete, or force refresh [ref](https://docs.oracle.com/cd/E11882_01/server.112/e10706/repmview.htm#REPLN351).

A very high level summary:

| Method        | Description  |
| ------------- |-------------|
| COMPLETE  | To perform a complete refresh of a materialized view, the server that manages the materialized view executes the materialized view's defining query, which essentially re-creates the materialized view. To refresh the materialized view, the result set of the query replaces the existing materialized view data. Oracle can perform a complete refresh for any materialized view.  |
| FAST | Uses Materialized View Logs which are created on the tables defined in the MView query. These logs track changes since the last refresh. Oracle uses these to identify the changes that occurred in the master since the most recent refresh of the materialized view and then applies these changes to the materialized view. |
| FORCE | Oracle tries to perform a fast refresh. If a fast refresh is not possible, then Oracle performs a complete refresh. Use the force setting when you want a materialized view to refresh if a fast refresh is not possible |

That note also introduces the following which was new to me:

> If you have materialized views based on partitioned master tables, then you might be able to use **Partition Change Tracking (PCT)** to identify which materialized view rows correspond to a particular partition. PCT is also used to support fast refresh after partition maintenance operations on a materialized view's master table. PCT-based refresh on a materialized view is possible only if several conditions are satisfied.

Wait, our objects are partitioned so this could definitely be a factor here.

To summarise our MView:

1. It is a complex MView as it uses 3 tables joined in the query. 
2. 1 of those tables is range partitioned into 115 partitions
3. The other 2 tables are non-partitioned and these both have Materialized View Logs created on them
2. The MView itself is also range partitioned into 115 partitions, using the same key as the partitioned table 

Let's find out a bit more about PCT then [ref](https://docs.oracle.com/cd/E11882_01/server.112/e25554/advmv.htm#DWHSG8227):

> It is possible and advantageous to track freshness to a finer grain than the entire materialized view. The ability to identify which rows in a materialized view are affected by a certain detail table partition, is known as Partition Change Tracking. When one or more of the detail tables are partitioned, it may be possible to identify the specific rows in the materialized view that correspond to a modified detail partition(s); those rows become stale when a partition is modified while all other rows remain fresh.

> You can use PCT to identify which materialized view rows correspond to a particular partition. **PCT is also used to support fast refresh after partition maintenance operations on detail tables.** For instance, if a detail table partition is truncated or dropped, the affected rows in the materialized view are identified and deleted.

To support PCT, a materialized view must satisfy the following requirements:

1. At least one of the detail tables referenced by the materialized view must be partitioned
2. Partitioned tables must use either range, list or composite partitioning
3. The top level partition key must consist of only a single column
4. The materialized view must contain either the partition key column or a partition marker or ROWID or join dependent expression of the detail table. See Oracle Database PL/SQL Packages and Types Reference for details regarding the DBMS_MVIEW.PMARKER function
5.If PCT is enabled using either the partitioning key column or join expressions, the materialized view should be range or list partitioned
6. If you use a GROUP BY clause, the partition key column or the partition marker or ROWID or join dependent expression must be present in the GROUP BY clause
7. If you use an analytic window function or the MODEL clause, the partition key column or the partition marker or ROWID or join dependent expression must be present in their respective PARTITION BY subclauses
8. Data modifications can only occur on the partitioned table. If PCT refresh is being done for a table which has join dependent expression in the materialized view, then data modifications should not have occurred in any of the join dependent tables.
9. The COMPATIBILITY initialization parameter must be a minimum of 9.0.0.0.0
10. PCT is not supported for a materialized view that refers to views, remote tables, or outer joins
11. PCT refresh is nonatomic
 
Excellent, so now I have a much better understanding of how an MView can be refreshed. I can see that we have our configuration setup to try and use PCT (MView partitioning matches the one partitioned table in the query) and fast refreshes (both of the other tables have Materalized View Logs to track changes on those tables).

Back to the issue in hand. I need to prove that it is indeed choosing different refresh methods in the good and bad runs.

After a bit of digging in an excellent post by **Robin Moffat** [here](https://rnm1978.wordpress.com/2011/01/08/materialised-views-pct-partition-truncation/) and Mos note [PCT Refresh Issues Delete Where it Should Issue Truncate (Doc ID 733673.1)](https://support.oracle.com/epmos/faces/DocumentDisplay?id=733673.1&displayIndex=4#CHANGE), we need to use a 10979 trace before we initiate the refresh and it will spit out a trace file that will show us what is going on.

What do we find in the traces (heavily edited here)?

###good###

```
partition [1020] is retrieved. 
partition [1030] is retrieved. 
partition [1040] is retrieved. 
Overall dml cause [3] partitions updated
 ONLY TRUNCATE based PCT REFRESH possible 
...
Value of _mv_refresh_costing : rule_pt_pd_fa_co
...
 Refresh method Complete : 
...
 Refresh method PCT - MIX : 
...
 Refresh method picked PCT - MIX 
 Executed Stmt -
 /* MV_REFRESH (MRG) */ MERGE INTO...
 ALTER TABLE "XXXX"."OUR_MVIEW" TRUNCATE PARTITION PART_20160229 UPDATE GLOBAL INDEXES 
 ALTER TABLE "XXXX"."OUR_MVIEW" TRUNCATE PARTITION PART_20160201 UPDATE GLOBAL INDEXES 
 ALTER TABLE "XXXX"."OUR_MVIEW" TRUNCATE PARTITION PART_20151228 UPDATE GLOBAL INDEXES
 /* MV_REFRESH (INS) */ INSERT /*+ APPEND BYPASS_RECURSIVE_CHECK */...
```

This shows Oracle evaluating a Complete refresh with a PCT (MIX?) refresh and deciding to use the PCT - MIX version. We can see it firstly doing the MERGE INTO, then truncating the partitions in the MView that it is going to refresh, and finally doing the insert into the MView but **only for those partitions it truncated**. Note this sequence of events correlates back to what we saw from our ASH query earlier.

###bad###

```
 Refresh method picked Complete 
 /* MV_REFRESH (DEL) */ truncate table "XXXX"."OUR_MVIEW" purge snapshot log
 /* MV_REFRESH (INS) */INSERT INTO "XXXX"."OUR_MVIEW"...
```

This shows no evaluation of any methods. Oracle has immediately decided that only a Complete refresh is possible. It truncates the entire MView and then recreates it from scratch.

So now we have the proof that it our hypothesis was correct.

Now we need to answer the second question - why is it not able to use a PCT refresh?
