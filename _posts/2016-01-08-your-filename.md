---
published: true
layout: post
title: "Build The COSMAC ELF A Low-Cost Experimenter’s Microcomputer"
tags: computing old
categories: generic
share: "twitter --twitter-hashtags"
---



<div class="toc"></div>

##Intro
I’ve recently had a chance to start to re-read *Assembly Language Step-by-Step (2nd Edition) by Jeff Duntemann*.

In it the author links to a reproduction of a Popular Electronics article from 1976 on building your own microcomputer by Joseph Weisbecker.

Fascinating stuff about how computers actually work inside and well worth a read:

##Articles
- [Part 1-1](http://incolor.inetnebr.com/bill_r/elf/html/elf-1-33.htm)
- Part 1-2
- Part 1-3
- Part 1-4
- Part 1-5
- Part 1-6

  some code goes here  
    is this still code  
    <?php>

and back to text

##Testing Code

{% highlight javascript %}
function code_example_with_javascript_syntax_highlights() {
  console.log("Showing the syntax highlighting rendering of code using gh-pages-blog.");
}
{% endhighlight %}

###And double indent
Something else in geordie
And a liddle bitty bit of
aasdasdsadsssss
asdasdasdasssspppp
back to work for me!
ssdsdssss

{% highlight sql %}
--------------------------------------------------------------------------------
--
-- File name:   sql_monitor_summary.sql
--
-- Purpose:     Show gv$sql_monitor summary line items
--
-- Author:      Steve Senior
--
-- Usage:       sql_monitor_summary [instance] [sql_id] [status] [top_n_rows]
--
--              e.g. @sql_monitor_summary.sql % 5bs6rvnn7wn3q DONE 13
--                   @sql_monitor_summary.sql % % EXECUTING 15
--
-- Other:
--
--------------------------------------------------------------------------------
set lines 200 pages 1000 verify off
col inst for 99
col status for a20
col SQL_EXEC_START for a19
col LAST_REFRESH_TIME for a19

select * from (
select inst_id inst
      , status
      , sid
      , sql_id
      , sql_exec_id
      , to_char(sql_exec_start,'DD-Mon-YY HH24:MI:SS') sql_exec_start
      , to_char(last_refresh_time,'DD-Mon-YY HH24:MI:SS') last_refresh_time
      , px_servers_requested px_req
      , px_servers_allocated px_got
      , sql_plan_hash_value plan_hash_value
      , round((last_refresh_time-sql_exec_start)*86400,1) duration
      --, round(elapsed_time/1000000,2) duration
      , round((application_wait_time+concurrency_wait_time+cluster_wait_time+user_io_wait_time+plsql_exec_time+java_exec_time+cpu_time)/1000000,2) dbtime
      , buffer_gets
      , disk_reads
      , round((physical_read_bytes+physical_write_bytes)/1024/1024) io_mbytes
from  gv$sql_monitor
where inst_id like '%&1%'
and   sql_id like '%&2%'
and   status like '%&3%'
order by sql_exec_start desc
) where rownum <= nvl(&4,10)
/
{% endhighlight %}
