
Unter "Automatic Data Optimization" fasst Oracle verschiedene neue Features zusammen, mit denen Daten, basierend auf Policies, automatisch komprimiert und bewegt werden können.

Der Grundgedanke hierbei ist, das Daten, welche nicht oder zumindest nicht regelmässig bearbeitet werden komprimiert und evtl. auf einen günstigeren Storage ausgelagert werden. Dies kann mittels Policies, die auf Tabellenebene definiert werden, automatisch durchgeführt werden.

# Heat Map

Zentrales Feature hierbei ist die Heat-Map, welches Statistikdaten bez. der Verwendung der Daten sammelt.

| aktivieren | deaktivieren |
| --- | --- |
| alter system set heat_map = on scope=both; | alter system set heat_map = off scope=both; |

# Automatische Komprimierung

## Segment-Level

In folgendem Beispiel wird eine Segment-Level-Policy definiert, welche Partitionen, deren Daten länger als 30 Tage nicht verändert wurden, mit der Option "row store compress advanced" komprimiert:

```sql
ALTER TABLE orders ILM ADD POLICY
ROW STORE COMPRESS ADVANCED SEGMENT
AFTER 30 DAYS OF NO MODIFICATION;
```

## Row-Level

Das Beispiel oben kann so ähnlich auch auf einzelne Datenblöcke angewendet werden, deren Rows seit 3 Tagen nicht mehr geändert wurden:

```sql
ALTER TABLE orders ILM ADD POLICY
ROW STORE COMPRESS ADVANCED ROW
AFTER 3 DAYS OF NO MODIFICATION;
```

Row-Level bezieht sich entgegen dem Namen aber tatsächlich auf Block-Ebene. Werden die Datensätze in einem Block länger als 3 Tage nicht bearbeitet, werden sie komprimiert.

## Automatisches Storage-Tiering

ADO bietet weiterhin die Möglichkeit Daten, basierend auf Policies, automatisch auf ein anderes Tablespace zu verschieben. Ein Anwendungsfall wäre hierbei das Verschieben und ggf. gleichzeitiges Komprimieren auf einen günstigeren Storage (Exadata Storage Cell vs. ZFS Storage Appliance), um Kosten zu sparen.

Folgendes Kommando verschiebt die Tabelle orders auf das Tablespace low_cost_store, wenn der freie Platz des aktuellen Tablespaces einen festgelegten Schwellwert unterschreitet:

```sql
ALTER TABLE orders ILM ADD POLICY tier to low_cost_store;
```

Die Schwellwerte für diese Funktionen können mit Hilfe des Packages dbms_ilm_admin verwaltet werden. Folgende Kommando's setzen den Schwellwert für belegten bzw. freien Platz auf 95% bzw. 5%:

```sql
exec dbms_ilm_admin.customize_ilm(DBMS_ILM_ADMIN.TBS_PERCENT_FREE,95)
exec dbms_ilm_admin.customize_ilm(DBMS_ILM_ADMIN.TBS_PERCENT_USED,5)
```

# Tags:

[[HowTo]] - [[NewFeatures]]
