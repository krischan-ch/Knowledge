
# Generierung des Index DDL mit Compress Klausel

Das folgende Statement generiert für bestimmte Indizes des Users "SIEBDBA" die DDL's und hängt die Compress-Klausel, basierend auf der Anzahl der indizierten Spalten, an das generierte Statement an:

```sql
select
  dbms_metadata.get_ddl('INDEX',index_name,'SIEBDBA') ||
  (select case when count(column_name) >= 2 then 'COMPRESS ' || to_char(count(column_name)-1) else 'NOCOMPRESS' end from dba_ind_columns where index_name=oj.index_name)
from v$object_usage oj
  where index_name is not null and used='YES';
```

Dieses Statement selektiert die Namen der Indizes aus der View v$object_usage. Hier kann aber auch dba_indexes oder jede andere Quelle verwendet werden.

Das Statement generiert für alle Indizes aus der Liste mittels dbms_metadata das DDL und konkateniert es mit dem Output eines select auf die View dba_ind_columns (compress Klausel).

# Tags:

[[HowTo]] - [[ACO]] - [[Compression]]
