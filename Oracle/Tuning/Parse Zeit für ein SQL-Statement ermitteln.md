# Abstract

Die Parse Zeit für ein SQL-Statement kann aus der View v$sess_time_model selektiert werden.

Der Name der entsprechenden Statistik ist parse time elapsed und die STAT_ID dafür ist 1431595225. Der Wert ist in Microsekunden angegeben.

Beispiel

```sql
set serveroutput on

declare
  v_start number;
  v_ende  number;
  v_diff  number;
begin
  select value into v_start from v$sess_time_model where stat_id=1431595225 and sid=(select distinct sid from v$mystat);
  execute immediate ('select count(1) from STV3686_HOD833X_BAFELESEN_SQL0');

  select value into v_ende from v$sess_time_model where stat_id=1431595225 and sid=(select distinct sid from v$mystat);

  v_diff:=v_ende-v_start;

  dbms_output.put_line('Parse Time (ms): ' || v_diff);
end;
/
```

Die drei Variable v_start, v_ende und v_diff enthalten die Werte für parse time elapsed. Zwischen der Selektion v_start und v_ende wird das Statement ausgeführt, für welches die Parse Time ermittelt werden soll.

Da die View Statistikedaten für mehr als eine Session enthält muss mit Hilfe des folgenden Subselekts die eigenen Session ID selektiert werden:

```sql
select distinct sid from v$mystat;
```

Tags:

[[HowTo]] - [[PerformanceTuning]]