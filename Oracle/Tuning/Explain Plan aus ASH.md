
Explain Plan aus der Active Session History (ASH)

Anzeige der SQL-Plan Hashes und der Anzahl Ausführungen einer SQL-Id aus der Active Session History, gruppiert pro Stunde:

```sql
select to_char(SAMPLE_TIME,'yyyy-mm-dd hh24')||':xx' ASH_TIME,
SQL_PLAN_HASH_VALUE,
count()
from V$ACTIVE_SESSION_HISTORY
where SQL_ID='g9jgnw8bgwmdg'
group by to_char(SAMPLE_TIME,'yyyy-mm-dd hh24')||':xx', SQL_PLAN_HASH_VALUE
order by 1;
```

-> wenn mehrere SQL-Plan-Hashes angezeigt werden, gab es unterschiedliche SQL-Plans dieses SQL's!

Anzeige aller SQL-Pläne einer SQL-Id aus der Active Session History resp. vom AWR-Snaphot:

```sql
select plan_table_output from table (dbms_xplan.display_awr('g9jgnw8bgwmdg'));
```

Anzeige eines spezifischen SQL-Plans einer SQL-Id aus der Active Session History resp. vom AWR-Snaphot mit einem SQL-Plan-Hash:

```sql
select plan_table_output from table (dbms_xplan.display_awr(SQL_ID=>'g9jgnw8bgwmdg', PLAN_HASH_VALUE=>1847408957));
```

Anzeige eines SQL-Plans vom Shared Pool (V$SQL):

```sql
select plan_table_output from table (dbms_xplan.display_cursor('g9jgnw8bgwmdg'));
```

Tags:
[[ASH]] - [[ExplainPlan]] - [[PerformanceTuning]]