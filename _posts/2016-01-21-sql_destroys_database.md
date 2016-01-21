---
published: true
layout: post
title: "How a piece of SQL can destroy your database or cause a node eviction"
tags: XML Performance SQL
categories: Performance
date: "2015-12-17 14:34"
---
We recently had a situation where our 8 node cluster started evicting nodes. I was able to jump on and observe what was happening at the time.

I saw massive swap operations and this was causing the heartbeat mechanism to timeout and fail leading to node evictions.

When I looked inside the database I saw a lot of active sessions all running the same SQL:

```
SELECT xmlserialize(DOCUMENT xml_inc_root AS CLOB NO indent) AS xmltext
      INTO v_xml_doc
      FROM
        (SELECT xmlroot (xml_qry.xml_tag, VERSION '1.0" encoding="utf-8') xml_inc_root
        FROM
          (SELECT XMLELEMENT("Stocks", xmlagg( XMLELEMENT("Record", XMLFOREST("ProductID", "LocationID", "ValidDay" , "Timestamp" , "Type", "UnitID" , "Quantity")))) xml_tag
          FROM
            (SELECT "ProductID",
              "LocationID",
              "ValidDay",
              "Timestamp",
              "Type",
              "UnitID",
              "Quantity"
            FROM
              (SELECT "ProductID",
                "LocationID",
                "ValidDay",
                "Timestamp",
                "Type",
                "UnitID",
                "Quantity" --,
               -- row_number() OVER (PARTITION BY 'X' ORDER BY ROWID ASC) rn
              From system.W_Byndr_Stocks
              where "LocationID" between v_loc_from and v_loc_to
              )
            --WHERE rn BETWEEN v_row_loop_from AND v_row_loop_to
            )
          ) xml_qry
        );
```

The memory being consumed by each of these processes was in the GB.

Service was stabilised and then the post-mortem began.

It seems using xmlserialize on such a large dataset is the root cause of the issue since oracle builds a temporary lob segment in memory to store the results.

This naturally leads on to the question : How do you limit the memory that can be consumed by a single oracle process?
 
This is the summary of the situation:
 
From inside the database:

	1. PGA_AGGREGATE_TARGET (Target size for the aggregate PGA memory consumed by the instance) is just a target and can be exceeded
	2. _pga_max_size parameter (Maximum size of the PGA memory for one process) exists but does not limit a process to this value – I have tested and confirmed this to be true
	3. There is no setting in Resource Manager that would limit the PGA consumed by a process
	4. There is a new parameter at 12c (PGA_AGGREGATE_LIMIT) which appears to hard limit the total PGA consumed by the instance which could help, but this is no good for us on 11g

The only way this could be achieved is outside of the database at the os level with, for e.g.:

	1. Using ulimit to set max memory size/virtual memory
	2. Using container groups to limit memory (memory.limit_in_bytes/memory.memsw.limit_in_bytes)

 
The problem with both of these at the os level is they would need to be applied to all oracle processes which would make it dangerous/impractical for us to implement.



And here is the proof

###11g

--create the base table

```
CREATE TABLE "SYSTEM"."W_BYNDR_STOCKS"
   (    "ProductID" VARCHAR2(30),
        "LocationID" NUMBER,
        "ValidDay" VARCHAR2(10),
        "Timestamp" VARCHAR2(25),
        "Type" CHAR(9),
        "UnitID" VARCHAR2(3),
        "Quantity" NUMBER
   ) SEGMENT CREATION IMMEDIATE
  PCTFREE 0 PCTUSED 40 INITRANS 1 MAXTRANS 255
  STORAGE(INITIAL 81920 NEXT 1048576 MINEXTENTS 1 MAXEXTENTS 2147483645
  PCTINCREASE 0 FREELISTS 1 FREELIST GROUPS 1
  BUFFER_POOL DEFAULT FLASH_CACHE DEFAULT CELL_FLASH_CACHE DEFAULT)
  TABLESPACE "SYSAUX";
```

--insert some rows to allow the xml sql to work

```
set serveroutput on
DECLARE
       v_loc NUMBER := 1;
       v_loc_limit NUMBER := 100;
       v_line NUMBER := 1;
       v_line_limit NUMBER := 15000;
       v_sql_stmt VARCHAR2(8000);
       BEGIN
WHILE (v_loc <= v_loc_limit)
       LOOP
              WHILE (v_line <= v_line_limit)
              LOOP
                     --dbms_output.put_line('v_loc: '||v_loc||', v_line: '||v_line);
                     v_sql_stmt := 'INSERT INTO SYSTEM.W_BYNDR_STOCKS values (''100001477'','||v_loc||',''2015-08-26'',''2015-08-26T23:59:59+00:00'',''INVENTORY'',''PCS'',0)';
                     --dbms_output.put_line(v_sql_stmt);
                     execute immediate v_sql_stmt;
                     v_line := v_line + 1;
              END LOOP;
              v_line := 1;
              WHILE (v_line <= v_line_limit)
              LOOP
                     v_sql_stmt := 'INSERT INTO SYSTEM.W_BYNDR_STOCKS values (''100001477'','||v_loc||',''2015-08-08'',''2015-08-26T23:59:59+00:00'',''INVENTORY'',''PCS'',0)';
                     --dbms_output.put_line(v_sql_stmt);
                     execute immediate v_sql_stmt;
                     v_line := v_line + 1;
              END LOOP;
              v_line := 1;
       v_loc := v_loc + 1;
       END LOOP;
commit;      
END;
/
```

--verify what is there

```
set lines 200 pages 1000
select "LocationID"
              ,"ValidDay"
              ,"Type"
              ,count(*)
From   SYSTEM.W_Byndr_Stocks
group by "LocationID"
              ,"ValidDay"
              ,"Type"
order by 1,2,3;
```

Results:

```
LocationID ValidDay   Type        COUNT(*) 
---------- ---------- --------- ---------- 
         1 2015-08-08 INVENTORY      15000
         1 2015-08-26 INVENTORY      15000
         2 2015-08-08 INVENTORY      15000
         2 2015-08-26 INVENTORY      15000
         3 2015-08-08 INVENTORY      15000
         3 2015-08-26 INVENTORY      15000
         4 2015-08-08 INVENTORY      15000
         4 2015-08-26 INVENTORY      15000
         5 2015-08-08 INVENTORY      15000
         5 2015-08-26 INVENTORY      15000
         6 2015-08-08 INVENTORY      15000
         6 2015-08-26 INVENTORY      15000
         7 2015-08-08 INVENTORY      15000
...snip...

200 rows returned
```

--show relevant systemwide pga parameters

```
set linesize 150
set pagesize 1000
col ksppinm for a42
col ksppstvl for 99999
col ksppstdf for a10
col ksppdesc for a70 wrap
select a.ksppinm, round((b.ksppstvl/1024/1024),0) ksppstvl, b.ksppstdf, a.ksppdesc
from x$ksppi a, x$ksppcv b
where a.indx = b.indx
and ksppinm in ('__pga_aggregate_target', '_pga_max_size', 'pga_aggregate_target', 'pga_aggregate_limit')
order by ksppinm
/

KSPPINM                                    KSPPSTVL KSPPSTDF   KSPPDESC
------------------------------------------ -------- ---------- -----------------------------------------------------------------
__pga_aggregate_target                           60 FALSE      Current target size for the aggregate PGA memory consumed
_pga_max_size                                   200 TRUE       Maximum size of the PGA memory for one process
pga_aggregate_target                             60 TRUE       Target size for the aggregate PGA memory consumed by the instance
```

--logoff and back on

--check current pga session usage

```
select n.NAME, round(t.VALUE/1024/1024,2) size_mb
      from v$mystat t, v$statname n
     where t.STATISTIC# = n.STATISTIC#
       and n.NAME like 'session%memory%'
     order by name;

NAME                                                                SIZE_MB
---------------------------------------------------------------- ----------
session pga memory                                                     2.48
session pga memory max                                                21.66
session uga memory                                                     1.68
session uga memory max                                                 1.68
```

--run the query

```
DECLARE
  v_loc_from VARCHAR2(10) := 1 ;
  v_loc_to   VARCHAR2(10) := 10;
  v_xml_doc CLOB;
BEGIN
   SELECT xmlserialize(DOCUMENT xml_inc_root AS CLOB NO indent) AS xmltext
      INTO v_xml_doc
      FROM
        (SELECT xmlroot (xml_qry.xml_tag, VERSION '1.0" encoding="utf-8') xml_inc_root
        FROM
          (SELECT XMLELEMENT("Stocks", xmlagg( XMLELEMENT("Record", XMLFOREST("ProductID", "LocationID", "ValidDay" , "Timestamp" , "Type", "UnitID" , "Quantity")))) xml_tag
          FROM
            (SELECT "ProductID",
              "LocationID",
              "ValidDay",
              "Timestamp",
              "Type",
              "UnitID",
              "Quantity"
            FROM
              (SELECT "ProductID",
                "LocationID",
                "ValidDay",
                "Timestamp",
                "Type",
                "UnitID",
                "Quantity" --,
               -- row_number() OVER (PARTITION BY 'X' ORDER BY ROWID ASC) rn
              From system.W_Byndr_Stocks
              where "LocationID" between v_loc_from and v_loc_to
              )
            --WHERE rn BETWEEN v_row_loop_from AND v_row_loop_to
            )
          ) xml_qry
        );
END; 
/   
```

--check new pga session usage

```
select n.NAME, round(t.VALUE/1024/1024,2) size_mb
      from v$mystat t, v$statname n
     where t.STATISTIC# = n.STATISTIC#
       and n.NAME like 'session%memory%'
     order by name;


NAME                                                                SIZE_MB
---------------------------------------------------------------- ----------
session pga memory                                                     2.48
session pga memory max                                               321.66
session uga memory                                                     1.68
session uga memory max                                               319.54
```

If you now increase v_loc_to from 10 to say 50 and repeat the test,  you will see massive memory usage (and swapping occurring probably) – this actually destroyed my VM and the query never finished.

So in 11g the process just eats all of the memory available.


###12c

Same test as above, but 

1) connect as non-SYS user
2) before running the test set PGA_AGGREGATE_LIMIIT:

```
SQL> alter system set pga_aggregate_limit=1G scope=memory;
alter system set pga_aggregate_limit=1G scope=memory
*
ERROR at line 1:
ORA-02097: parameter cannot be modified because specified value is invalid
ORA-00093: pga_aggregate_limit must be between 1282M and 100000G


SQL> alter system set pga_aggregate_limit=1282M scope=memory;

System altered.
```

--show relevant systemwide pga parameters

```
KSPPINM                                    KSPPSTVL KSPPSTDF   KSPPDESC
------------------------------------------ -------- ---------- ----------------------------------------------------------------------
__pga_aggregate_target                          280 FALSE      Current target size for the aggregate PGA memory consumed
_pga_max_size                                   200 TRUE       Maximum size of the PGA memory for one process
pga_aggregate_limit                            1282 TRUE       limit of aggregate PGA memory consumed by the instance
pga_aggregate_target                              0 TRUE       Target size for the aggregate PGA memory consumed by the instance
```

--now run the test again
--in another session run

```
set linesize 200 pages 1000
col osuser for a20 wrap
col USERNAME for a20 wrap
col terminal for a15 wrap
col program for a20 wrap
col module for a20 wrap
col last_act for a15
col sess_inf for a15 heading "INST:(SID,SER#)"
col user_inf for a15 heading "USER (OSUSER)"
col prog_inf for a15 heading "MODULE (PROGRAM)"
col os_pid for a10
col wait_event for a28
col PGA_MG for 99999

SELECT  s.inst_id||': ('||s.sid||','||s.serial#||')' sess_inf
        ,s.username||' ('||nvl(s.osuser,'Unknown')||')' user_inf
        --,nvl(s.TYPE,'Unknown') TYPE
        --,nvl(S.MODULE,'Unknown')||' ('||nvl(S.PROGRAM,'Unknown')||')' prog_inf
        ,nvl(substr(S.MODULE,1,15),'Unknown') prog_inf
        ,p.spid OS_PID
        ,nvl(S.STATUS,'Unknown') STATUS
        ,to_char((sysdate - S.last_call_et / 86400),'DD-MM-RR HH24:MI') LAST_ACTIVITY
        ,round(S.last_call_et / 3600, 2) hrs_ago
        ,nvl(s.SQL_ID,'Unknown') SQL_ID
        ,nvl(s.event,'Unknown') wait_event
        ,s.seconds_in_wait SEC_WT
        ,round(t.VALUE/1024/1024,2) PGA_MB
FROM    GV$SESSION S
        ,GV$PROCESS P
        ,GV$SESSTAT T
        ,V$STATNAME N
WHERE   S.USERNAME <> sys_context('USERENV','SESSION_USERID')
AND     P.ADDR = S.PADDR
AND     P.INST_ID = S.INST_ID
AND     S.STATUS = 'ACTIVE'
AND     S.INST_ID = T.INST_ID
AND     S.SID = T.SID
AND     T.STATISTIC# = N.STATISTIC#
AND     N.NAME='session pga memory'
--AND   S.TYPE = 'USER'
--AND   round(S.last_call_et / 3600, 2) > 8
order by 7 DESC;
```

And you will see the PGA_MB rise, and rise, and rise...

```
INST:(SID,SER#) USER (OSUSER)   MODULE (PROGRAM OS_PID     STATUS   LAST_ACTIVITY     HRS_AGO SQL_ID        WAIT_EVENT                       SEC_WT     PGA_MB
--------------- --------------- --------------- ---------- -------- -------------- ---------- ------------- ---------------------------- ---------- ----------
1: (60,12655)   SYSTEM (oracle) SQL*Plus        3056       ACTIVE   17-12-15 05:21        .01 at3s6amwg91pp db file sequential read               0       4.42
1: (55,18731)   SYS (oracle)    sqlplus@localho 3266       ACTIVE   17-12-15 05:22          0 ag95rgbqryd5r SQL*Net message to client             0       1.48

INST:(SID,SER#) USER (OSUSER)   MODULE (PROGRAM OS_PID     STATUS   LAST_ACTIVITY     HRS_AGO SQL_ID        WAIT_EVENT                       SEC_WT     PGA_MB
--------------- --------------- --------------- ---------- -------- -------------- ---------- ------------- ---------------------------- ---------- ----------
1: (60,12655)   SYSTEM (oracle) SQL*Plus        3056       ACTIVE   17-12-15 05:21        .05 at3s6amwg91pp db file sequential read               0     558.54
1: (55,18731)   SYS (oracle)    sqlplus@localho 3266       ACTIVE   17-12-15 05:24          0 ag95rgbqryd5r SQL*Net message to client             0       1.48

..snip...

INST:(SID,SER#) USER (OSUSER)   MODULE (PROGRAM OS_PID     STATUS   LAST_ACTIVITY     HRS_AGO SQL_ID        WAIT_EVENT                       SEC_WT     PGA_MB
--------------- --------------- --------------- ---------- -------- -------------- ---------- ------------- ---------------------------- ---------- ----------
1: (60,12655)   SYSTEM (oracle) SQL*Plus        3056       ACTIVE   17-12-15 05:21        .22 at3s6amwg91pp acknowledge over PGA limit           69    1199.54
1: (44,29847)   MYDBA (oracle)  Unknown         3316       ACTIVE   17-12-15 05:32        .04 417vkyxgfhqh8 null event                            0       1.11
1: (55,18731)   SYS (oracle)    sqlplus@localho 3266       ACTIVE   17-12-15 05:34          0 ag95rgbqryd5r SQL*Net message to client             0       1.73
```

Eventually in the session running the SQL:

```
SQL> DECLARE
  v_loc_from VARCHAR2(10) := 1 ;
  v_loc_to   VARCHAR2(10) := 20 ;
  v_xml_doc CLOB;
BEGIN
   SELECT xmlserialize(DOCUMENT xml_inc_root AS CLOB NO indent) AS xmltext
      INTO v_xml_doc
      FROM
        (SELECT xmlroot (xml_qry.xml_tag, VERSION '1.0" encoding="utf-8') xml_inc_root
        FROM
          (SELECT XMLELEMENT("Stocks", xmlagg( XMLELEMENT("Record", XMLFOREST("ProductID", "LocationID", "ValidDay" , "Timestamp" , "Type", "UnitID" , "Quantity")))) xml_tag
          FROM
            (SELECT "ProductID",
              "LocationID",
              "ValidDay",
              "Timestamp",
              "Type",
              "UnitID",
              "Quantity"
            FROM
              (SELECT "ProductID",
                "LocationID",
                "ValidDay",
                "Timestamp",
                "Type",
                "UnitID",
                "Quantity" --,
               -- row_number() OVER (PARTITION BY 'X' ORDER BY ROWID ASC) rn
              From system.W_Byndr_Stocks
              where "LocationID" between v_loc_from and v_loc_to
              )
            --WHERE rn BETWEEN v_row_loop_from AND v_row_loop_to
            )
          ) xml_qry
        );
END;
/     2    3    4    5    6    7    8    9   10   11   12   13   14   15   16   17   18   19   20   21   22   23   24   25   26   27   28   29   30   31   32   33   34   35   36   37
ERROR:
ORA-03113: end-of-file on communication channel
Process ID: 3056
Session ID: 60 Serial number: 12655
```

And in the alert.log:

```
Thu Dec 17 05:37:17 2015
PGA memory used by the instance exceeds PGA_AGGREGATE_LIMIT of 1282 MB
Immediate Kill Session#: 60, Serial#: 12655
Immediate Kill Session: sess: 0x90ac97b8  OS pid: 3056
```

Good stuff.

But when I run the test connected as SYS, the session is not killed. Instead I receive this message in the alert.log

```
Thu Dec 17 04:05:04 2015
PGA_AGGREGATE_LIMIT has been exceeded but some processes using the most PGA
memory are not eligible to receive ORA-4036 interrupts.  Further occurrences
of this condition will be written to the trace file of the CKPT process.
```

PS - if you couldn't get on during the issue to confirm which sessions/sql_id were the biggest PGA consumers then this query will help:

```
select  to_char(sample_time,'DD-MON-YY HH24:MI') hhmm
        ,instance_number
        ,session_id
        ,session_serial#
        ,sql_id
        ,program
        ,round((max(pga_allocated)/1024/1024),2) PGA_MB
from    dba_hist_active_sess_history
where   sample_time between to_date('11-DEC-2015 19:00','DD-MON-YYYY HH24:MI') and to_date('11-DEC-2015 22:00','DD-MON-YYYY HH24:MI')
and     instance_number=1
group by to_char(sample_time,'DD-MON-YY HH24:MI')
        ,instance_number
        ,session_id
        ,session_serial#
        ,sql_id
        ,program
having max(pga_allocated) > (1024*1024*500)
order by 1;

HHMM    INSTANCE_NUMBER     SESSION_ID     SESSION_SERIAL#     SQL_ID     PROGRAM     PGA_MB
11-DEC-15 19:23     1     1052     4607     79zhfdtsbsppy     oracle@pgx0db01.unix.morrisons.net (J003)     1804.47
11-DEC-15 19:23     1     1440     1823     79zhfdtsbsppy     oracle@pgx0db01.unix.morrisons.net (J004)     1799.54
11-DEC-15 19:23     1     1506     6831     79zhfdtsbsppy     oracle@pgx0db01.unix.morrisons.net (J005)     1827.22
11-DEC-15 19:24     1     851     339     79zhfdtsbsppy     oracle@pgx0db01.unix.morrisons.net (J000)     2150.35
11-DEC-15 19:24     1     920     2277     79zhfdtsbsppy     oracle@pgx0db01.unix.morrisons.net (J001)     2978.47
...snip...
```
