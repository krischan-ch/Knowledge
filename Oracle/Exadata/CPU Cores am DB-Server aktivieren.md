
# Abstract

Diese Seite beschreibt wie man auf einem Exadata DB-Server zusätzliche CPU-Cores freischaltet. Dazu wird das dbmcli Utility verwendet.

Alle Informationen in dieser Seite stammen aus dem Exadata Database Machine Maintenance Guide Version 12.1.

# Cores aktivieren
> (als root)

```bash
dbmcli -e LIST DBSERVER attributes coreCount
dbmcli -e ALTER DBSERVER pendingCoreCount = 12
dbmcli -e LIST DBSERVER attributes pendingCoreCount
reboot
dbmcli -e LIST DBSERVER attributes coreCount
```


Tags:
[[Exadata]] - [[HowTo]] - [[dbmcli]]
