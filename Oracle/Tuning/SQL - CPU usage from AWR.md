
# Abstract

SQL zum Abfragen von AWR-Tabellen nach CPU Zeit u.ä.

# SQL

```sql
REM Name:  awr-cpu.sql
REM Purpose:  Retrieve CPU-related metrics from AWR
REM Usage:  From DB Instance, SQL> @lst05-01-awr-cpu.sql

set arraysize 5000
set termout on
set echo off verify off
set lines 290
set pages 900
col id format 99999 head 'Snap|ID'
col tm format a15 head 'Snap|End'
col instances format 999 head 'RAC|Nodes'
col dur format 999.99 head 'Duration|Mins'
col CPUs format 999 head 'CPUs'
col cap         format 9999990.00       head 'Tot CPU Time|Avail (s)'
col dbt         format 999990.00        head "DB|Time"
col dbc         format 99990.00         head "DB|CPU"
col bgc         format 99990.00         head "Bg|CPU"
col rmanc       format 99990.00         head "RMAN|CPU"
col aas         format 999.0            head 'AAS'
col totora      format 9999990.0        head 'Tot Ora|CPU (s)'
col load        format 990.0            head 'Tot OS|Load'
col totos       format 9999990.0        head 'Tot OS|CPU (s)'
col oracpupct   format 990.0            head 'Ora|CPU%'
col rmancpupct  format 990.0            head 'RMAN|CPU%'
col oscpupct    format 990.00           head 'OS|CPU%'
col usrcpupct    format 990.00          head 'Usr|CPU%'
col syscpupct    format 990.00          head 'Sys|CPU%'
col iowaitcpupct format 990.00          head 'IOWait|CPU%'
col logons_curr format 999990.0         head 'Logons|Current'
col logons_cum  format 990.0            head 'Logons|Per Sec'
col execs       format 9999990.0           head 'Execs|Per Sec'

set colsep "|"
set echo on

select  [snaps.id](http://snaps.id/), snaps.tm,snaps.dur,snaps.instances,
        osstat.num_cpus CPUs,
        osstat.num_cpus * dur * 60 cap,
        ((timemodel.dbt - lag(timemodel.dbt,1,0) over (order by [snaps.id](http://snaps.id/))))/1000000 dbt,
        ((timemodel.dbc - lag(timemodel.dbc,1,0) over (order by [snaps.id](http://snaps.id/))))/1000000 dbc,
        ((timemodel.bgc - lag(timemodel.bgc,1,0) over (order by [snaps.id](http://snaps.id/))))/1000000 bgc,
        ((timemodel.rmanc - lag(timemodel.rmanc,1,0) over (order by [snaps.id](http://snaps.id/))))/1000000 rmanc,
        (((timemodel.dbt - lag(timemodel.dbt,1,0) over (order by [snaps.id](http://snaps.id/))))/1000000)/dur/60 aas ,
        (((timemodel.dbc - lag(timemodel.dbc,1,0) over (order by [snaps.id](http://snaps.id/))))/1000000) +
          (((timemodel.bgc - lag(timemodel.bgc,1,0) over (order by [snaps.id](http://snaps.id/))))/1000000) totora ,
        osstat.load load        ,
        ((osstat.busy_time - lag(osstat.busy_time,1,0) over (order by [snaps.id](http://snaps.id/))))/100 totos,
        round(100*(((((timemodel.dbc - lag(timemodel.dbc,1,0) over (order by [snaps.id](http://snaps.id/))))/1000000) +    -- (DB_CPU + BG_CPU) / (CPUs * 60 seconds to Mins)
          (((timemodel.bgc - lag(timemodel.bgc,1,0) over (order by [snaps.id](http://snaps.id/))))/1000000)) /
                (osstat.num_cpus * 60 * dur)),2) oracpupct,
        round(100*((((timemodel.rmanc - lag(timemodel.rmanc,1,0) over (order by [snaps.id](http://snaps.id/))))/1000000) / -- (RMAN_CPU) / (CPUs * 60 seconds to Mins)
                (osstat.num_cpus * 60 * dur)),2) rmancpupct,
        round(100*((((osstat.busy_time - lag(osstat.busy_time,1,0) over (order by [snaps.id](http://snaps.id/))))/100) /
                (osstat.num_cpus * 60 * dur)),3) oscpupct,
        round(100*((((osstat.user_time - lag(osstat.user_time,1,0) over (order by [snaps.id](http://snaps.id/))))/100) /
                (osstat.num_cpus * 60 * dur)),3) usrcpupct,
        round(100*((((osstat.sys_time - lag(osstat.sys_time,1,0) over (order by [snaps.id](http://snaps.id/))))/100) /
                (osstat.num_cpus * 60 * dur)),3) syscpupct,
        round(100*((((osstat.iowait_time - lag(osstat.iowait_time,1,0) over (order by [snaps.id](http://snaps.id/))))/100) /
                (osstat.num_cpus * 60 * dur)),3) iowaitcpupct,
        sysstat.logons_curr ,
        ((sysstat.logons_cum - lag (sysstat.logons_cum,1,0) over (order by [snaps.id](http://snaps.id/))))/dur/60  logons_cum,
        ((sysstat.execs - lag (sysstat.execs,1,0) over (order by [snaps.id](http://snaps.id/))))/dur/60 execs
from
( /* DBA_HIST_SNAPSHOT */
select distinct id,dbid,tm,instances,max(dur) over (partition by id) dur from (
select distinct s.snap_id id, s.dbid,
   to_char(s.end_interval_time,'DD-MON-RR HH24:MI') tm,
    count(s.instance_number) over (partition by snap_id) instances,
    1440*((cast(s.end_interval_time as date) - lag(cast(s.end_interval_time as date),1) over (order by s.snap_id))) dur
from   dba_hist_snapshot s,
    v$database d
where s.dbid=d.dbid)
) snaps,
( /* Data from DBA_HIST_OSSTAT */
  select  *
        from
        (select snap_id,dbid,stat_name,value from
        dba_hist_osstat
  ) pivot
  (sum(value) for (stat_name)
        in ('NUM_CPUS' as num_cpus,'BUSY_TIME' as busy_time,
            'LOAD' as load,'USER_TIME' as user_time, 'SYS_TIME' as sys_time, 'IOWAIT_TIME' as iowait_time))
  ) osstat,
  ( /* DBA_HIST_TIME_MODEL */
   select * from
        (select snap_id,dbid,stat_name,value from
        dba_hist_sys_time_model
   ) pivot
   (sum(value) for (stat_name)
        in ('DB time' as dbt, 'DB CPU' as dbc, 'background cpu time' as bgc,
             'RMAN cpu time (backup/restore)' as rmanc))
  ) timemodel,
  ( /* DBA_HIST_SYSSTAT */
    select * from
        (select snap_id, dbid, stat_name, value from
        dba_hist_sysstat
    ) pivot
    (sum(value) for (stat_name) in
        ('logons current' as logons_curr, 'logons cumulative' as logons_cum, 'execute count' as execs))
  ) sysstat
where dur > 0
and [snaps.id](http://snaps.id/)=osstat.snap_id
and snaps.dbid=osstat.dbid
and [snaps.id](http://snaps.id/)=timemodel.snap_id
and snaps.dbid=timemodel.dbid
and [snaps.id](http://snaps.id/)=sysstat.snap_id
and snaps.dbid=sysstat.dbid
order by id asc
/
```

Credits: Nenad Gajic

### Tags_

[[HowTo]] - [[AWR]] - [[PerfortmanceTuning]] - [[NenadGajic]]