
# Einschränkungen

Ein ORACLE_HOME kann nur direkt nach der Software-Only-Installation als read only home konfiguriert werden.

Laut Aussage vom Oracle Support ist es nicht möglich dies zu ändern sobald eine DB im entsprechenden Home erstellt wurde.

# Home Installation

## Filesystem erstellen

```bash
mkdir /app/oracle/product/18.3.0
lvcreate -L 30G -n ora1830_lv /dev/VGExaDb
mkfs -t ext4 -c /dev/VGExaDb/ora1830_lv
echo "/dev/VGExaDb/ora1830_lv /app/oracle/product/18.3.0 ext4 defaults 0 0" >>/etc/fstab
chown --reference /app/oracle/product/18.0.0 /app/oracle/product/18.3.0
chmod --reference /app/oracle/product/18.0.0 /app/oracle/product/18.3.0
mount /app/oracle/product/18.3.0
chown --reference /app/oracle/product/18.0.0 /app/oracle/product/18.3.0
chmod --reference /app/oracle/product/18.0.0 /app/oracle/product/18.3.0
```

## Base Release installieren

siehe Dokument 18c - Exadata rdbms home Installation

## 18.3 Patchset installieren

```bash
#root
oramnt anhängen
/usr/local/db/bin/oramnt -m

#oracle

#Opatch Prüfen
opatch version
OPatch Version: 12.2.0.1.14
OPatch succeeded.

#Sonst
cdh
rm -rf Opatch
cp -rp /app/oracle/product/18.3.0/OPatch /app/oracle/product/18.3.0.0.180717/OPatch
PATCH_LOC=/oramnt/ORACLE_LINUX/orapatch/exadata/12c/18.3.0.0.180717/28096386; export PATCH_LOC
$ORACLE_HOME/OPatch/opatch prereq CheckConflictAgainstOHWithDetail -phBaseDir $PATCH_LOC/28090523
$ORACLE_HOME/OPatch/opatch prereq CheckConflictAgainstOHWithDetail -phBaseDir $PATCH_LOC/28090553
$ORACLE_HOME/OPatch/opatch prereq CheckSystemSpace -phBaseFile $PATCH_LOC/patch_list_dbhome.txt
#opatchauto apply ${PATCH_LOC}/28096386 -oh $ORACLE_HOME -inplace
${PATCH_LOC}/28090553/custom/scripts/prepatch.sh -dbhome ${ORACLE_HOME}
${ORACLE_HOME}/OPatch/opatch apply -oh ${ORACLE_HOME} -local ${PATCH_LOC}/28090523
${ORACLE_HOME}/OPatch/opatch apply -oh ${ORACLE_HOME} -local ${PATCH_LOC}/28090553
${PATCH_LOC}/28090553/custom/scripts/postpatch.sh -dbhome ${ORACLE_HOME} 

#root
/usr/local/db/bin/oramnt -u
```

# Links

| Website |
| --- |
| [dbi Services - 18c Read Only Oracle Home]([https://blog.dbi-services.com/18c-read-only-oracle-home](https://blog.dbi-services.com/18c-read-only-oracle-home)) |
| [Oracle Doku - Understanding Read-Only Oracle Homes]([https://docs.oracle.com/en/database/oracle/oracle-database/18/ladbi/configuring-read-only-oracle-homes.html#GUID-906DA159-AC83-4ACC-A8A6-5B4A39EB72E1](https://docs.oracle.com/en/database/oracle/oracle-database/18/ladbi/configuring-read-only-oracle-homes.html#GUID-906DA159-AC83-4ACC-A8A6-5B4A39EB72E1)) |

# Tags:

[[HowTo]] - [[ReadOnlyHome]]
