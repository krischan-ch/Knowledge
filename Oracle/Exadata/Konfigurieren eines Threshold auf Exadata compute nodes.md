
# Abstract

Diese Dokument beschreibt die Erstellung eines Threshold auf einem Exadata compute node. In diesem Beispiel wird ein Schwellwert für Filesystem Usage >25% für das /u01 Filesystem konfiguriert.

# Threshold erstellen

```bash
DBMCLI> create threshold DS_FSUT."/u01" comparison='>', warning=25
Threshold DS_FSUT."/u01" successfully created

DBMCLI> list threshold detail
         name:                   DS_FSUT./u01
         comparison:             >
         warning:                25.0

DBMCLI> DROP THRESHOLD "DS_FSUT./u01"
Threshold DS_FSUT."/u01" successfully dropped
```

Tags:
[[HowTo]] - [[Exadata]] - [[Monitoring]]

