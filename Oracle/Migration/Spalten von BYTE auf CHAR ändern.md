
# Abstract

Bei einer Charactersetmigration von Single- auf Multibyte müssen die varchar2 Spalten mit NLS_LENGTH_SEMANTICS=BYTE auf CHAR konvertiert werden.

# Spalten ändern

Das folgende Statement generiert für alle Spalten in denen die NLS_LENGTH_SEMANTICS auf BYTE steht ein "alter table" Statement mit dem die Spalte entsprechend umkonfiguriert werden kann:

​
```sql
select 'alter table '||a.owner||'.'||a.table_name||' modify '||a.column_name||' '||a.data_type ||' ('||a.data_length||
  case when char_used='C' then ' CHAR '
  when char_used='B' then ' CHAR '
  else ' CHAR '
  end || ');'
from dba_tab_columns a ,dba_tables b
  where a.owner in ('FWS30')       and a.data_type like '%CHAR%' and a.table_name = b.table_name ;
```


Nach dem Ändern der Spalten sollte es beim Import der Daten keine Probleme mehr mit Oracle Fehlermeldungen bezüglich zu langen Werten geben.

### Tags:

[[HowTo]] - [[NLS]]
