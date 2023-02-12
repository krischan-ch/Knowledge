# Abstract

Dieses Dokument zeigt anhand der KRG02 Datenbank auf, wie ein Umbau einer RAC DB zu einer Single DB erfolgt.

# Standby DB

```bash
export DB_UNIQUE_NAME=KRG02H
export ZKB_DB_NAME=KRG02 falls nicht durch sid.conf gesetzt
export DB_NODE_SINGLE=exa101s02
export ORACLE_HOME=/app/oracle/product/19.6.0.0.200114
```

**Dataguard Status prüfen, Cluster Database disablen**

```bash
dgstatus.sh

echo "alter system set cluster_database=false sid='' scope=spfile;" | sqlplus / as sysdba
exit;
```

Aktuelle Konfiguration auslesen

```bash
srvctl config database -d ${DB_UNIQUE_NAME}
```

DB stoppen, aus Cluster entfernen

```bash
srvctl stop database -d ${DB_UNIQUE_NAME}
srvctl remove database -d ${DB_UNIQUE_NAME}
```

DB neu im Cluster eintragen, Services hinzufügen (RW/RO)

```bash
srvctl add database -db ${DB_UNIQUE_NAME} -instance ${DB_UNIQUE_NAME}2 -oraclehome ${ORACLE_HOME} -domain [prod.zkb.ch](http://prod.zkb.ch/) -spfile +DATA01/${DB_UNIQUE_NAME}/spfile${ZKB_DB_NAME}.ora -pwfile /app/oracle/admin/${ZKB_DB_NAME}/pfile/orapw${ZKB_DB_NAME} -dbname ${ZKB_DB_NAME} -c SINGLE -node ${DB_NODE_SINGLE} -startoption mount -role PHYSICAL_STANDBY

srvctl setenv database -db ${DB_UNIQUE_NAME} -envs TNS_ADMIN=/app/oracle/admin/${ZKB_DB_NAME}/pfile

srvctl add service -db ${DB_UNIQUE_NAME} -service ${ZKB_DB_NAME}_[RW.prod.zkb.ch](http://rw.prod.zkb.ch/) -preferred ${DB_UNIQUE_NAME}2 -tafpolicy BASIC -role PRIMARY -failovertype SESSION -failovermethod BASIC -failoverretry 10 -failoverdelay 10 -notification TRUE

srvctl add service -db ${DB_UNIQUE_NAME} -service ${ZKB_DB_NAME}_[RO.prod.zkb.ch](http://ro.prod.zkb.ch/) -preferred ${DB_UNIQUE_NAME}2 -tafpolicy BASIC -role PHYSICAL_STANDBY -failovertype SESSION -failovermethod BASIC -failoverretry 10 -failoverdelay 10 -notification TRUE
```

DB Konfiguration auslesen, DB Start

```bash
srvctl config database -d ${DB_UNIQUE_NAME}
srvctl getenv database -db ${DB_UNIQUE_NAME}
srvctl start database -d ${DB_UNIQUE_NAME}
```

Listener Services überprüfen

```bash
lsnrctl status | grep ${ZKB_DB_NAME}
```

# Primary DB

```bash
export DB_UNIQUE_NAME=KRG02G
export ZKB_DB_NAME=KRG02 #falls nicht durch sid.conf gesetzt
export DB_NODE_SINGLE=exa102s02
export ORACLE_HOME=/app/oracle/product/19.6.0.0.200114
```

Cluster Database disablen

```bash
echo "alter system set cluster_database=false sid='*' scope=spfile;" | sqlplus / as sysdba
exit;
```

Aktuelle Konfiguration auslesen

```bash
srvctl config database -d ${DB_UNIQUE_NAME}
```

DB stoppen, aus Cluster entfernen

```bash
srvctl stop database -d ${DB_UNIQUE_NAME}
srvctl remove database -d ${DB_UNIQUE_NAME}
```

DB neu im Cluster eintragen, Services hinzufügen (RW/RO)

```bash
srvctl add database -db ${DB_UNIQUE_NAME} -instance ${DB_UNIQUE_NAME}2 -oraclehome ${ORACLE_HOME} -domain [prod.zkb.ch](http://prod.zkb.ch/)-spfile +DATA01/${DB_UNIQUE_NAME}/spfile${ZKB_DB_NAME}.ora -pwfile /app/oracle/admin/${ZKB_DB_NAME}/pfile/orapw${ZKB_DB_NAME} -dbname ${ZKB_DB_NAME} -c SINGLE -node ${DB_NODE_SINGLE}

srvctl getenv database -db ${DB_UNIQUE_NAME} -envs TNS_ADMIN=/app/oracle/admin/${ZKB_DB_NAME}/pfile

srvctl add service -db ${DB_UNIQUE_NAME} -service ${ZKB_DB_NAME}_[RW.prod.zkb.ch](http://rw.prod.zkb.ch/) -preferred ${DB_UNIQUE_NAME}2 -tafpolicy BASIC -role PRIMARY -failovertype SESSION -failovermethod BASIC -failoverretry 10 -failoverdelay 10 -notification TRUE

srvctl add service -db ${DB_UNIQUE_NAME} -service ${ZKB_DB_NAME}_[RO.prod.zkb.ch](http://ro.prod.zkb.ch/) -preferred ${DB_UNIQUE_NAME}2 -tafpolicy BASIC -role PHYSICAL_STANDBY -failovertype SESSION -failovermethod BASIC -failoverretry 10 -failoverdelay 10 -notification TRUE
```

DB Konfiguration auslesen, DB Start

```bash
srvctl config database -d ${DB_UNIQUE_NAME}
srvctl getenv database -db ${DB_UNIQUE_NAME}
srvctl start database -d ${DB_UNIQUE_NAME}
```

Listener Services überprüfen

```bash
lsnrctl status | grep ${ZKB_DB_NAME}
```

Dataguard prüfen

```bash
dgstatus.sh
```

### Tags:

[[HowTo]] - [[Exadata]] - [[RAC]]