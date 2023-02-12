
```sql
SELECT /*+ ordered use_nl(p) */ s.report_id, s.sql_id, s.sql_exec_start, s.duration_sec, s.status, p.pga_mb
FROM (
  SELECT report_id, sql_id, to_date(sql_exec_start,'MM/DD/YYYY HH24:MI:SS') AS sql_exec_start, duration_sec, status
  FROM dba_hist_reports,
       XMLTable('/report_repository_summary/sql' passing XMLType(report_summary)
                COLUMNS sql_id VARCHAR2(13) PATH '@sql_id',
                        sql_exec_start VARCHAR2(19) PATH '@sql_exec_start',
                        duration_sec NUMBER PATH 'stats[@type="monitor"]/stat[@name="duration"]',
                        status VARCHAR2(100) PATH 'status')
  WHERE component_name = 'sqlmonitor'
) s, (
  select /*+ index(DBA_HIST_REPORTS_DETAILS) */ report_id,
         round(max(value/1024),0) AS pga_mb
  from DBA_HIST_REPORTS_DETAILS,
       xmltable('/report/sql_monitor_report/stattype/buckets/bucket/stat[@id=9]'
                passing xmltype(report)
                columns value number path '@value')
  group by report_id
) p
WHERE s.report_id = p.report_id
AND to_date(s.sql_exec_start,'MM/DD/YYYY HH24:MI:SS') BETWEEN to_date('07/25/2017 18:00:00','MM/DD/YYYY HH24:MI:SS') AND to_date('07/26/2017 10:00:00','MM/DD/YYYY HH24:MI:SS')
--AND s.duration_sec >= 300
--AND s.sql_id = '7unxbmh5grv58'
order by 3;
```


### Tags:

[[HowTo]] - [[PerformanceTuning]] - [[PGA]]