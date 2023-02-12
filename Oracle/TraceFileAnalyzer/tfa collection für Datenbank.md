AHF - Autonomous Health Framework checks and Diagnostics

Autonomous Health Framework ( AHF ) - Including TFA and ORAchk/EXAChk (Doc ID 2550798.1)

# Trace File Analyzer

Der Trace File Analyser kann unter Anderem Tracedateien für RAC Cluster und deren Datenbanken sammeln. Der Oracle Support fragt bei Service Requests die Exadata- bzw. RAC-Datenbanken betreffen nach einer tfa Collection.

Diese Daten können mit dem tfactl Utility gesammelt werden. Das tfactl Utility befindet sich im ORACLE_HOME der Grid Infrastructure und je nach dem auch im $ORACLE_HOME der einzelnen DBs.

# Daten sammeln

Mit folgendem Kommando können für die EAI01G Datenbank die Tracefiles der letzten fünf Stunden gesammelt werden. Der Zeitrahmen definiert sich hierbei über den Zeitpunkt, zu dem der Fehler aufgetreten ist, für den der SR geöffnet wurde.

Manche Kommandos werden als root-User ausgeführt, da nur dieser Berechtigungen auf alle Tracefiles hat (Trennung User grid & oracle).​

| Task | Syntax |
| --- | --- |
| BasEnv sourcen | ​. /usr/local/bin/basenv.zkb |
| Environment setzen | ​+ASM1 |
| tfa starten | ​tfactl diagcollect -database EAI01G -last 5h |
| ​tfa für ORA-600 sammeln (als user Oracle) | ​$ORACLE_HOME/suptools/tfa/release/tfa_home/bin/tfactl diagcollect  -database TMS61 -srdc ORA-00600​ |

​tfactl sammelt Diagnosticdaten aller im Cluster vorhandenen Server und speichert die gepackten Dateien im tfa Repository. Die Lokation der gepackten Dateien werden am Ende angezeigt. Diese Files werden dann zum Oracle Support hochgeladebn.

# Referenzen

[Autonomous Health Framework (AHF) - Including TFA and ORAchk/EXAchk (Doc ID 2550798.1)](https://support.oracle.com/epmos/faces/DocContentDisplay?_afrLoop=76330728186368&id=2550798.1)

[Procwatcher: Script to Monitor and Examine Oracle DB and Clusterware Processes (Doc ID 459694.1)](https://support.oracle.com/epmos/faces/DocContentDisplay?_afrLoop=76418092328698&id=459694.1)

[OSWatcher (Includes: [Video]) (Doc ID 301137.1)](https://support.oracle.com/epmos/faces/DocContentDisplay?_afrLoop=76492467428116&id=301137.1)

[OS Watcher User's Guide (Doc ID 1531223.1)](https://support.oracle.com/epmos/faces/DocumentDisplay?_afrLoop=76665270424576&id=1531223.1)

[OSWatcher Analyzer User Guide (Doc ID 461053.1)](https://support.oracle.com/epmos/faces/DocContentDisplay?_afrLoop=76709997091382&id=461053.1)

[Best Practices: Proactively Avoiding Database and Query Performance Issues (Doc ID 1482811.1)](https://support.oracle.com/epmos/faces/DocContentDisplay?_afrLoop=76762015987345&id=1482811.1)
[Best Practices: Proactive Data Collection for Performance Issues (Doc ID 1477599.1)](https://support.oracle.com/epmos/faces/DocContentDisplay?_afrLoop=76901094978531&id=1477599.1)

[oratop - Utility for Near Real-time Monitoring of Databases, RAC and Single Instance (Doc ID 1500864.1)](https://support.oracle.com/epmos/faces/DocContentDisplay?_afrLoop=76947073973674&id=1500864.1)


### Tags:

[[HowTo]] - [[PerformanceTuning]] - [[tfa]] - [[ahf]]