# Abstract

Mit folgender Syntax kann der Oracle-eigene gather_table_stats Job so eingestellt werden, das er nur für Oracle-eigene Schemata arbeitet.
Applikationsobjekte werden nicht mehr gerechnet.

```sql
begin
  dbms_stats.set_global_prefs('autostats_target','oracle');
end;
/
```

Tags:

[[OEM]] - [[EnterpriseManager]] - [[GatherStats]] - [[HowTo]] - [[PerformanceTuning]]
