> (am Beispiel von ZK's EAI DB)

# Ablage

Die Files die in dieser Seite genannt werden befinden sich auf der exa301s01 im Verzeichnis /zfssa/oraddl/EAI01/stats/synopsis/

Alle Angaben auf dieser Seite beziehen sich auf die EAI01.eng Datenbank.

# SYSAUX Tablespace erweitern

Der initiale Lauf sammelt Synopsis Informationen zu den Daten in den Tabellen. Auf der EAI01.prod wurden dazu ca. 550GB Daten im SYSAUX Tablespace abgelegt.

Das SYSAUX Tablespace muss also mind. 600GB freien Speicherlatz aufweisen.

# Aktivieren von inkrementellen Statistiken

Ablage:  /zfssa/oraddl/EAI01/stats/synopsis/activate_incr_stats.log

Inhalt:

```sql
@crspool activate_incr_stats.log
set heading on echo on verify on feedback on trimspool on
exec DBMS_STATS.SET_SCHEMA_PREFS (ownname=>'ODSH1', pname=>'INCREMENTAL', pvalue=>'TRUE');
exec DBMS_STATS.SET_SCHEMA_PREFS (ownname=>'ODSH1', pname=>'INCREMENTAL_STALENESS', pvalue=>'use_stale_percent');
exec DBMS_STATS.SET_SCHEMA_PREFS (ownname=>'ODSH1', pname=>'ESTIMATE_PERCENT', pvalue=>dbms_stats.auto_sample_size);
exec DBMS_STATS.SET_SCHEMA_PREFS (ownname=>'ODSH1', pname=>'GRANULARITY', pvalue=>'ALL');
spool
spool off

# SQL File generieren

sqlplus / as sysdba <<EOSQL
spool gather_stats.sql
select 'exec dbms_stats.gather_table_stats(ownname=>''ODSH1'',tabname=>'''|| table_name || ''',degree=>8);' from dba_tables where owner='ODSH1';
spool off
EOSQL
```

# Aufteilen auf mehrere Files

Wenn mehr als eine Datenbankinstanz zur Verfügung steht kann die Arbeitslast auf alle Instanzen verteilt werden. Dazu kann der split Befehl verwendet werden.

Anzahl ermitteln

```sql
grep gather_table_stats gather_stats.sql | wc -l
```

Die Anzahl der Zeilen sollte gleichmässig auf die Files aufgeteilt werden. In diesem Fall sollen vier Files erzeugt werden.

# Files aufteilen

```bash
split -l 214 -d gather_stats.sql gather_odsh1_stats_

​-l 214   definiert die Anzahl pro File
​-d         ​numerisches Suffix verwenden anstatt eines alphabetisch
​gather_stats.sql          ​Source-Filename
​gather_odsh1_stats_  ​zu verwendendes Präfix

Spool Kommandos einfügen und Endung anhängen

for _FILE in gather_odsh1_stats*
do
  sed "1,1s/^/set heading on echo on verify on feedback on trimspool on timing on\n/" ${_FILE} >${_FILE}.work
  sed "1,1s/^/@crspool ${_FILE}.log\n/" ${_FILE}.work >${_FILE}.sql
  echo "spool" >>${_FILE}.sql
  echo "spool off" >>${_FILE}.sql
  rm -f ${_FILE} ${_FILE}.work
done
```

# Skripte ausführen

Die generierten Skripte sollten auf den zur Verfügung stehenden Clusternodes verteilt ausgeführt werden.

Tags:
[[HowTo]] - [[GatherStats]] - [[PerformanceTuning]]

