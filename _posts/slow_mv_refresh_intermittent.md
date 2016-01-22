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

The first thing I did was to start digging around in ASH, comparing a "good" (10-15min) run with a "bad" (1-2hr) run to see if that could shed any light on the difference in execution time.




