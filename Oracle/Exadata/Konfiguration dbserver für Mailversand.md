
# Abstract

Diese Seite beschreibt die Konfiguration für eMail-Benachrichtigung auf einem Exadata compute node.

# Konfiguration

```bash
dbmcli
ALTER DBSERVER smtpServer="mailserver.$(dnsdomainname)", smtpFromAddr="$(hostname -s)@zkb.ch", smtpFrom="dbmcli@$(hostname -s)", smtpToAddr=[dbma_mail@zkb.ch](mailto:dbma_mail@zkb.ch), notificationPolicy="critical,warning,clear", notificationMethod="mail,snmp"
```

## Konfiguration mittels dcli auf allen compute nodes

```bash
vi /tmp/dbmcli.conf

#!/bin/bash

dbmcli <<EOT

ALTER DBSERVER smtpServer="mailserver.$(dnsdomainname)", smtpFromAddr="$(hostname -s)@zkb.ch", smtpFrom="dbmcli@$(hostname -s)", smtpToAddr=[dbma_mail@zkb.ch](mailto:dbma_mail@zkb.ch), notificationPolicy="critical,warning,clear", notificationMethod="mail,snmp"

EOT

dcli -g /u01/patches/dbs_group -l root -f /tmp/dbmcli.conf -d /tmp/
dcli -g /u01/patches/dbs_group -l root "chmod +x /tmp/dbmcli.conf"
dcli -g /u01/patches/dbs_group -l root "ls -la /tmp/dbmcli.conf"
dcli -g /u01/patches/dbs_group -l root "cat /tmp/dbmcli.conf"
dcli -g /u01/patches/dbs_group -l root "/tmp/dbmcli.conf"
dcli -g /u01/patches/dbs_group -l root "dbmcli -e alter dbserver validate mail"
```

Tags:
[[HowTo]] - [[Exadata]] - [[dbmcli]]

