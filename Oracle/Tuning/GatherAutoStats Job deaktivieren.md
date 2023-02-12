
# Anzeigen aller Auto Jobs:

```sql
set lines 200 pages 100
col CLIENT_NAME for a40
col CONSUMER_GROUP for a20
col WINDOW_GROUP for a20
col MEAN_JOB_DURATION for a40
col MEAN_JOB_CPU for a30
col LAST_CHANGE for a35

select CLIENT_NAME, STATUS, CONSUMER_GROUP, WINDOW_GROUP, MEAN_JOB_DURATION, MEAN_JOB_CPU, LAST_CHANGE from dba_autotask_client order by 1;
```

# Deaktivieren des Statistik Jobs

```sql
BEGIN
  DBMS_AUTO_TASK_ADMIN.DISABLE (
     client_name => 'auto optimizer stats collection',
     operation   => NULL,
     window_name => NULL
  );
END;
/
```

# Aktivieren des Statistik Jobs

```sql
BEGIN
  DBMS_AUTO_TASK_ADMIN.ENABLE (
     client_name => 'auto optimizer stats collection',
     operation   => NULL,
     window_name => NULL
  );
END;
/
```

Tags:  
[[DDL]] - [[GatherStats]] - [[OEM]] - [[HowTo]] - [[PerformanceTuning]]
