
# Abstract

Datenbank Migration mittels "Transportable Tablespace" Methode (TTS / xTTS).

# Features

- Cross Plattform Endian-Konvertierung: geeignet für Migrationen AIX -> Exadata
- RDBMS Upgrade: Oracle Version X -> Y (z.B. 12c -> 19c)
- DB Storage Konvertierung: Filesystem/ASM -> ASM
- sehr kleine Downtime mit der inkrementellen Backup-Methode (Grossteil der Migration erfolgt online)
- Automatisierung: einfache Handhabung durch xTTS Scripts von Oracle

# xTTS - grober Ablauf

- Erstellen einer leeren DB auf dem Zielsystem
- INC0 File-Backup aller zu transportierender User-Tablespaces aufs NFS-Share
- Konvertierung und Restore der User-Tablespace-Backupfiles auf der Ziel-DB
- Setzen aller zu transportierenden User-Tablespaces auf READ-ONLY
- INC1 File-Backup aller User-Tablespaces und Restore auf der Ziel-DB
- Export der Tablespace- und User-Objekt-Metadaten und Import auf der Ziel-DB
- Setzen aller transportierten User-Tablespaces auf der Ziel-DB auf READ-WRITE

# Einschränkungen

- Characterset und Blocksize von Source- und Ziel-DB müssen gleich sein
- compatible Parameter von Source- und Ziel-DB müssen gleich sein
- User Metadaten Export und Import erfolgt separat (mit Datapump)
- Anzahl der zu transportierenden User Tablespaces ist limitiert (maximale Zeilenlänge im Konfig-File beschränkt)
- RMAN Backup Compression muss auf der Souce- und Ziel-DB temporär deaktiviert werden
- Source/Ziel DB im Archivelog-Modus

# xTTS Dokumentation und Scripts

## Oracle MoS Note
V4 Reduce Transportable Tablespace Downtime using Cross Platform Incremental Backup ([MoS Note #2471245.1](https://support.oracle.com/epmos/faces/DocContentDisplay?_afrLoop=18678438612348&id=2471245.1)).

## xTTS Skripts
xtt.properties
xttdriver.pl
...

## xTTS Script Handling

Konfigurationsfile:
  xtt.properties
```bash
xTTS Backup starten:
  $ORACLE_HOME/perl/bin/perl xttdriver.pl --backup

xTTS Restore starten:
  $ORACLE_HOME/perl/bin/perl xttdriver.pl --restore
```
Das Perl-Script generiert verschiedene txt-Files und orientiert sich anhand den schon vorhandenen txt-Files als Referenz zum vorherigen Lauf, was zu tun ist, v.a.:
  res.txt
  incrbackups.txt
  xttnewdatafiles.txt
  xttplan.txt
  ...

xTTS Logfiles werden unter dem aktuellen (oder konfigurierten) Pfad in einem Subdirectory angelegt.

# xTTS Vorgehen

## Vorbereitung und Setup
### Leere Ziel-DB auf Ziel-Server einrichten

- Characterset und Blocksize müssen gleich sein!
- Source/Ziel DB im Archivelog-Modus
- Data Guard und Flashback werden für die Migration deaktiviert (wegen Overhead)

referenzierte Pfade:

**script-dir**: Pfad für SQL-Scripts, am einfachsten auf NFS-Share (z.B. /zfssa/transfer1/${ORACLE_SID}/scripts)
**xtts-work-dir**: Pfad für xtts-Scripts, am einfachsten auf NFS-Share (z.B. /zfssa/transfer1/${ORACLE_SID}/scripts)
**backup-dir**: Pfad für Backup Files, auf NFS-Share (z.B. /zfssa/transfer1/${ORACLE_SID}/backup)
**import-dir**: Pfad für Data Pump Files, auf NFS-Share (z.B. /zfssa/transfer1/${ORACLE_SID}/dmp)

# Appl-DB-User auf Ziel-DB einrichten
Default-TBS der appl. User muss temporär aufs TBS USERS konfiguriert werden.

## Source-DB:

```bash
cd <script-dir>
sq
@spool_asm_tablespaces.sql
@spool_roles.sql
@spool_system_ZKB_tables.sql
@spool_users.sql
@spool_user_sys_grants.sql
@spool_privileges.sql
show parameter compatible
exit
```

## Ziel-DB:

```bash
cd <script-dir>

# auf Source-DB generierte Scripts aufs Ziel-System kopieren (nicht nötig wenn <script-dir> auf NFS)

# Default-User-TBS muss aufs Tablespace USERS geändert werden und das Script zum Zurücksetzen vorbereiten:

awk '{if($1 ~ /DEFAULT/ && $2 ~ /TABLESPACE/ && $3 !~ /USERS/ && $3 !~ /SYSAUX/ && $3 !~ /TEMP/) {print "    DEFAULT TABLESPACE USERS"} else print $0}' create_user.sql > create_user_DEFTBS_USERS.sql
awk '{if($1 ~ /CREATE/) {u=$3; getline; print "alter user " u " " $0 ";"}}' create_user.sql > alter_user_DEFTBS_back.sql 
 
sq
@create_role.sql
@create_user_DEFTBS_USERS.sql
@seluser                                   -- User vorhanden mit Default TBS = USERS
@seltbs                                     -- noch keine User TBS vorhanden
@selobj                                     -- noch keine User Objekte vorhanden
@seldb                                      -- Archivelog
@selflash                                  -- Flashback OFF
show parameter compatible  -- muss gleich der Source-DB Einstellung sein​
exit

# DG: Standby DB mit 'dgcmd.sh -d' deaktivieren
```

# NAS Share am Source- und Ziel-System über NFS mounten
Die verfügbare Grösse des NAS-Shares muss dem unkomprimierten INC0 Backup-Volumen + INC1 + Reserve entsprechen!

```bash
mount | grep oramnt
sudo oramnt -m
mount | grep oramnt
ll /zfssa/transfer
df -gP /zfssa/transfer?/P?
```

# TTS User Tablespaces identifizieren
Auflistung aller zu transportierenden User Tablespaces nach aufsteigender Grösse (wichtig wegen der Parallelitätsverarbeitung) und als Liste für xtt.properties:

```bash
sq
set lines 300 pages 100 serverout on
def TBS_LIST=""
col TBS_LIST new_v TBS_LIST for a200
select distinct TABLESPACE_NAME tbs, round(sum(bytes)/1024/1024/1024,1) GB from DBA_SEGMENTS where OWNER not in ('AUDSYS','DBSNMP','GSMADMIN_INTERNAL','OUTLN','SYSTEM','SYS','WMSYS','XDB','APPQOSSYS','DBSFWUSER') group by TABLESPACE_NAME order by 1;
select listagg(tbs, ',') within group (order by GB) TBS_LIST from (select distinct TABLESPACE_NAME tbs, round(sum(bytes)/1024/1024/1024,1) GB from DBA_SEGMENTS where OWNER not in ('AUDSYS','DBSNMP','GSMADMIN_INTERNAL','OUTLN','SYSTEM','SYS','WMSYS','XDB','APPQOSSYS','DBSFWUSER') group by TABLESPACE_NAME);

--Ermitteln TBS für Kontrolle ob alle TBS gemäss dem obigen sql ausgewiesen wurde
select name tbs from v$tablespace where name not in ('SYSTEM','SYSAUX','UNDOTBS1','TEMP','USERS','SYSAUD') order by 1;

-- die zu transportierenden Tablespaces müssen self-contained sein, Check:

execute DBMS_TTS.TRANSPORT_SET_CHECK('&TBS_LIST', TRUE);
select * from TRANSPORT_SET_VIOLATIONS;

-- wenn hier ein zusätzliches Tablespace aufgelistet wird, muss dieses auch noch mittransportiert werden, d.h. ins xtt.properties eingetragen werden (nächstes Kapitel)
exit
```

# xTTS Konfiguration

**ACHTUNG**: diret ins ZFS kann nicht geschrieben werden, ins TEMP zuerst und dann moven
```bash
mkdir /tmp/XTTS
cd /tmp/XTTS 
sudo zsa -n -d Oracle -g rman_xttconvert_VER4.3.zip -o oracle
mv * <xtts-work-dir>/

# Holen der xTTS-Scripts: 
cd <xtts-work-dir>
sudo zsa -n -d Oracle -g rman_xttconvert_VER4.3.zip -o oracle
```

Konfig-File xtt.properties konfigurieren:   
**ACHTUNG**: keine Spaces am Ende der locations hinzufügen (copy-paste!), mit VI prüfen dass keine Blanks am Ende der Zeile nach dem Einfügen vorhanden ist!

```
tablespaces=   # Liste der xTTS User Tablespaces, kommasepariert, am effektivsten nach aufsteigender Grösse geordnet (copy-paste vom obigen Select)  
platformid=  # 6 für AIX (select PLATFORM_ID from V$DATABASE;)  
src_scratch_location=   # Pfad der Backup-Files vom Source-System aus   = <backup-dir>  
dest_datafile_location=   # +DATA01 für ASM als Destination  
dest_scratch_location=    # Pfad der Backup-Files vom Ziel-System aus  
parallel=   # Parallelitätsgrad für RMAN, z.B = 4 (hier irrelevant)  
usermantransport=1   # MUSS bei Source-DB >= 12c  (nicht default!)  
```

# Prepare Phase (Applikation bleibt online)

## Source-DB: RMAN Default-Settings für xTTS anpassen

RMAN Compression deaktivieren + effektiven Parallelitätsgrad einstellen (hier: 12):

```bash
rmanc
show all;
CONFIGURE DEFAULT DEVICE TYPE TO DISK;
CONFIGURE DEVICE TYPE DISK PARALLELISM 12 BACKUP TYPE TO BACKUPSET;
show all;
exit
```

# Source-DB: INC0 TTS Backup
```bash
sudo zsicron -s $ORACLE_SID stop

ll /tmp/rman_$ORACLE_SID      # File NICHT vorhanden - kein Backup am laufen!
touch /tmp/rman_$ORACLE_SID

cd <xtts-work-dir>
export NLS_LANG=AMERICAN_AMERICA.WE8ISO8859P1    # für Perl-Script notwendig
export TMPDIR=$PWD              # wird benötigt für den Pfad vom Outputfile res.txt
ll res.txt                                      # File ist NICHT vorhanden - darf für INC0 Backup nicht vorhanden sein, sonst umbenennen!
nohup $ORACLE_HOME/perl/bin/perl xttdriver.pl --backup  &
tail -f nohup.out

# Status-Check zwischendurch:
selsess -S selrman -i

# wenn Job fertig ist:
ls -ltr *.txt                                  # neue txt-Files wurden generiert, u.a. res.txt newfile.txt, xttplan.txt
ll <backup-dir>                          # aktuelle INC0 Backupfiles vorhanden, Namensformat *.tf​

# xtts Logfile unter backup_<Datum>/<Datum>.log
```

# Source-DB: RMAN Default-Settings zurücksetzen
```bash
rmanc
CONFIGURE DEFAULT DEVICE TYPE TO 'SBT_TAPE';
CONFIGURE DEVICE TYPE DISK PARALLELISM 1 BACKUP TYPE TO COMPRESSED BACKUPSET;
exit

rm /tmp/rman_$ORACLE_SID

sudo zsicron -s $ORACLE_SID start

sudo oramnt -u
```

# Ziel-DB: INC0 TTS Restore / Konvertierung

```bash
  cd <xtts-work-dir>                  # wenn nicht auf NFS Share: xtts Scripts und txt-Files vom Source-System hierher kopieren

# RMAN Compression auf der Ziel-DB deaktivieren (sonst Fehler RMAN-03009: failure of conversion at target command ... ORA-19699: cannot make copies with compression enabled) + effektiven Parallelitätsgrad einstellen (hier: 12):
rmanc
show DEVICE TYPE;
CONFIGURE DEVICE TYPE DISK PARALLELISM 12 BACKUP TYPE TO BACKUPSET;
exit



ll res.txt                                    # aktuelles File vom INC0 Backup vorhanden
ll <backup-dir>                        # aktuelle INC0 Backupfiles vorhanden

export NLS_LANG=AMERICAN_AMERICA.WE8ISO8859P1    # für Perl-Script notwendig
export TMPDIR=$PWD             # wird benötigt für den Pfad vom Inputfile res.txt
nohup $ORACLE_HOME/perl/bin/perl xttdriver.pl --restore  &
tail -f nohup.out


# Status-Check zwischendurch:
selsess -S selrman -i

# falls ein Fehler beim Restore passiert und von Neuem angefangen werden soll, muss das erstellte File "FAILED" vorher gelöscht werden

 
# wenn Job fertig ist, RMAN Compression auf der Ziel-DB wieder aktivieren:
# Parallelitätsgrad mit Ursprungswert ersetzen (hier: 6):
rmanc
show DEVICE TYPE;
CONFIGURE DEVICE TYPE DISK PARALLELISM 6 BACKUP TYPE TO COMPRESSED BACKUPSET;
exit

# xtts Logfile unter restore_<Datum>/<Datum>.log
```

# Final Incremental Backup / Transport Phase (Applikation offline)
## Source-DB: Vorbereitungsarbeiten

```bash
  cd <script-dir>

sq

 
select count(*) from dba_recyclebin;
-- falls > 0:
  purge dba_recyclebin;

-- cyclic/dangling synonyms check
-- Anzeige der Objekte:
 
col OWNER for a14
col SYNONYM_NAME for a30
col TABLE_OWNER for a10
col TABLE_NAME for a30
col DB_LINK for a10
set lines 200
with base as
 (select t1.owner, 
         t1.synonym_name,
         t1.table_owner,
         t1.table_name,
         t2.object_type
    from dba_synonyms t1, dba_objects t2
   where t1.table_owner = t2.OWNER
     and t1.table_name = t2.OBJECT_NAME),
conn as
 (select t1.owner,  
         t1.synonym_name,
         t1.table_owner,
         t1.table_name,
         prior t1.owner prior_owner,
         prior t1.synonym_name prior_synonym,
         prior t1.table_owner prior_tab_owner,
         prior t1.table_name prior_tab_name,
         level lvl, 
         connect_by_root object_type base,
         sys_connect_by_path(owner || '.' || synonym_name, '\') paths
    from base t1
  connect by nocycle prior owner = table_owner
         and prior synonym_name = table_name
   start with object_type <> 'SYNONYM')
select *
  from dba_synonyms
 where (owner, synonym_name) not in (select owner, synonym_name from conn)
   and table_owner not like 'SYS%'
 order by table_owner, table_name;

 
-- Check der Objekte z.b. mit desc ....
...

 
-- Bereinigung der Objekte:

set serverout on
DECLARE 
   CURSOR c1 IS
 with base as
 (select t1.owner,
         t1.synonym_name,
         t1.table_owner,
         t1.table_name,
         t2.object_type
    from dba_synonyms t1, dba_objects t2
   where t1.table_owner = t2.OWNER
     and t1.table_name = t2.OBJECT_NAME),
conn as 
 (select t1.owner,
         t1.synonym_name,
         t1.table_owner,
         t1.table_name,
         prior t1.owner prior_owner,
         prior t1.synonym_name prior_synonym,
         prior t1.table_owner prior_tab_owner,
         prior t1.table_name prior_tab_name,
         level lvl,
         connect_by_root object_type base,
         sys_connect_by_path(owner || '.' || synonym_name, '\') paths
    from base t1
  connect by nocycle prior owner = table_owner
         and prior synonym_name = table_name
   start with object_type <> 'SYNONYM')
select *
  from dba_synonyms
 where (owner, synonym_name) not in (select owner, synonym_name from conn)
   and table_owner not like 'SYS%' 
 order by table_owner, table_name;
   stmt varchar2(1000);
BEGIN
   FOR r1 IN c1
   LOOP 
      dbms_output.put_line('removing cyclic synonym: '||r1.OWNER||'.'||r1.SYNONYM_NAME);
      if r1.OWNER = 'PUBLIC' 
      then 
        stmt := 'drop public synonym '||r1.SYNONYM_NAME;
      else
        stmt := 'drop synonym '||r1.OWNER||'.'||r1.SYNONYM_NAME;
      end if;
      EXECUTE IMMEDIATE stmt;
   END LOOP;
END;
/

@selinvobj   -- keine invaliden Objekte!, sonst @utlrp

@spool_public_synonyms.sql

exit
```

# Source-DB: RMAN Default-Settings für xTTS anpassen

```sql
rmanc
CONFIGURE DEFAULT DEVICE TYPE TO DISK;
CONFIGURE DEVICE TYPE DISK PARALLELISM 8 BACKUP TYPE TO BACKUPSET;
exit

sudo zsicron -s $ORACLE_SID stop
```

# Source-DB: TTS Tablespaces READ-ONLY setzen

```sql
sq

show parameter job_queue_processes
alter system set job_queue_processes=0 scope=memory;
 
@seltbs
@selsess       -- keine Appl-Sessions vorhanden (oder nur irrelevante)
@seltbs         -- TTS User Tablespaces sind ONLINE
 
-- TBS read-only setzen:

set lines 300 pages 100 serverout on

DECLARE
   CURSOR c1 IS        
     select name tbs from v$tablespace where name not in ('SYSTEM','SYSAUX','UNDOTBS1','TEMP','USERS','SYSAUD') order by 1;    
   stmt varchar2(1000);
BEGIN
   FOR r1 IN c1
   LOOP
      dbms_output.put_line('setting TBS read-only: '||r1.tbs);
      stmt := 'alter tablespace '||r1.tbs||' read only';
      EXECUTE IMMEDIATE stmt;
   END LOOP;
END;
/

@seltbs         -- alle TTS User Tablespaces sind READ ONLY
exit
```

# Source-DB: INC1 TTS Backup

```bash
mount | grep oramnt
sudo oramnt -m
mount | grep oramnt
ll /zfssa/transfer
df -gP /zfssa/transfer?/P?

ll /tmp/rman_$ORACLE_SID       # kein File vorhanden - kein Backup am laufen
touch /tmp/rman_$ORACLE_SID

cd <xtts-work-dir>

export NLS_LANG=AMERICAN_AMERICA.WE8ISO8859P1    # für Perl-Script notwendig
export TMPDIR=$PWD                   # wird benötigt für den Pfad vom Outputfile res.txt

ll <backup-dir>                               # aktuelle INC0 Backupfiles vorhanden​

​ll newfile.txt res.txt xttnewdatafiles.txt xttplan.txt    # Files vom vorherigen INC0 Backup müssen vorhanden sein
 
$ORACLE_HOME/perl/bin/perl xttdriver.pl --backup
 
# diese Warnings am Ende sind zu ignorieren, da TBS read-only sind:
  ...
  Warnings found in executing /usr/local/amds01/AMDS01/patch/2020_ExaMigration/backup_Apr1_Wed_10_50_53_840//xttpreparenextiter.sql
  ####################################################################
  Prepare newscn for Tablespaces: 'SPEZM_I03','SPEZM_LOAD'
  DECLARE*
  ERROR at line 1:
  ORA-20001: TABLESPACE(S) IS READONLY OR,
  OFFLINE JUST CONVERT, COPY
  ORA-06512: at line 285
  ...

# xtts Logfile unter backup_<Datum>/<Datum>.log
 
ls -ltr *.txt                             # Files upgedatet: newfile.txt, tsbkupmap.txt, incrbackups.txt, res.txt, xttplan.txt

ll <backup-dir>                     # neu generierte INC1 Backupfiles vorhanden, Namensformat xxxxxxxx_1_1​​
```
# Source-DB: Tablespace Metadata + User Metadata Export

```bash
cd <import-dir>

# Transportable Tablespace Export:

echo "EXCLUDE=STATISTICS"                > exp_tts_meta.par
echo "TRANSPORT_FULL_CHECK=Y"  >> exp_tts_meta.par​
awk -F= '$1~/^tablespaces/ {print "TRANSPORT_TABLESPACES=" $2}' ../scripts/xtt.properties >> exp_tts_meta.par​

cat exp_tts_meta.par                # Check (TRANSPORT_TABLESPACES Definition sollte auf einer Zeile sein)

expdp_full_db.sh -e $PWD -l $PWD -f TTS_META.dmp -P exp_tts_meta.par -F -Z

# User Metadata Objects Export:
echo "INCLUDE=FUNCTION"                                   > exp_user_meta_obj.par
echo "INCLUDE=JOB"                                             >> exp_user_meta_obj.par
echo "INCLUDE=MATERIALIZED_VIEW"                >> exp_user_meta_obj.par        # not all Mat-Views are included in exp_tts_meta 
echo "INCLUDE=PACKAGE"                                   >> exp_user_meta_obj.par
echo "INCLUDE=PROCEDURE"                                 >> exp_user_meta_obj.par
echo "INCLUDE=SEQUENCE"                                  >> exp_user_meta_obj.par
echo "INCLUDE=SYNONYM"                                   >> exp_user_meta_obj.par
echo "INCLUDE=TABLE:\"IN (select TABLE_NAME from DBA_TABLES where TEMPORARY = 'Y')\""  >> exp_user_meta_obj.par      # global temporary tables
echo "INCLUDE=TYPE"                                            >> exp_user_meta_obj.par        # not all Types are included in exp_tts_meta​
echo "INCLUDE=VIEW"                                           >> exp_user_meta_obj.par

expdp_user_db.sh -e $PWD -l $PWD -f USER_META.dmp -R -o <USERS> -P exp_user_meta_obj.par -p 2          # alle zu transportierenden User-Schemas mit Objekten mit -o angeben

# special SYS Objects Export (nur falls vorhanden):
echo "INCLUDE=CONTEXT"   > exp_sys_obj.par
echo "INCLUDE=DB_LINK"  >> exp_sys_obj.par

expdp_full_db.sh -e $PWD -l $PWD -f SYS_OBJ.dmp -R -P exp_sys_obj.par

# Kontrolle:

ll *_expdp.log                            # export Logfiles vorhanden
ll *.dmp                                      # export Dumpfiles vorhanen

grep ORA- *_expdp.log             # NO errors!

# Abschluss:

rm /tmp/rman_$ORACLE_SID

rmanc
CONFIGURE DEFAULT DEVICE TYPE TO 'SBT_TAPE';
CONFIGURE DEVICE TYPE DISK PARALLELISM 1 BACKUP TYPE TO COMPRESSED BACKUPSET;

sudo zsicron -s ${ORACLE_SID} start                     # Crontab wieder aktivieren

sudo oramnt -u
```

# Ziel-DB: INC1 TTS Restore
  
```bash
## oradeploy-Server: 

# DB-Backup deaktivieren:
<sid>    # DB Env setzen
sudo dbcron -l
sudo dbcron -s ${ORACLE_SID} -d

## Exa-Server:

cd <xtts-work-dir>                  # wenn nicht auf NFS Share: neue txt-Files vom Source-System hierher kopieren

# RMAN Compression auf der Ziel-DB deaktivieren (sonst Fehler RMAN-03009: failure of conversion at target command ... ORA-19699: cannot make copies with compression enabled):
# Parallelitätsgrad wählen (hier: 8):
rmanc
show DEVICE TYPE;
CONFIGURE DEVICE TYPE DISK PARALLELISM 8 BACKUP TYPE TO BACKUPSET;
exit

ll <backup-dir>                         # INC1 (+INC0) Backupfiles vorhanden, Namensformat xxxxxxxx_1_1​​

ll newfile.txt res.txt xttnewdatafiles.txt xttplan.txt    # Files vom vorherigen INC1 Backup müssen vorhanden sein

export NLS_LANG=AMERICAN_AMERICA.WE8ISO8859P1    # für Perl-Script notwendig
export TMPDIR=$PWD                     # wird benötigt für den Pfad vom Inputfile res.txt

$ORACLE_HOME/perl/bin/perl xttdriver.pl --restore

# xtts Logfile unter restore_<Datum>/<Datum>.log
 
# wenn OK, RMAN Compression auf der Destination DB wieder aktivieren, Parallelität zurückstellen (hier: 6):
rmanc
show DEVICE TYPE;
CONFIGURE DEVICE TYPE DISK PARALLELISM 6 BACKUP TYPE TO COMPRESSED BACKUPSET;
exit

# Ziel-DB: TBS Metadata + User Metadata Import

cd <import-dir>

ll *.dmp                              # Dump-Files vom Export vorhanden​

# User Metadata Objects Import:
impdp_full_db.sh -i $PWD -l $PWD -f USER_META_%U.dmp -p 2

# special SYS Objects Import (nur falls vorhanden):
impdp_full_db.sh -i $PWD -l $PWD -f SYS_OBJ_01.dmp

# Transportable Tablespace Import:
cat ../scripts/xttnewdatafiles.txt                        # File und Inhalt vorhanden

awk -F, '{if ($2!~/^$/) files=files "\047" $2 "\047,"} END {print "TRANSPORT_DATAFILES=" substr(files,1,length(files)-1)}' ../scripts/xttnewdatafiles.txt > imp_tts_meta.par
cat imp_tts_meta.par                           # Check (TRANSPORT_DATAFILES komplett und auf einer Zeile)

impdp_full_db.sh -i $PWD -l $PWD -f TTS_META_01.dmp -P imp_tts_meta.par -F

ll *_impdp.log               # import Logfiles vorhanden

grep -c ^ORA- *_impdp.log    # Anzahl Fehler pro Logfile
grep ^ORA- *_impdp.log | awk '{print $1 " " $2 " " $3 " ..."}' | sort | uniq -c    # keine relevanten Fehler!   

# zu erwartende datapump Warnings:
  ORA-31684: Object type XXXXX:ZZZ already exists
  ORA-31685: Object type MATERIALIZED_VIEW: YYYY.ZZZ failed due to insufficient privileges
  ORA-39082: Object type XXXXX:YYYY.ZZZ created with compilation warnings
```

# Ziel-DB: Postprocessing

```bash
cd <script-dir>                        # wenn nicht auf NFS: neue *.sql Files vom Source-System hierher kopieren

ll alter_user_DEFTBS_back.sql create_privileges.sql create_public_synonym.sql         # alle Files vorhanden

sq
-- User und Grants richtigstellen:
@crspool alter_users.log
@alter_user_DEFTBS_back.sql
@create_privileges.sql
@cre_usr_sys_grants.sql
@create_public_synonym.sql
@utlrp  
@selinvobj         -- idealerweise keine invaliden Objekte vorhanden!
exit
```

# Ziel-DB: RMAN Validierung der TTS Tablespaces

```sql
# RMAN Parallelität temporär höher setzen (je nach Anzahl Datafiles)
rmanc
show DEVICE TYPE;
CONFIGURE DEVICE TYPE DISK PARALLELISM 16 BACKUP TYPE TO COMPRESSED BACKUPSET;
exit

awk -F= '$1~/^tablespaces/ {print "validate tablespace " $2 " check logical;"}' xtt.properties | rman target / | tee ${ORACLE_SID}_rman_validate_tbs.log

egrep -e '^[0-9][0-9]* ' ${ORACLE_SID}_rman_validate_tbs.log

# Output: TBS-File Status muss bei allen OK sein, keine korrupten Blöcke!

awk '$1~/(Data|Index|Other)/ {if($2!~/0/) print $0}' ${ORACLE_SID}_rman_validate_tbs.log
# kein Output!


# RMAN Parallelität wieder zürücksetzen
rmanc
show DEVICE TYPE;
CONFIGURE DEVICE TYPE DISK PARALLELISM 8 BACKUP TYPE TO COMPRESSED BACKUPSET;
exit
```
  
# Ziel-DB: TTS Tablespaces READ-WRITE setzen

```sql
@seltbs     -- TTS Tablespaces sind vorhanden aber noch READ ONLY

set lines 300 pages 100 serverout on
DECLARE
   CURSOR c1 IS       
     select tablespace_name tbs from DBA_TABLESPACES where tablespace_name not in ('SYSTEM','SYSAUX','UNDOTBS1','TEMP','USERS','SYSAUD') and status='READ ONLY' order by 1;
   stmt varchar2(1000);
BEGIN
   FOR r1 IN c1
   LOOP
      dbms_output.put_line('setting TBS read-write: '||r1.tbs);
      stmt := 'alter tablespace '||r1.tbs||' read write';
       EXECUTE IMMEDIATE stmt;
   END LOOP;
END;
/
 
@seltbs     -- TTS Tablespaces sind READ WRITE

-- cyclic/dangling synonyms check
 
-- Anzeige der Objekte:
 
col OWNER for a14
col SYNONYM_NAME for a30
col TABLE_OWNER for a10
col TABLE_NAME for a30
col DB_LINK for a10
set lines 200
 
with base as
 (select t1.owner,
         t1.synonym_name,
         t1.table_owner,
         t1.table_name,
         t2.object_type
    from dba_synonyms t1, dba_objects t2
   where t1.table_owner = t2.OWNER
     and t1.table_name = t2.OBJECT_NAME),
conn as
 (select t1.owner,
         t1.synonym_name,
         t1.table_owner,
         t1.table_name,
         prior t1.owner prior_owner,
         prior t1.synonym_name prior_synonym,
         prior t1.table_owner prior_tab_owner,
         prior t1.table_name prior_tab_name,
         level lvl,
         connect_by_root object_type base,
         sys_connect_by_path(owner || '.' || synonym_name, '\') paths
    from base t1
  connect by nocycle prior owner = table_owner
         and prior synonym_name = table_name
   start with object_type <> 'SYNONYM')
select *
  from dba_synonyms
 where (owner, synonym_name) not in (select owner, synonym_name from conn)
   and table_owner not like 'SYS%'
 order by table_owner, table_name;

-- Check der Objekte z.b. mit desc ....
...

-- Bereinigung der Objekte:

set serverout on
DECLARE
   CURSOR c1 IS         
with base as
 (select t1.owner,
         t1.synonym_name,
         t1.table_owner,
         t1.table_name,
         t2.object_type
    from dba_synonyms t1, dba_objects t2
   where t1.table_owner = t2.OWNER
     and t1.table_name = t2.OBJECT_NAME),
conn as
 (select t1.owner,
         t1.synonym_name,
         t1.table_owner,
         t1.table_name,
         prior t1.owner prior_owner,
         prior t1.synonym_name prior_synonym,
         prior t1.table_owner prior_tab_owner,
         prior t1.table_name prior_tab_name,
         level lvl,
         connect_by_root object_type base,
         sys_connect_by_path(owner || '.' || synonym_name, '\') paths
    from base t1
  connect by nocycle prior owner = table_owner
         and prior synonym_name = table_name
   start with object_type <> 'SYNONYM')
select *
  from dba_synonyms
 where (owner, synonym_name) not in (select owner, synonym_name from conn)
   and table_owner not like 'SYS%'
 order by table_owner, table_name;
   stmt varchar2(1000);
BEGIN
   FOR r1 IN c1
   LOOP
      dbms_output.put_line('removing cyclic synonym: '||r1.OWNER||'.'||r1.SYNONYM_NAME);
      if r1.OWNER = 'PUBLIC'
      then
        stmt := 'drop public synonym '||r1.SYNONYM_NAME;
      else
        stmt := 'drop synonym '||r1.OWNER||'.'||r1.SYNONYM_NAME;
      end if;
      EXECUTE IMMEDIATE stmt;
   END LOOP;
END;
/

@selinvobj    -- keine (neuen) invaliden Objekte

exit
```

# Source-DB UND Ziel-DB: Check der User-Objekte

```sql
# Check und Vergleich der User-Objekte auf Source und Ziel DB:

cd <script-dir>​

sq
set lines 200 pages 200
col owner for a20
col PUBLIC_SYNONYMS for a20

break on OWNER skip 1
compute sum of COUNT on OWNER
 
@define_infra_users

def HOST=""
col HOST new_value HOST
select HOST_NAME HOST from v$instance;
set verify off trimsp on

spool object_status_&_CONNECT_IDENTIFIER._&HOST..log

select owner, object_type, count(*) COUNT 
 from DBA_OBJECTS 
 where owner not in (&INFRA_USERS1, &INFRA_USERS2, &INFRA_USERS3, &INFRA_USERS4)
 group by owner, object_type 
 order by owner, object_type; 

select owner, constraint_type, count(*) COUNT 
 from DBA_CONSTRAINTS 
 where owner not in (&INFRA_USERS1, &INFRA_USERS2, &INFRA_USERS3, &INFRA_USERS4)
 group by owner, constraint_type 
 order by owner, constraint_type;
 
select TABLE_OWNER PUBLIC_SYNONYMS, count(*) COUNT 
 from DBA_SYNONYMS 
 where TABLE_OWNER not in (&INFRA_USERS1, &INFRA_USERS2, &INFRA_USERS3, &INFRA_USERS4)
   and OWNER = 'PUBLIC'
 group by TABLE_OWNER
 order by TABLE_OWNER;

exit
```

# Ziel-DB: Nacharbeiten
```sql
sq
@selbct

@selflash
alter database flashback on;
@crspool nacharbeiten
@load_ZKB_OIDCONTROL.sql
@load_ZKB_AUTHCONTROL.sql
@load_ZKB_AUTHORIZATION.sql
@load_ZKB_AUTHLOG.sql
@load_ZKB_APPLPATCH_LOG.sql
@load_ZKB_RA_STATUS.sql
exit

# wenn BCT nicht enabled: check_bct.sh

###  Die geladenen Backup-Files sind auf der Ziel-DB im ASM direkt unter +DATA01 verlinkt und in der DB auch so referenziert
###  diese Pfade müssen in der DB und im ASM korrigiert werden:
 
# Check:
sudo -u grid /app/grid/product/*/bin/asmcmd ls -l DATA01/*.dbf | grep "${BE_ORA_DB_UNIQUE_NAME}"       # Files direkt unter +DATA01 vorhanden
selsess -S seldataf                                                                                 # TTS DB-Files direkt unter +DATA01 referenziert
 
# Erstellen der SQL-Rename-Befehle, z.B.
sudo -u grid /app/grid/product/*/bin/asmcmd ls -l DATA01/*.dbf | grep "${BE_ORA_DB_UNIQUE_NAME}" | awk '$1~/DATAFILE/ {printf("alter database rename file \047+DATA01/%s\047 to \047%s\047;\n",$(NF-2),$NF)}'  | tee /tmp/rename_files_${BE_ORA_DB_UNIQUE_NAME}.sql

# diese SQL's auf der gemounteten DB ausführen
cat /tmp/rename_files_${BE_ORA_DB_UNIQUE_NAME}​.sql​ | sqlplus -l "/ as sysdba"
 
# anschliessend die Links (Aliase) im ASM als User grid entfernen (Vorsicht: nicht mit "rm" sondern mit "rmalias"!), generieren der asmcmd Befehle:
sudo -u grid /app/grid/product/*/bin/asmcmd ls -l DATA01/*.dbf | grep "${BE_ORA_DB_UNIQUE_NAME}" | awk '$1~/DATAFILE/ {printf("asmcmd rmalias +DATA01/%s\n",$(NF-2))}'  | tee /tmp/remove_aliases_${BE_ORA_DB_UNIQUE_NAME}.asm
chmod a+r /tmp/remove_aliases_${BE_ORA_DB_UNIQUE_NAME}.asm

# als User grid diese asmcmd Befehle ausführen:
DB=xxx     # DB-Unique-Name entsprechend setzen
cat /tmp/remove_aliases_${DB}.asm | asmcmd

# Check:
sudo -u grid /app/grid/product/*/bin/asmcmd ls -l DATA01/*.dbf | grep "${BE_ORA_DB_UNIQUE_NAME}"    # keine Files direkt unter +DATA01 vorhanden
selsess -S seldataf                                                                                  # TTS DB-Files im korrekten ASM-Pfad
```

# Ziel-DB: abschliessende Arbeiten:
```bash
auth.pl -refresh

backup_db.sh

 
dg_finish.sh

## oradeploy-Server: 
# DB-Backup wieder aktivieren:
<sid>    # DB Env setzen
sudo dbcron -l
sudo dbcron -s ${ORACLE_SID} -a
```
  
# Abschluss

## Source-DB: Tablespaces auf read-write setzen, Appl.-User sperren

# Source-DB: 

sq
-- Job_queue_processes to original value and scheduled jobs re-enable (value=60)
show parameter job_queue_processes
alter system set job_queue_processes=60 scope=memory;
 
-- Setzen der TBS wieder zurück auf read-write (falls gewünscht)
 
@seltbs

set lines 300 pages 100 serverout on
DECLARE
   CURSOR c1 IS       
     select tablespace_name tbs from DBA_TABLESPACES where tablespace_name not in ('SYSTEM','SYSAUX','UNDOTBS1','TEMP','USERS','SYSAUD') and status='READ ONLY' order by 1;
   stmt varchar2(1000);
BEGIN
   FOR r1 IN c1
   LOOP
      dbms_output.put_line('setting TBS read-write: '||r1.tbs);
      stmt := 'alter tablespace '||r1.tbs||' read write';
       EXECUTE IMMEDIATE stmt;
   END LOOP;
END;
/

@seltbs

-- Sperren der Appl.-User
@lock_unlock.sql
exit

# Tags:  
  [[xTTS] - [[Export]] - [[Clone]] - [[Migration]] - [[CrossEndianess]] - [[HowTo]]
