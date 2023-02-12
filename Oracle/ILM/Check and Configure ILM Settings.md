# Abstract

Diese Seite beschreibt wie man die Parametersettings für das Heat Map Feature anzeigt und verändert.

# Selektieren

```bash
SQL> SELECT name, value FROM  dba_ilmparameters ORDER BY name;

NAME                      VALUE
-------------------- ----------
ENABLED                       1
EXECUTION INTERVAL           15
EXECUTION MODE                2
JOB LIMIT                     2
POLICY TIME                   0
RETENTION TIME               60
TBS PERCENT FREE             25
TBS PERCENT USED             85

8 rows selected.
```

# Ändern

```bash
begin
dbms_ilm_admin.customize_ilm(dbms_ilm_admin.retention_time, 90);
end;
/

SQL> SELECT name, value FROM  dba_ilmparameters ORDER BY name;

NAME                      VALUE
-------------------- ----------
ENABLED                       1
EXECUTION INTERVAL           15
EXECUTION MODE                2
JOB LIMIT                     2
POLICY TIME                   0
RETENTION TIME               90
TBS PERCENT FREE             25
TBS PERCENT USED             85

8 rows selected.
```

# Referenz

[Link](https://oracle-base.com/articles/12c/heat-map-ilm-ado-12cr2) zur Anleitung auf [oracle-base.com](https://oracle-base.com/).

Tags:

[[ILM]] - [[HowTo]] - [[OracleBase]] - [[HeatMap]]
