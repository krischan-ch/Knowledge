Mit folgendem Skript wird ein Logon-Trigger erstellt, der das Tracing für den User BFEUSER aktiviert. Für jede Session, die nach der Erstellung des Triggers geöffnet wird, ist das Tracing aktiviert.

```sql
create or replace trigger logon_trigger_sql_trace after logon on database
declare
  v_user dba_users.username%TYPE:=user;
begin
  if (v_user='BFEUSER') then
    execute immediate ('alter session set events ''10046 trace name context  forever, level 12''');
end if;

if (v_user='PLAFOUSER') then
    execute immediate ('alter session set events ''10046 trace name context forever, level 12''');
end if;
end;
/

```

### Tags:

[[HowTo]] - [[PerformanceTuning]]