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

code

~~~
function code_example_with_javascript_syntax_highlights() {
  console.log("Showing the syntax highlighting rendering of code using gh-pages-blog.");
}
~~~

###And double indent
Something else in geordie
And a liddle bitty bit of
aasdasdsadsssss
asdasdasdasssspppp
back to work for me!
ssdsdssss

~~~
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
~~~

Show me the last 20 statements currently executing:

~~~
SQL> @/shared/oracle/ssenior/scripts/sql_monitor_summary.sql % % EXECUTING 20

INST STATUS                      SID SQL_ID        SQL_EXEC_ID SQL_EXEC_START      LAST_REFRESH_TIME       PX_REQ     PX_GOT PLAN_HASH_VALUE   DURATION     DBTIME BUFFER_GETS DISK_READS  IO_MBYTES
---- -------------------- ---------- ------------- ----------- ------------------- ------------------- ---------- ---------- --------------- ---------- ---------- ----------- ---------- ----------
   1 EXECUTING                  3507 5bs6rvnn7wn3q    16867067 24-Nov-15 16:41:02  24-Nov-15 16:41:08                             1705989724          6       5.95        1965        717          6
   1 EXECUTING                  1445 5bs6rvnn7wn3q    16867060 24-Nov-15 16:40:52  24-Nov-15 16:41:08                             1705989724         16      15.93        9126       1838         15
   1 EXECUTING                  1399 5bs6rvnn7wn3q    16867057 24-Nov-15 16:40:43  24-Nov-15 16:41:07                             1705989724         24      24.12       29132       4062         32
   1 EXECUTING                  2358 7sju427q5ht9y    16796216 24-Nov-15 16:40:43  24-Nov-15 16:41:07                             3637077694         24      23.55        7519       2857         22
   1 EXECUTING                  1058 7sju427q5ht9y    16796214 24-Nov-15 16:40:38  24-Nov-15 16:41:08                             3637077694         30      29.86       22692       3438         27
   1 EXECUTING                  1540 7sju427q5ht9y    16796213 24-Nov-15 16:40:33  24-Nov-15 16:41:07                             3637077694         34      33.73       24503       2297         18
   1 EXECUTING                  3126 7sju427q5ht9y    16796212 24-Nov-15 16:40:17  24-Nov-15 16:41:07                             3637077694         50         50       15312       6555         51
   1 EXECUTING                   872 7sju427q5ht9y    16796211 24-Nov-15 16:40:12  24-Nov-15 16:41:08                             3637077694         56      55.69       16798       6832         53
   1 EXECUTING                  2835 5bs6rvnn7wn3q    16867035 24-Nov-15 16:40:09  24-Nov-15 16:41:08                             1705989724         59      59.38       13092       9092         71
   1 EXECUTING                  2551 7sju427q5ht9y    16796210 24-Nov-15 16:40:09  24-Nov-15 16:41:07                             3637077694         58      57.89       19852       6760         53
   1 EXECUTING                  3794 7sju427q5ht9y    16796209 24-Nov-15 16:40:05  24-Nov-15 16:41:07                             3637077694         62      61.63       19338       6876         54
   1 EXECUTING                   345 7sju427q5ht9y    16796208 24-Nov-15 16:39:57  24-Nov-15 16:41:07                             3637077694         70      69.98       29459       7832         61
   1 EXECUTING                  1586 fmsyr6bs8xvr9    16802312 24-Nov-15 16:39:52  24-Nov-15 16:41:07                                      0         75      75.34       69532       9704         76
   1 EXECUTING                  2837 7sju427q5ht9y    16796207 24-Nov-15 16:39:39  24-Nov-15 16:41:07                             3637077694         88      88.28       24497       8184         64
   1 EXECUTING                  3607 7sju427q5ht9y    16796206 24-Nov-15 16:39:26  24-Nov-15 16:41:07                             3637077694        101     101.94       29166      12650         99
   1 EXECUTING                   482 7sju427q5ht9y    16796202 24-Nov-15 16:39:06  24-Nov-15 16:41:08                             3637077694        122     122.94       37511      15595        122
   1 EXECUTING                  2595 7sju427q5ht9y    16796201 24-Nov-15 16:38:57  24-Nov-15 16:41:08                             3637077694        131     131.23       44736      15974        125
   1 EXECUTING                     3 7sju427q5ht9y    16796200 24-Nov-15 16:38:53  24-Nov-15 16:41:08                             3637077694        135     135.11       45073      15748        123
   1 EXECUTING                  1972 7sju427q5ht9y    16796197 24-Nov-15 16:38:34  24-Nov-15 16:41:08                             3637077694        154     155.06       68470      20222        158
   1 EXECUTING                  1782 apq6011myzjp9    16777319 24-Nov-15 16:35:50  24-Nov-15 16:41:08                             3094607888        318      320.2     4650836      44820      33911

20 rows selected.
~~~


{% highlight SQL %}
SQL> @/shared/oracle/ssenior/scripts/sql_monitor_summary.sql % % EXECUTING 20

INST STATUS                      SID SQL_ID        SQL_EXEC_ID SQL_EXEC_START      LAST_REFRESH_TIME       PX_REQ     PX_GOT PLAN_HASH_VALUE   DURATION     DBTIME BUFFER_GETS DISK_READS  IO_MBYTES
---- -------------------- ---------- ------------- ----------- ------------------- ------------------- ---------- ---------- --------------- ---------- ---------- ----------- ---------- ----------
   1 EXECUTING                  3507 5bs6rvnn7wn3q    16867067 24-Nov-15 16:41:02  24-Nov-15 16:41:08                             1705989724          6       5.95        1965        717          6
   1 EXECUTING                  1445 5bs6rvnn7wn3q    16867060 24-Nov-15 16:40:52  24-Nov-15 16:41:08                             1705989724         16      15.93        9126       1838         15
   1 EXECUTING                  1399 5bs6rvnn7wn3q    16867057 24-Nov-15 16:40:43  24-Nov-15 16:41:07                             1705989724         24      24.12       29132       4062         32
   1 EXECUTING                  2358 7sju427q5ht9y    16796216 24-Nov-15 16:40:43  24-Nov-15 16:41:07                             3637077694         24      23.55        7519       2857         22
   1 EXECUTING                  1058 7sju427q5ht9y    16796214 24-Nov-15 16:40:38  24-Nov-15 16:41:08                             3637077694         30      29.86       22692       3438         27
   1 EXECUTING                  1540 7sju427q5ht9y    16796213 24-Nov-15 16:40:33  24-Nov-15 16:41:07                             3637077694         34      33.73       24503       2297         18
   1 EXECUTING                  3126 7sju427q5ht9y    16796212 24-Nov-15 16:40:17  24-Nov-15 16:41:07                             3637077694         50         50       15312       6555         51
   1 EXECUTING                   872 7sju427q5ht9y    16796211 24-Nov-15 16:40:12  24-Nov-15 16:41:08                             3637077694         56      55.69       16798       6832         53
   1 EXECUTING                  2835 5bs6rvnn7wn3q    16867035 24-Nov-15 16:40:09  24-Nov-15 16:41:08                             1705989724         59      59.38       13092       9092         71
   1 EXECUTING                  2551 7sju427q5ht9y    16796210 24-Nov-15 16:40:09  24-Nov-15 16:41:07                             3637077694         58      57.89       19852       6760         53
   1 EXECUTING                  3794 7sju427q5ht9y    16796209 24-Nov-15 16:40:05  24-Nov-15 16:41:07                             3637077694         62      61.63       19338       6876         54
   1 EXECUTING                   345 7sju427q5ht9y    16796208 24-Nov-15 16:39:57  24-Nov-15 16:41:07                             3637077694         70      69.98       29459       7832         61
   1 EXECUTING                  1586 fmsyr6bs8xvr9    16802312 24-Nov-15 16:39:52  24-Nov-15 16:41:07                                      0         75      75.34       69532       9704         76
   1 EXECUTING                  2837 7sju427q5ht9y    16796207 24-Nov-15 16:39:39  24-Nov-15 16:41:07                             3637077694         88      88.28       24497       8184         64
   1 EXECUTING                  3607 7sju427q5ht9y    16796206 24-Nov-15 16:39:26  24-Nov-15 16:41:07                             3637077694        101     101.94       29166      12650         99
   1 EXECUTING                   482 7sju427q5ht9y    16796202 24-Nov-15 16:39:06  24-Nov-15 16:41:08                             3637077694        122     122.94       37511      15595        122
   1 EXECUTING                  2595 7sju427q5ht9y    16796201 24-Nov-15 16:38:57  24-Nov-15 16:41:08                             3637077694        131     131.23       44736      15974        125
   1 EXECUTING                     3 7sju427q5ht9y    16796200 24-Nov-15 16:38:53  24-Nov-15 16:41:08                             3637077694        135     135.11       45073      15748        123
   1 EXECUTING                  1972 7sju427q5ht9y    16796197 24-Nov-15 16:38:34  24-Nov-15 16:41:08                             3637077694        154     155.06       68470      20222        158
   1 EXECUTING                  1782 apq6011myzjp9    16777319 24-Nov-15 16:35:50  24-Nov-15 16:41:08                             3094607888        318      320.2     4650836      44820      33911

20 rows selected.
{% endhighlight %}
