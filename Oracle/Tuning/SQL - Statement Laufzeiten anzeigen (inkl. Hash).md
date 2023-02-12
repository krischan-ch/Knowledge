
# Abstract

Diese Statement selektiert die Laufzeiten für ein SQL Statement aus den AWR Historie Tabellen.

Bei der Ausführung wird die entsprechende SQL_ID abgefragt.

# Statement

```sql
SELECT stat.sql_id
     , stat.plan_hash_value
     , TO_CHAR(snap.end_interval_time, 'dd.mm.yyyy') exec_date
     , ROUND(SUM(elapsed_time_delta)/1000000) elapsed
     , ROUND(SUM(cpu_time_delta)/1000000) cputime
     , ROUND(SUM(iowait_delta)/1000000) iotime
     , SUM(stat.rows_processed_delta) num_rows
     , SUM(stat.executions_delta) execs
     , ROUND(SUM(elapsed_time_delta)/GREATEST(SUM(stat.executions_delta), 1)/1000000, 2) sec_per_exec
  FROM dba_hist_sqlstat stat
  JOIN dba_hist_snapshot snap
    ON snap.dbid            = stat.dbid
   AND snap.instance_number = stat.instance_number
   AND snap.snap_id         = stat.snap_id
WHERE stat.sql_id          = '&sql_id'
   AND stat.elapsed_time_delta > 0
GROUP BY stat.sql_id
    , stat.plan_hash_value
    , TO_CHAR(snap.end_interval_time, 'dd.mm.yyyy')
ORDER BY TO_DATE(exec_date, 'dd.mm.yyyy');​
```

### Tags:

[[HowTo]] - [[PerfortmanceTuning]] 