
# Abstract
Die hier beschriebenen Code-Fragmente dienen zum Stoppen und Starten aller HAS-Ressourcen auf einem AIX-Server.

# Oracle Homes
Mittels srvctl's stop home Funktionalität werden alle Ressourcen gestoppt die im angegebenen Home laufen. Der Status vor der Ausführung wird in einem Statefile gespeichert.

```bash
for _HOME in $(ls -d /app/oracle/product/*/bin/oracle | sed -e 's-/bin/oracle--')
do
  ${_HOME}/bin/srvctl stop home -oraclehome ${_HOME} -statefile /tmp/has_$(echo ${_HOME} | awk -F\/ '{print $NF}' | sed -e 's/\.//g')_$(hostname -s).state
done
```

Mittels srvctl's start home Funktionalität werden alle Ressourcen des angegebenen Homes anhand des vorher generierten State-File wieder gestartet.

```bash
for _HOME in $(ls -d /app/oracle/product/*/bin/oracle | sed -e 's-/bin/oracle--')
do
  ${_HOME}/bin/srvctl start home -oraclehome ${_HOME} -statefile /tmp/has_$(echo ${_HOME} | awk -F\/ '{print $NF}' | sed -e 's/\.//g')_$(hostname -s).state
done
```

# High Availability Services
Die übrigen HAS-Prozesse, welche nach dem stop home noch laufen werden mittels ohasd Skript gestoppt. Der Start der Prozesse ist nach dem Rebooten eines Servers nicht nötig beim Neustart das ohasd Skript ausgeführt wird.

```bash
/etc/ohasd stop

/etc/ohasd start
```

Tags:
[[AIX]] - [[HAS]] - [[StartStop]] - [[HowTo]]
