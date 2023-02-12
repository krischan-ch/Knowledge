
Es gibt verschiedene Parameter, die das Sammeln der Objekt-Statistiken beeinflussen:

| Parameter/Option | Beschreibung |
| --- | --- |
| Genauigkeit (ESTIMATE_PERCENT) | Emfehlung: DBMS_STATS.AUTO_SAMPLE_SIZE (=Default) |
| Histogramme (METHOD_OPT) | Emfehlung: FOR ALL COLUMNS SIZE AUTO (=Default), ausser für spezielle Tabellen |
| Parallelität (DEGREE) | Emfehlung: NULL (=Default), ausser für grosse, partitionierte Tabellen (DBMS_STATS.AUTO_DEGREE, Einstellung auf Tabellen-Ebene) |
| Gleichzeitigkeit (CONCURRENT) | Emfehlung: kleine DB's: OFF (=Default), DB's mit viel Änderungen: AUTOMATIC |

Die Parameter können auch kombiniert werden und beziehen sich sowohl auf den manuellen als auch auf den automatischen Gather-Stats-Job.

Abfrage der aktuellen globalen Einstellung, Bsp:  

```sql
SQL> select dbms_stats.get_prefs ('CONCURRENT') from dual;

DBMS_STATS.GET_PREFS('CONCURRENT')
--------------------------------------------------------------------------------
OFF
```

Setzen eines Parameters global auf DB-Ebene, Bsp:   

```sql
SQL> exec dbms_stats.set_global_prefs ('CONCURRENT','AUTOMATIC');
```

Concurrent and/or Parallel Statistics Gathering

Concurrent Statistics Gathering rechnet Statistiken gleichzeitig auf mehreren (Sub-)Objekten, Parallel verwendet dagegen parallele Jobs pro Objekt. Sehr gut erklärt ist es in diesen Blogeinträgen:

[Nigel Bayliss: How to Gather Optimizer Statistics Fast](https://blogs.oracle.com/optimizer/post/how-to-gather-optimizer-statistics-fast)

Concurrent Statistics Gathering

Der Concurrent-Modus lohnt sich vor allem dann, wenn das Zeitfenster für das nächtliche Statistik-Sammeln nicht ausreicht und man noch genügend freie CPU-Ressourcen hat.

Abfrage der letzten langen Auto-Statistik-Läufe:   

```sql
set lines 200 pages 100
col WINDOW_NAME for a18
col JOB_NAME for a24
col JOB_STATUS for a10
col JOB_START_TIME for a42
col JOB_DURATION for a14
col JOB_INFO for a64

select WINDOW_NAME, JOB_NAME, JOB_STATUS, JOB_START_TIME, JOB_DURATION, JOB_ERROR, JOB_INFO
from DBA_AUTOTASK_JOB_HISTORY
where CLIENT_NAME = 'auto optimizer stats collection'
and WINDOW_START_TIME >= sysdate-20
and JOB_DURATION > NUMTODSINTERVAL(0.5, 'hour')
order by JOB_START_TIME asc;

WINDOW_NAME        JOB_NAME                 JOB_STATUS JOB_START_TIME                             JOB_DURATION    JOB_ERROR JOB_INFO
------------------ ------------------------ ---------- ------------------------------------------ -------------- ---------- ----------------------------------------------------------------

WEDNESDAY_WINDOW   ORA$AT_OS_OPT_SY_71092   STOPPED    24.10.18 23:40:03'307915 AFRICA/TUNIS      +000 04:00:00           0 REASON="Stop job called because associated window was closed"

THURSDAY_WINDOW    ORA$AT_OS_OPT_SY_71112   STOPPED    25.10.18 23:40:03'898285 AFRICA/TUNIS      +000 04:00:00           0 REASON="Stop job called because associated window was closed"

FRIDAY_WINDOW      ORA$AT_OS_OPT_SY_71132   STOPPED    26.10.18 23:40:03'861677 AFRICA/TUNIS      +000 04:00:01           0 REASON="Stop job called because associated window was closed"

SATURDAY_WINDOW    ORA$AT_OS_OPT_SY_71134   SUCCEEDED  27.10.18 06:00:04'053733 AFRICA/TUNIS      +000 07:13:21           0

 MONDAY_WINDOW      ORA$AT_OS_OPT_SY_71232   STOPPED    05.11.18 23:40:03'999297 AFRICA/TUNIS      +000 03:59:58           0 REASON="Stop job called because associated window was closed"

TUESDAY_WINDOW     ORA$AT_OS_OPT_SY_71234   STOPPED    06.11.18 23:40:03'265678 AFRICA/TUNIS      +000 03:59:58           0 REASON="Stop job called because associated window was closed"

WEDNESDAY_WINDOW   ORA$AT_OS_OPT_SY_71236   SUCCEEDED  07.11.18 23:40:03'641060 AFRICA/TUNIS      +000 02:56:06           0
```

**Objektliste**

Die Liste der Objekte kann entsprechend eingeschränkt werden. Auch dazu gibt's einen super Blogeintrag:

[Maria Colgan: How do I restrict concurrent statistics gathering to a small set of tables from a single schema?](https://blogs.oracle.com/optimizer/post/how-do-i-restrict-concurrent-statistics-gathering-to-a-small-set-of-tables-from-a-single-schema)



Tags:

[[HowTo]] - [[MariaColgan]] - [[NigelBayliss]] - [[GatherStats]] - [[PerformanceTuning]]