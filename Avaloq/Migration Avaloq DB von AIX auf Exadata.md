
# Pre Migration

## Neue DB löschen und initialisieren

## Sichern von AAA ENV

```bash
mkdir /u01/patches/aaa_backup
tar cvf /u01/patches/aaa_backup/aaaenv.tar /usr/app/avaloq/
tar rvf /u01/patches/aaa_backup/aaaenv.tar /var/opt/oracle/
```

## DB löschen

```bash
WSA01
dbca -silent -deleteDatabase -sourceDB WSA01 -sysDBAUserName sys -sysDBAPassword $(pwfetch -o sys)
```

# DB bauen

@ansible-ora.prod.zkb.ch

```bash
ap ./create-db.yml --limit=exa102s54.prod.zkb.ch -e "DBNAME=WSA01 dbname=WSA01 DBTYPE=SINGLEDG dbtype=SI OHOME=/app/oracle/product/12.1.0.2.171017.113 ZKB_DEBUG=1 WSA=1" -v
```

## Vorarbeiten

AUD und AUD_ITEM Partitions moven

```sql
sqlplus / as sysdba <<EOSQL
set heading on echo on verify on feedback on trimspool on feedback on timing on
spool move_aud_parts.sql
select 'alter table k.aud_item move partition ' || partition_name || ' tablespace aud_tables;' from dba_segments where segment_name='AUD_ITEM' and tablespace_name != 'AUD_TABLES';
spool off

@crspool move_aud_parts.log
set heading on echo on verify on feedback on trimspool on feedback on timing on
@move_aud_parts.sql
spool off
EOSQL
```

## Archivelog-Backup der Quell-DB

```bash
rman_backup.sh -a
```

> SCN notieren

# Export Quell-DB

## Trigger deaktivieren

Vor dem Export werden sämtliche Trigger deaktiviert, um zu verhindern, das diese beim Laden der Daten Verarbeitungen auslösen. Ausserdem wird die Import-Performance dadurch verbessert.

Sollzustand sichern

Der aktuelle Zustand aller Trigger wird gesichert, um ihn hinterher restoren zu können:

```bash
sqlplus -l -s / as sysdba <<EOSQL
  spool restore_trigger.sql
  set heading on echo on verify on feedback on trimspool on lines 180 pages 0
  select 'alter trigger ' || owner || '.' || trigger_name || ' ' || ecode(status,'ENABLED','ENABLE','DISABLED','DISABLE') || ';'
  from dba_triggers where owner in ('K','X','JOB_EXEC');
  spool off
EOSQL
```

Das Spool-File wird später sowohl auf der Quelldatenbank nach dem Export, als auch nach dem Migrieren auf der Zieldatenbank ausgeführt.

## Deaktivieren aller Trigger

Trigger vor dem Export deaktivieren damit sie beim Laden inaktiv sind:

```sql
sqlplus -l -s / as sysdba <<EOSQL
  set heading off echo off verify off feedback off trimspool off lines 180 pages 0
  spool disable_triggers.sql
  set heading on echo on verify on feedback on trimspool on lines 180 pages 0

  select 'alter trigger ' || owner || '.' || trigger_name || ' disable;' from dba_triggers where owner in ('K','X','JOB_EXEC') and status != 'DISABLED';

  spool off

  set heading on echo on verify on feedback on trimspool on lines 180 pages 0
  @crspool disable_triggers.log
  @disable_triggers.sql
  @disable_triggers.sql

  spool off

  col owner format a40
  set lines 180
  select owner,status,count(1) from dba_triggers where owner in ('K','X','JOB_EXEC') group by owner,status;
EOSQL
```

Das Spool-File muss zweimal laufen, da beim ersten Lauf noch Schutztrigger das deaktivieren von Triggern verhindern, welche beim zweiten Lauf nicht mehr aktiv sind. 

## System-Trigger deaktivieren

```sql
alter system set "_system_trig_enabled"=FALSE scope=memory;
```

## Scheduler deaktivieren

Das Deaktivieren des Schedulers und der Job Queue Prozesse soll eine Verarbeitung auf der Datenbank während des Exports verhindern.

```sql
alter system set job_queue_processes=0 scope=spfile;
exec dbms_scheduler.set_scheduler_attribute('SCHEDULER_DISABLED','TRUE');
```

## UNDO Retention erhöhen

Die UNDO Retention wird auf 24 Stunden konfiguriert um ORA-1555 Fehlermeldungen während des Exports zu verhindern.

```bash
echo "alter system set undo_retention=86400;" | sqlplus / as sysdba
```

Parameterfile (exclude.par)

```bash
#EXCLUDE=INDEX
#EXCLUDE=CONSTRAINT
#EXCLUDE=REF_CONSTRAINT
EXCLUDE=STATISTICS
EXCLUDE=TABLE:"IN(select table_name from dba_tables where table_name='AUD_ITEM' and owner='K')"
```

## Export

```bash
time ZKB_EI_FILESIZE=256G expdp_full_db.sh -e $PWD -l $PWD -f fullexport.dmp -p 96 -P $PWD/exclude.par -c
```

Die Anzahl der maximal möglichen parallelen Exportprozesse kann Urs Stahel aufgrund der verfügbaren Hardware ermitteln.

Dauer im IT ca. 7 Stunden

Laufzeit im ST mit "-p 32": aix2569.st.zkb.ch -> OpenNet Adapter 1GBit / StoreNet 10GBit -> ZFS via StoreNet gemounted. -> 10:32h

## migrate.sh

```bash
migrate.sh -e $PWD -i $PWD -l $PWD -f exp.dmp -t ${_SID} -a
```

create_asm_tablespace.sql -> U* und I* smallfile nach bigfile ändern; nur ein File mit maxsize unlimited; uniform allocation durch autoallocate ersetzen

create_user.sql -> suchen nach "ENABLE EDITIONS" und Semikolon in Zeile obendrüber anhängen

# Neue DB erstellen

mittels Ansible create-db & set-db-param Playbook.

```bash
export _SID=WSA01G
export _BASEDIR=/WSAMIG/AVALOQ01/fullexp
export _MIGRATEDIR=${_BASEDIR}/migrate
export _DUMPDIR=${_BASEDIR}
export _SKRIPTDIR=${_BASEDIR}/skripte
export _LINKNAME=avaloq01.prod.zkb.ch
export _LOGDIR=${_BASEDIR}/log

echo $_SID

ls -lad $_BASEDIR
ls -lad $_MIGRATEDIR
ls -lad $_DUMPDIR
ls -lad $_SKRIPTDIR

echo $_LINKNAME
```

Nach dem Export alten Zustand der Quell-DB wiederherstellen (Trigger, JOB_QUEUE_PROCESSES, Scheduler enabled etc.).

**Auf dem Ziel-DBSRV**

vi ${_SKRIPTDIR}/cr_dblink.sql

> anpassen an Source-DB (Connectstring, Passwort etc.)

# Anpassungen an Ziel-DB

# Flashback & Archivelogging ausschalten:

```sql
srvctl stop database -d ${_SID} -v
srvctl start database -d ${_SID} -startoption mount -v

sqlplus / as sysdba <<EOSQL
  @crspool ${_LOGDIR}/change_db_config.log
  alter database flashback off;
  alter database noarchivelog;
  alter system set sga_max_size=64G scope=spfile;
  alter system set sga_target=64G scope=spfile;
  alter system set pga_aggregate_limit=16G scope=spfile;
  alter system set pga_aggregate_target=8G scope=spfile;
  alter system set parallel_degree_limit=14 scope=spfile;
  alter system set parallel_adaptive_multi_user=FALSE scope=spfile;
  alter system set parallel_degree_policy=AUTO scope=spfile;
  alter system set O7_DICTIONARY_ACCESSIBILITY=TRUE scope=spfile;
  alter system set db_files=5000 scope=spfile;
  alter system set max_dump_file_size='10240' scope=spfile;
  alter system set open_cursors=5000 scope=spfile;
  alter system set processes=4000 scope=spfile;
  alter system set sessions=1600 scope=spfile;
  alter system set undo_retention=86400 scope=spfile;
EOSQL

srvctl stop database -d ${_SID} -v
srvctl start database -d ${_SID} -v

sqlplus / as sysdba <<EOSQL
  @crspool ${_LOGDIR}/deactivate_schedules.log
  @${_SKRIPTDIR}/cr_dblink.sql

  exec dbms_stats.gather_system_stats('EXADATA');

  alter system set "_system_trig_enabled"=FALSE scope=both;
  alter system set job_queue_processes=0 scope=spfile;

  exec dbms_scheduler.set_scheduler_attribute('SCHEDULER_DISABLED','TRUE');
EOSQL
```

## Fake PW-Function anlegen

```bash
sqlplus / as sysdba <<EOSQL
  @${_SKRIPTDIR}/fake_pwdVerifyFunc-wsa.sql
EOSQL
```

## Directories anlegen

```sql
sqlplus / as sysdba <<EOSQL
  set heading off echo off verify off feedback off trimspool on pages 100 lines 180
  spool ${_MIGRATEDIR}/cr_directories.sql
  select 'create or replace directory ' || directory_name || ' as ''' || directory_path || ''';'
  from dba_directories@${_LINKNAME};
  spool off
  set heading on echo on verify on feedback on trimspool on pages 100 lines 180
  @crspool ${_MIGRATEDIR}/cr_directories.log
  @${_MIGRATEDIR}/cr_directories.sql
  spool off
EOSQL
```

--> noch verbessern (geänderte SID berücksichtigen)

## TEMP / UNDO Tablespace

Wenn noch nicht Bigfile sollte das TEMP Tablespace als Bigfile Tablespace angelegt werden. Das macht das Handling des Datafiles erheblich einfacher bei einer Grösse wie der des Avaloq TEMP TS.

## UNDO TBS vergrössern (kein BIGFILE-TBS)

```bash
echo "select BIGFILE from dba_tablespaces where TABLESPACE_NAME = 'TEMP';" | sqlplus -s -l "/ as sysdba"

sqlplus / as sysdba <<EOSQL
  create bigfile temporary tablespace tempxx tempfile size 1G autoextend on next 1G maxsize 512G;
  alter database default temporary tablespace tempxx;
  drop tablespace temp including contents and datafiles;
  create bigfile temporary tablespace temp tempfile size 1G autoextend on next 1G maxsize 512G;
  alter database default temporary tablespace temp;
  drop tablespace tempxx including contents and datafiles;
  alter tablespace UNDOTBS1 add datafile '+DATA01' size 31g;
EOSQL
```

# Profiles

```sql
sqlplus / as sysdba <<EOSQL
  @${_MIGRATEDIR}/create_profile.sql
EOSQL
```

## migrate1.sql

```sql
sqlplus / as sysdba <<EOSQL
  @${_MIGRATEDIR}/migrate1.sql
  @${_MIGRATEDIR}/create_user.sql
  @${_MIGRATEDIR}/cr_user_sys_grants.sql
  create or replace directory skript_dir as '${_SKRIPTDIR}';
EOSQL
```

## Quotas von Quelle übernehmen

```sql
sqlplus / as sysdba <<EOSQL
  set heading on echo on verify on feedback on trimspool on
  spool ${_MIGRATEDIR}/quotas.sql
  select 'alter user ' || username || ' quota ' || bytes || ' on ' || tablespace_name || ';' from dba_ts_quotas@${_LINKNAME};
  spool off

  @crspool ${_MIGRATEDIR}/quotas.log
  @${_MIGRATEDIR}/quotas.sql

  alter user audsys quota unlimited on sysaux;
  spool off
EOSQL
```

## Tablespaces prüfen

```sql
sqlplus / as sysdba <<EOSQL
select tablespace_name from dba_tablespaces@${_LINKNAME}
minus
select tablespace_name from dba_tablespaces;
EOSQL
```

## Fehlende Tablespace noch anlegen!

```sql
sqlplus / as sysdba <<EOSQL
  create tablespace indx datafile size 1G autoextend on next 256M maxsize unlimited extent management local autoallocate segment space management auto;
  create tablespace example datafile size 1G autoextend on next 256M maxsize unlimited extent management local autoallocate segment space management auto;
  create tablespace tools datafile size 1G autoextend on next 12G maxsize unlimited extent management local autoallocate segment space management auto;
  create tablespace perfstat datafile size 1G autoextend on next 12G maxsize unlimited extent management local autoallocate segment space management auto;
  create tablespace sysaud datafile size 1G autoextend on next 12G maxsize unlimited extent management local autoallocate segment space management auto;
EOSQL
```

## USERS Tablespace ggf. nach Bigfile umbauen

```sql
sqlplus / as sysdba <<EOSQL
  @crspool ${_LOGDIR}/recreate_users.log
  alter database default tablespace sysaux;
  drop tablespace users including contents and datafiles;
  create bigfile tablespace users datafile size 1G autoextend on next 512M maxsize unlimited extent management local autoallocate segment space management auto;
  alter database default tablespace users;
EOSQL
```

```sql
sqlplus / as sysdba <<EOSQL
  @crspool ${_LOGDIR}/resize_datafiles.log
  alter database datafile 1 autoextend on next 1G maxsize unlimited;
  alter database datafile 2 autoextend on next 1G maxsize unlimited;
  alter database datafile 3 autoextend on next 1G maxsize unlimited;
  alter database datafile 4 autoextend on next 1G maxsize unlimited;
  alter database tempfile 1 autoextend on next 1G maxsize unlimited;
EOSQL
```

Darauf achten das alle Tablespaces ausreichend Platz haben (ggf. Bigfile mit unlimited maxsize).

```bash
echo "@${_MIGRATEDIR}/create_privilege.sql" | sqlplus / as sysdba
echo "@${_MIGRATEDIR}/cr_user_sys_grants.sql" | sqlplus / as sysdba
```

## Types importieren

```bash
time impdp directory=skript_dir network_link=${_LINKNAME} INCLUDE=TYPE SCHEMAS=K,X logfile=create_types.log userid=\"/ as sysdba\"
```

-> Laufzeit ca. 5 Min.

## CONTEXT's importieren

Auf Quell-DB:

```sql
sqlplus -l -s / as sysdba <<EOSQL
  set long 1000000 pages 1000 lines 180
  col DDL format a170 word_wrapped
  spool ${_MIGRATEDIR}/cr_context.sql replace
  select dbms_metadata.get_ddl('CONTEXT',namespace) || ';' DDL from dba_context;
  spool off
EOSQL
```

Auf Ziel-DB:

```sql
sqlplus -l -s / as sysdba <<EOSQL
  @crspool ${_MIGRATEDIR}/cr_context.log
  set heading on echo on verify on feedback on trimspool on
  @${_MIGRATEDIR}/cr_context.sql
  spool off
EOSQL
```

## Userlist erstellen

### Userlist aus ${_MIGRATEDIR}/exp.par erstellen

```bash
echo "SCHEMAS=" >>${_DUMPDIR}/userlist.par
cat ${_MIGRATEDIR}/exp.par | egrep -v '(^USERID|^FILE|^LOG|^GRANTS|^CONSTRAINTS|^INDEXES|^TRIGGERS|^ROWS|^BUFFER|^CONSISTENT|^OWNER=|^STATISTICS)' >>${_DUMPDIR}/userlist.par
```

userlist.par auf syntaktische Korrektheit prüfen!

### OJVMSYS aus userlist.par entfernen

## Import starten

!!! Den Import in jedem Fall auf einem PCMZ starten !!!

```bash
echo "transform=disable_archive_logging:y" >>${_DUMPDIR}/userlist.par

time impdp_user_db.sh -i ${_DUMPDIR} -l ${_DUMPDIR} -f fullexport_%U.dmp -p 96 -P ${_DUMPDIR}/userlist.par
```

Dauer im ST auf exa201s54: 3:59h

# aud_item exportieren/importieren

## Quell-DB

### Indizes migrieren

#### DDL's erstellen (auf Quell-DB)

mkdir /ZFS-stor/avaloq01/fullexp/indexes
cd /ZFS-stor/avaloq01/fullexp/indexes

```bash
sqlplus -l -s / as sysdba <<EOSQL
  select count(1) from dba_indexes where owner in ('JOB_EXEC','K','X','ZKB') and index_type!='LOB';
EOSQL
```

```bash
sqlplus -l -s / as sysdba <<EOSQL
  drop table dada purge;

  create table dada as select rownum rownr,index_name from dba_indexes where owner in ('JOB_EXEC','K','X','ZKB') and index_type!='LOB';

EOSQL
```

```bash
sqlplus -l -s / as sysdba <<EOSQL >/dev/null 2>&1
  EXECUTE DBMS_METADATA.SET_TRANSFORM_PARAM(DBMS_METADATA.SESSION_TRANSFORM,'STORAGE',false);
  REM # 1-5000
  set pages 0 feedback off trimspool on verify off heading off line 300 termout off long 1000000
  col ddl format a180 word_wrapped
  spool @cr_idx1.sql
  prompt @cr_spool cr_idx1.log

  select
  dbms_metadata.get_ddl('INDEX',index_name,owner) || ' PARALLEL (DEGREE 8);' DDL
  from dba_indexes
  where owner in ('JOB_EXEC','K','X','ZKB')
  and index_type!='LOB'
  and index_name in (select index_name from dada where rownr between 1 and 200);

  prompt spool off
  spool off

  REM # 5001-10000
  set pages 0 feedback off trimspool on verify off heading off line 300 termout off long 1000000
  spool @cr_idx2.sql
  prompt spool cr_idx2.log

  select
  dbms_metadata.get_ddl('INDEX',index_name,owner) || ' PARALLEL (DEGREE 8);' DDL
  from dba_indexes
  where owner in ('JOB_EXEC','K','X','ZKB')
  and index_type!='LOB'
  and index_name in (select index_name from dada where rownr between 201 and 500);

  prompt spool off
  spool off

  REM # 10001-15000
  set pages 0 feedback off trimspool on verify off heading off line 300 termout off long 1000000
  spool @cr_idx3.sql
  prompt spool cr_idx3.log

  select
  dbms_metadata.get_ddl('INDEX',index_name,owner) || ' PARALLEL (DEGREE 8);' DDL
  from dba_indexes
  where owner in ('JOB_EXEC','K','X','ZKB')
  and index_type!='LOB'
  and index_name in (select index_name from dada where rownr between 501 and 2500);

  prompt spool off
  spool off

  REM # 15001-20000
  set pages 0 feedback off trimspool on verify off heading off line 300 termout off long 1000000
  spool @cr_idx4.sql
  prompt spool cr_idx4.log

  select
  dbms_metadata.get_ddl('INDEX',index_name,owner) || ' PARALLEL (DEGREE 8);' DDL
  from dba_indexes
  where owner in ('JOB_EXEC','K','X','ZKB')
  and index_type!='LOB'
  and index_name in (select index_name from dada where rownr between 2501 and 7500);

  prompt spool off
  spool off

  REM # 20001-25000
  set pages 0 feedback off trimspool on verify off heading off line 300 termout off long 1000000
  spool @cr_idx5.sql
  prompt spool cr_idx5.log

  select
  dbms_metadata.get_ddl('INDEX',index_name,owner) || ' PARALLEL (DEGREE 8);' DDL
  from dba_indexes
  where owner in ('JOB_EXEC','K','X','ZKB')
  and index_type!='LOB'
  and index_name in (select index_name from dada where rownr between 7501 and 15000);

  prompt spool off
  spool off

  REM # 25000-26260
  set pages 0 feedback off trimspool on verify off heading off termout off
  spool @cr_idx6.sql
  prompt spool cr_idx6.log

  select
  dbms_metadata.get_ddl('INDEX',index_name,owner) || ' PARALLEL (DEGREE 8);' DDL
  from dba_indexes
  where owner in ('JOB_EXEC','K','X','ZKB')
  and index_type!='LOB'
  and index_name in (select index_name from dada where rownr between 15001 and 30000);

  prompt spool off
  spool off
EOSQL
```

### Indizes erstellen (auf Ziel-DB)

Im Index-Subdir:

```bash
for _FILE in $(ls AVALOQ01cr_idx*.sql)
do
  nohup echo "@${_FILE}" | sqlplus / as sysdba &
done
```

Dauer im ST: 2:15h

### Constraints DDL's exportieren

Pro Constraint-Typ wird ein Spool-File erstellt. Diese werden dann parallel auf der Ziel-DB ausgeführt. Laufzeit ca. 20 Min.

```bash
mkdir ${_DUMPDIR}constraints/
cd ${_DUMPDIR}constraints/
```

```bash
sqlplus -l -s / as sysdba <<EOSQL
SET LONG 1000000 LONGCHUNKSIZE 20000 PAGESIZE 0 LINESIZE 1000 FEEDBACK OFF VERIFY OFF TRIMSPOOL ON
col DDL format a150 word_wrapped

exec DBMS_METADATA.set_transform_param (DBMS_METADATA.session_transform, 'SQLTERMINATOR', true);
exec DBMS_METADATA.set_transform_param (DBMS_METADATA.session_transform, 'PRETTY', true);
exec DBMS_METADATA.set_transform_param (DBMS_METADATA.session_transform, 'STORAGE', false);

set heading off echo off verify off feedback off trimspool on
spool u_constraints.sql replace

SELECT DBMS_METADATA.get_ddl ('CONSTRAINT', constraint_name, owner) DDL
FROM   all_constraints
WHERE  owner      in ('X','K','JOB_EXEC','ZKB')
AND    constraint_type IN ('U');

spool off

spool p_constraints.sql replace

SELECT DBMS_METADATA.get_ddl ('CONSTRAINT', constraint_name, owner) DDL
FROM   all_constraints
WHERE  owner      in ('X','K','JOB_EXEC','ZKB')
AND    constraint_type IN ('P');

spool off

spool c_constraints.sql replace

SELECT DBMS_METADATA.get_ddl ('CONSTRAINT', constraint_name, owner) DDL
FROM   all_constraints
WHERE  owner      in ('X','K','JOB_EXEC','ZKB')
AND    constraint_type IN ('C');

spool off

spool o_constraints.sql replace

SELECT DBMS_METADATA.get_ddl ('CONSTRAINT', constraint_name, owner) DDL
FROM   all_constraints
WHERE  owner      in ('X','K','JOB_EXEC','ZKB')
AND    constraint_type IN ('O');

spool off

spool r_constraints.sql replace

SELECT DBMS_METADATA.get_ddl ('REF_CONSTRAINT', constraint_name, owner) DDL
FROM   all_constraints
WHERE  owner      in ('X','K','JOB_EXEC','ZKB')
AND    constraint_type IN ('R');

spool off
EOSQL
```

### Table Degree spoolen

Der Table-Degree wird aus der Quelldatenbank gespoolt um ihn später in der Ziel-DB wiederherstellen zu können. Während der Erstellung der Constraints wird der Degree auf den Tabellen der Ziel-DB erhöht.

Auf Quell-DB:

im Indexes Verzeichnis

```bash
sqlplus -l -s / as sysdba <<EOSQL

SET LONG 1000000 LONGCHUNKSIZE 20000 PAGESIZE 0 LINESIZE 1000 FEEDBACK OFF VERIFY OFF TRIMSPOOL ON

set heading off echo off verify off feedback off trimspool on

spool restore_degree.sql replace

select 'alter table ' || owner || '.' || table_name || ' parallel (degree ' || trim(degree) || ');' from dba_tables where owner in ('X','K','JOB_EXEC','ZKB','SLB','ZKB','KOBL');

select 'alter index ' || owner || '.' || index_name || ' parallel (degree ' || trim(degree) || ');' from dba_indexes where owner in ('X','K','JOB_EXEC','ZKB','SLB','ZKB','KOBL');

spool off

spool chg_degree.sql replace

select 'alter table ' || owner || '.' || table_name || ' parallel (degree 8);' from dba_tables where owner in ('X','K','JOB_EXEC','ZKB','SLB','ZKB','KOBL');

spool off

EOSQL
```

### Table degree umstellen (auf Ziel-DB)

```bash
echo "@chg_degree.sql" | sqlplus -l -s / as sysdba
```

### Constraints erstellen

Primary Constraints erstellen

```bash
time echo "@p_constraints.sql" | sqlplus / as sysdba
```

Laufzeit im ST: 1:04h

### Restliche Constraints erstellen

```bash
for _FILE in $(ls *constraints.sql | grep -v '^p_')
do
  time echo "@${_FILE}" | sqlplus / as sysdba >${_FILE}.log 2>&1 &
done
```

Dauer im ST: 1:05h

Da die referential constraints nur erstellt werden können, wenn ein entsprechender primary key existiert, muss das Skript r_constraints zweimal laufen.

```bash
sqlplus -l -s / as sysdba <<EOSQL
  set heading on echo on verify on feedback on lines 180 trimspool on
  @crspool r_constraints.log
  @r_constraints.sql
EOSQL
```

### Status prüfen

```sql
set lines 180 pages 0
col owner format a40

select owner,constraint_type,count(1) from dba_constraints where owner in ('X','K','JOB_EXEC','ZKB','SLB','ZKB','KOBL') group by owner,constraint_type order by 1,2;
```

## Privileges

```bash
echo "@${_MIGRATEDIR}/create_privilege.sql" | sqlplus / as sysdba
echo "@${_MIGRATEDIR}/cr_user_sys_grants.sql" | sqlplus / as sysdba
```

## Perfstat installieren

```bash
@?/rdbms/admin/spcreate.sql
```

## HPROF installieren

```bash
sqlplus / as sysdba <<EOSQL
  GRANT EXECUTE ON dbms_hprof TO K;

  CREATE OR REPLACE DIRECTORY profiler_dir AS '/app/oracle/admin/WSA01/diag/rdbms/wsa01*/WSA01/trace/';
  GRANT READ, WRITE ON DIRECTORY profiler_dir TO K;

  create public synonym DBMSHP_PARENT_CHILD_INFO for sys.DBMSHP_PARENT_CHILD_INFO;

  grant select on sys.DBMSHP_PARENT_CHILD_INFO to k with grant option;

  create public synonym dbmshp_function_info  for sys.dbmshp_function_info;

  grant select on sys.dbmshp_function_info  to k with grant option;
EOSQL
```

```bash
sqlplus k/$(pwfetch -o k) <<EOSQL
  @?/rdbms/admin/dbmshptab.sql
EOSQL
```

## Table & Index-Degree zurücksetzen

```bash
sqlplus / as sysdba <<EOSQL
  @crspool ${_LOG}/indexes/restore_degree.log
  set heading on echo on verify on feedback on trimspool on

  @${_DUMPDIR}/indexes/restore_degree.sql
EOSQL
```

## Trigger zurücksetzen

```bash
echo "@${_DUMPDIR}/trigger/restore_trigger.sql" | sqlplus / as sysdba
```

## Java Grants

```sql
sqlplus / as sysdba <<EOSQL
exec dbms_java.grant_permission( 'X', 'SYS:java.lang.RuntimePermission', 'getenv.ORACLE_DB_DATA', '' )

DECLARE
KEYNUM NUMBER;
BEGIN
  SYS.DBMS_JAVA.GRANT_PERMISSION(
     grantee           => 'X'
    ,permission_type   => 'SYS:java.lang.RuntimePermission'
    ,permission_name   => 'accessDeclaredMembers'
    ,permission_action => ''
    ,key               => KEYNUM
    );
END;
/

DECLARE
KEYNUM NUMBER;
BEGIN
  SYS.DBMS_JAVA.GRANT_PERMISSION(
     grantee           => 'X'
    ,permission_type   => 'SYS:java.lang.reflect.ReflectPermission'
    ,permission_name   => 'suppressAccessChecks'
    ,permission_action => ''
    ,key               => KEYNUM
    );
END;
/

DECLARE
KEYNUM NUMBER;
BEGIN
  SYS.DBMS_JAVA.GRANT_PERMISSION(
     grantee           => 'X'
    ,permission_type   => 'SYS:java.lang.RuntimePermission'
    ,permission_name   => 'getenv.*'
    ,permission_action => ''
    ,key               => KEYNUM
    );
END;
/
EOSQL
```

## Synonyme erstellen

```bash
sqlplus / as sysdba <<EOSQL
  CREATE OR REPLACE SYNONYM X.INSTALLER_CONFIG# FOR X.I2#B#INSTALLER_CONFIG#;
  CREATE OR REPLACE SYNONYM X.INSTALL_BOOT# FOR X.I2#B#INSTALL_BOOT#;
  CREATE OR REPLACE SYNONYM X.INSTALL_BOOT_LOG# FOR X.I2#B#INSTALL_BOOT_LOG#;
  CREATE OR REPLACE SYNONYM X.INSTALL_DBA_DDL_LOCKS FOR X.I2#B#INSTALL_DBA_DDL_LOCKS;
EOSQL
```

## Job Classes und Jobs erstellen

```sql
sqlplus / as sysdba <<EOSQL

BEGIN
  SYS.DBMS_SCHEDULER.DROP_JOB_CLASS
    (
      job_class_name   => 'SYS.AVQ$INSTALL_UTIL_JOB_CLASS'
     ,force            => TRUE
    );
END;
/

BEGIN
  SYS.DBMS_SCHEDULER.CREATE_JOB_CLASS
    (
      job_class_name          => 'AVQ$INSTALL_UTIL_JOB_CLASS'
     ,resource_consumer_group => NULL
     ,service                 => NULL
     ,logging_level           => SYS.DBMS_SCHEDULER.LOGGING_FULL
     ,log_history             => 30
     ,comments                => 'Job Class for debugger'
    );
END;
/

BEGIN
  SYS.DBMS_SCHEDULER.DROP_JOB_CLASS
    (
      job_class_name   => 'SYS.AVQ$INSTALL_SPOOL_JOB_CLASS'
     ,force            => TRUE
    );
END;
/

BEGIN
  SYS.DBMS_SCHEDULER.CREATE_JOB_CLASS
    (
      job_class_name          => 'AVQ$INSTALL_SPOOL_JOB_CLASS'
     ,resource_consumer_group => NULL
     ,service                 => NULL
     ,logging_level           => SYS.DBMS_SCHEDULER.LOGGING_FULL
     ,log_history             => 30
     ,comments                => 'Job Class for INSTALL_SPOOL_JOB'
    );
END;
/

BEGIN
  SYS.DBMS_SCHEDULER.DROP_JOB_CLASS
    (
      job_class_name   => 'SYS.AVQ$INSTALL_JOB_CLASS'
     ,force            => TRUE
    );
END;
/

BEGIN
  SYS.DBMS_SCHEDULER.CREATE_JOB_CLASS
    (
      job_class_name          => 'AVQ$INSTALL_JOB_CLASS'
     ,resource_consumer_group => NULL
     ,service                 => NULL
     ,logging_level           => SYS.DBMS_SCHEDULER.LOGGING_FULL
     ,log_history             => 30
     ,comments                => 'Job Class for INSTALL_JOB'
    );
END;
/

BEGIN
  SYS.DBMS_SCHEDULER.DROP_JOB_CLASS
    (
      job_class_name   => 'SYS.AVQ$INSTALLER_CLASS_MASTER1'
     ,force            => TRUE
    );
END;
/

BEGIN
  SYS.DBMS_SCHEDULER.CREATE_JOB_CLASS
    (
      job_class_name          => 'AVQ$INSTALLER_CLASS_MASTER1'
     ,resource_consumer_group => NULL
     ,service                 => NULL
     ,logging_level           => SYS.DBMS_SCHEDULER.LOGGING_FULL
     ,log_history             => 30
     ,comments                => 'Job Class for Avaloq Installer Master 1'
    );
END;
/

BEGIN
  SYS.DBMS_SCHEDULER.DROP_JOB_CLASS
    (
      job_class_name   => 'SYS.AVQ$INSTALL_INSTN_JOB_CLASS'
     ,force            => TRUE
    );
END;
/

BEGIN
  SYS.DBMS_SCHEDULER.CREATE_JOB_CLASS
    (
      job_class_name          => 'AVQ$INSTALL_INSTN_JOB_CLASS'
     ,resource_consumer_group => NULL
     ,service                 => NULL
     ,logging_level           => SYS.DBMS_SCHEDULER.LOGGING_FULL
     ,log_history             => 30
     ,comments                => 'Job Class for job running at specified Instance'
    );
END;
/

BEGIN
  SYS.DBMS_SCHEDULER.DROP_JOB_CLASS
    (
      job_class_name   => 'SYS.AVQ$INSTALL_DEBUG_JOB_CLASS'
     ,force            => TRUE
    );
END;
/

BEGIN
  SYS.DBMS_SCHEDULER.CREATE_JOB_CLASS
    (
      job_class_name          => 'AVQ$INSTALL_DEBUG_JOB_CLASS'
     ,resource_consumer_group => NULL
     ,service                 => NULL
     ,logging_level           => SYS.DBMS_SCHEDULER.LOGGING_FULL
     ,log_history             => 30
     ,comments                => 'Job Class for debugger'
    );
END;
/

BEGIN
  SYS.DBMS_SCHEDULER.DROP_JOB_CLASS
    (
      job_class_name   => 'SYS.AVQ$INSTALLER_CLASS_MASTER0'
     ,force            => TRUE
    );
END;
/

BEGIN
  SYS.DBMS_SCHEDULER.CREATE_JOB_CLASS
    (
      job_class_name          => 'AVQ$INSTALLER_CLASS_MASTER0'
     ,resource_consumer_group => NULL
     ,service                 => NULL
     ,logging_level           => SYS.DBMS_SCHEDULER.LOGGING_FULL
     ,log_history             => 30
     ,comments                => 'Job Class for Avaloq Installer Master 0'
    );
END;
/

BEGIN
  SYS.DBMS_SCHEDULER.DROP_JOB_CLASS
    (
      job_class_name   => 'SYS.AVQ$INSTALLER_CLASS_MASTER2'
     ,force            => TRUE
    );
END;
/

BEGIN
  SYS.DBMS_SCHEDULER.CREATE_JOB_CLASS
    (
      job_class_name          => 'AVQ$INSTALLER_CLASS_MASTER2'
     ,resource_consumer_group => NULL
     ,service                 => NULL
     ,logging_level           => SYS.DBMS_SCHEDULER.LOGGING_FULL
     ,log_history             => 30
     ,comments                => 'Job Class for Avaloq Installer Master 2'
    );
END;
/

BEGIN
  SYS.DBMS_SCHEDULER.DROP_JOB_CLASS
    (
      job_class_name   => 'SYS.AVQ$INSTALLER_CLASS_MASTER3'
     ,force            => TRUE
    );
END;
/

BEGIN
  SYS.DBMS_SCHEDULER.CREATE_JOB_CLASS
    (
      job_class_name          => 'AVQ$INSTALLER_CLASS_MASTER3'
     ,resource_consumer_group => NULL
     ,service                 => NULL
     ,logging_level           => SYS.DBMS_SCHEDULER.LOGGING_FULL
     ,log_history             => 30
     ,comments                => 'Job Class for Avaloq Installer Master 3'
    );
END;
/

BEGIN
  SYS.DBMS_SCHEDULER.DROP_JOB_CLASS
    (
      job_class_name   => 'SYS.AVQ$INSTALLER_CLASS_MASTER4'
     ,force            => TRUE
    );
END;
/

BEGIN
  SYS.DBMS_SCHEDULER.CREATE_JOB_CLASS
    (
      job_class_name          => 'AVQ$INSTALLER_CLASS_MASTER4'
     ,resource_consumer_group => NULL
     ,service                 => NULL
     ,logging_level           => SYS.DBMS_SCHEDULER.LOGGING_FULL
     ,log_history             => 30
     ,comments                => 'Job Class for Avaloq Installer Master 4'
    );
END;
/

BEGIN
  SYS.DBMS_SCHEDULER.DROP_JOB_CLASS
    (
      job_class_name   => 'SYS.AVQ$INSTALLER_CLASS_MASTER5'
     ,force            => TRUE
    );
END;
/

BEGIN
  SYS.DBMS_SCHEDULER.CREATE_JOB_CLASS
    (
      job_class_name          => 'AVQ$INSTALLER_CLASS_MASTER5'
     ,resource_consumer_group => NULL
     ,service                 => NULL
     ,logging_level           => SYS.DBMS_SCHEDULER.LOGGING_FULL
     ,log_history             => 30
     ,comments                => 'Job Class for Avaloq Installer Master 5'
    );
END;
/

BEGIN
  SYS.DBMS_SCHEDULER.DROP_JOB_CLASS
    (
      job_class_name   => 'SYS.AVQ$INSTALLER_CLASS_MASTER6'
     ,force            => TRUE
    );
END;
/

BEGIN
  SYS.DBMS_SCHEDULER.CREATE_JOB_CLASS
    (
      job_class_name          => 'AVQ$INSTALLER_CLASS_MASTER6'
     ,resource_consumer_group => NULL
     ,service                 => NULL
     ,logging_level           => SYS.DBMS_SCHEDULER.LOGGING_FULL
     ,log_history             => 30
     ,comments                => 'Job Class for Avaloq Installer Master 6'
    );
END;
/

BEGIN
  SYS.DBMS_SCHEDULER.DROP_JOB_CLASS
    (
      job_class_name   => 'SYS.AVQ$INSTALLER_CLASS_MASTER7'
     ,force            => TRUE
    );
END;
/

BEGIN
  SYS.DBMS_SCHEDULER.CREATE_JOB_CLASS
    (
      job_class_name          => 'AVQ$INSTALLER_CLASS_MASTER7'
     ,resource_consumer_group => NULL
     ,service                 => NULL
     ,logging_level           => SYS.DBMS_SCHEDULER.LOGGING_FULL
     ,log_history             => 30
     ,comments                => 'Job Class for Avaloq Installer Master 7'
    );
END;
/

BEGIN
  SYS.DBMS_SCHEDULER.DROP_JOB_CLASS
    (
      job_class_name   => 'SYS.AVQ$INSTALLER_CLASS_MASTER8'
     ,force            => TRUE
    );
END;
/

BEGIN
  SYS.DBMS_SCHEDULER.CREATE_JOB_CLASS
    (
      job_class_name          => 'AVQ$INSTALLER_CLASS_MASTER8'
     ,resource_consumer_group => NULL
     ,service                 => NULL
     ,logging_level           => SYS.DBMS_SCHEDULER.LOGGING_FULL
     ,log_history             => 30
     ,comments                => 'Job Class for Avaloq Installer Master 8'
    );
END;
/

BEGIN
  SYS.DBMS_SCHEDULER.DROP_JOB_CLASS
    (
      job_class_name   => 'SYS.AVQ$INSTALLER_CLASS_MASTER9'
     ,force            => TRUE
    );
END;
/

BEGIN
  SYS.DBMS_SCHEDULER.CREATE_JOB_CLASS
    (
      job_class_name          => 'AVQ$INSTALLER_CLASS_MASTER9'
     ,resource_consumer_group => NULL
     ,service                 => NULL
     ,logging_level           => SYS.DBMS_SCHEDULER.LOGGING_FULL
     ,log_history             => 30
     ,comments                => 'Job Class for Avaloq Installer Master 9'
    );
END;
/

BEGIN
  SYS.DBMS_SCHEDULER.CREATE_JOB
    (
       job_name        => 'K.INSTALL_JOB'
      ,start_date      => TO_TIMESTAMP_TZ('2019/03/20 11:30:17.597821 +01:00','yyyy/mm/dd hh24:mi:ss.ff tzr')
      ,repeat_interval => NULL
      ,end_date        => NULL
      ,job_class       => 'AVQ$INSTALL_JOB_CLASS'
      ,job_type        => 'PLSQL_BLOCK'
      ,job_action      => 'begin x.install_mgr#.install#do(x.install_mgr#.install#id); end;'
      ,comments        => NULL
    );

  SYS.DBMS_SCHEDULER.SET_ATTRIBUTE
    ( name      => 'K.INSTALL_JOB'
     ,attribute => 'RESTARTABLE'
     ,value     => FALSE);
  SYS.DBMS_SCHEDULER.SET_ATTRIBUTE
    ( name      => 'K.INSTALL_JOB'
     ,attribute => 'LOGGING_LEVEL'
     ,value     => SYS.DBMS_SCHEDULER.LOGGING_OFF);
  SYS.DBMS_SCHEDULER.SET_ATTRIBUTE_NULL
    ( name      => 'K.INSTALL_JOB'
     ,attribute => 'MAX_FAILURES');
  SYS.DBMS_SCHEDULER.SET_ATTRIBUTE_NULL
    ( name      => 'K.INSTALL_JOB'
     ,attribute => 'MAX_RUNS');
  SYS.DBMS_SCHEDULER.SET_ATTRIBUTE
    ( name      => 'K.INSTALL_JOB'
     ,attribute => 'STOP_ON_WINDOW_CLOSE'
    ,value     => FALSE);
  SYS.DBMS_SCHEDULER.SET_ATTRIBUTE
    ( name      => 'K.INSTALL_JOB'
     ,attribute => 'JOB_PRIORITY'
     ,value     => 3);
  SYS.DBMS_SCHEDULER.SET_ATTRIBUTE_NULL
    ( name      => 'K.INSTALL_JOB'
     ,attribute => 'SCHEDULE_LIMIT');
  SYS.DBMS_SCHEDULER.SET_ATTRIBUTE
    ( name      => 'K.INSTALL_JOB'
     ,attribute => 'AUTO_DROP'
     ,value     => FALSE);
  SYS.DBMS_SCHEDULER.SET_ATTRIBUTE
    ( name      => 'K.INSTALL_JOB'
     ,attribute => 'RESTART_ON_RECOVERY'
     ,value     => FALSE);
  SYS.DBMS_SCHEDULER.SET_ATTRIBUTE
    ( name      => 'K.INSTALL_JOB'
     ,attribute => 'RESTART_ON_FAILURE'
     ,value     => FALSE);
  SYS.DBMS_SCHEDULER.SET_ATTRIBUTE
    ( name      => 'K.INSTALL_JOB'
     ,attribute => 'STORE_OUTPUT'
     ,value     => TRUE);
END;
/
EOSQL
```

### Grants erneut einspielen

```bash
echo "@${_MIGRATEDIR}/create_privilege.sql" | sqlplus / as sysdba
echo "@${_MIGRATEDIR}/cr_user_sys_grants.sql" | sqlplus / as sysdba
```

### Jobs/Scheduler reaktivieren

```sql
sqlplus / as sysdba <<EOSQL
@crspool reactivate_scheduler_jobs.log

alter system set job_queue_processes=1000 scope=spfile;
exec dbms_scheduler.set_scheduler_attribute('SCHEDULER_DISABLED','FALSE');
alter system set "_system_trig_enabled"=TRUE scope=both;

spool off
EOSQL
```

### Archivelogging/Flashback aktivieren

```bash
srvctl stop database -d ${_SID} -v
srvctl start database -d ${_SID} -v -startoption mount
```

```bash
sqlplus / as sysdba <<EOSQL

  @crspool ${_LOGDIR}/change_db_config_postmig.log

  alter database flashback on;

  alter database archivelog;

EOSQL

srvctl stop database -d ${_SID} -v
srvctl start database -d ${_SID} -v
```

MaxSizes der Tablespaces ändern

aktuelle Grösse +20%

Job für Statistikberechnung deaktivieren/nur für Oracle Schemata?

# Tags:

[[HowTo]] - [[Avaloq]]

