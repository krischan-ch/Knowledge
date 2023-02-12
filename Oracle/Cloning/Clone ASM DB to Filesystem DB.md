# Abstract

Clonen einer ASM DB in ein Filesystem mittels ZFS Snapshots/Clones.

# Image Backup der Source DB erstellen

**Login auf Admin VM**
```bash
crontab  -e
```

**Backup Jobs der Source DB deaktivieren**

- Login auf DBServer (als Oracle)
- Basenv sourcen

```bash
mkdir /zfssa/ORA_BACKUP_PROD_101/EAI01/ImageBackup
```

**rcv-File für Image Copy erstellen**

```bash
vi /tmp/backup_FULL_image.rcv
run {
set nocfau;
CONFIGURE BACKUP OPTIMIZATION OFF;

ALLOCATE CHANNEL ch1 DEVICE TYPE DISK;
ALLOCATE CHANNEL ch2 DEVICE TYPE DISK;
ALLOCATE CHANNEL ch3 DEVICE TYPE DISK;
ALLOCATE CHANNEL ch4 DEVICE TYPE DISK;
ALLOCATE CHANNEL ch5 DEVICE TYPE DISK;
ALLOCATE CHANNEL ch6 DEVICE TYPE DISK;
ALLOCATE CHANNEL ch7 DEVICE TYPE DISK;
ALLOCATE CHANNEL ch8 DEVICE TYPE DISK;
ALLOCATE CHANNEL ch9 DEVICE TYPE DISK;
ALLOCATE CHANNEL ch10 DEVICE TYPE DISK;
ALLOCATE CHANNEL ch11 DEVICE TYPE DISK;
ALLOCATE CHANNEL ch12 DEVICE TYPE DISK;
ALLOCATE CHANNEL ch13 DEVICE TYPE DISK;
ALLOCATE CHANNEL ch14 DEVICE TYPE DISK;
ALLOCATE CHANNEL ch15 DEVICE TYPE DISK;
ALLOCATE CHANNEL ch16 DEVICE TYPE DISK;

backup as copy section size 256G database format '/zfssa/ORA_BACKUP_PROD_101/EAI01/ImageBackup/%U'
plus archivelog format '/zfssa/ORA_BACKUP_PROD_101/EAI01/ImageBackup/%U';
backup as copy current controlfile format '/zfssa/ORA_BACKUP_PROD_101/EAI01/ImageBackup/%U';

CONFIGURE BACKUP OPTIMIZATION ON;
}
```

**rcv-File im rman ausführen**
```
rman target / cmdfile=/tmp/backup_FULL_image.rcv log=/tmp/backup_FULL_image_EAI01.log
```
 
# Snapshots / Clones
Login auf zfs (zfs302s01-adm.eng.zkb.ch)
```js
shares
select ORA_CLONE_ENG_302
select DB_CLONE_01
snapshots
snapshot EDUMA01_ANON
select EDUMA01_ANON
clone EDUMA01_ATIN
get
commit
clone EDUMA01_ATOUT
get
commit
list clones
```

-> Share neu mounten auf OVM

**Restore Controlfile**
```bash
rman target /
startup nomount;
restore controlfile from '/zfssa/ORA_CLONE_ENG_302/DB_CLONE_01/EDUMA01/2fqt3b1v_1_1';
alter database mount;
```

**switch datafiles**
```sql
switch datafile 1 to copy;
switch datafile 2 to copy;
switch datafile 3 to copy;
switch datafile 4 to copy;
switch datafile 5 to copy;
switch datafile 6 to copy;
switch datafile 7 to copy;
```
**RedoLog rename**
```sql
sqlplus
set serveroutput on;
declare
    v_dbname           VARCHAR2(8);
    v_dbf_filesystem_1 VARCHAR2(40);
    v_dbf_filesystem_2 VARCHAR2(40);
    v_dbf_filesystem   VARCHAR2(40);
    v_redofile_new     VARCHAR2(100);
    v_redofile_suffix  VARCHAR2(1);
    cursor c_rdolog is select GROUP#, MEMBER from v$logfile;
begin
    select NAME INTO v_dbname from v$database;
    v_dbf_filesystem_1 := '/dbms/oracle/' || v_dbname || '/data01/';   
    v_dbf_filesystem_2 := '/dbms/oracle/' || v_dbname || '/rctl01/';  
    for r_rdolog in c_rdolog loop
        IF substr(r_rdolog.MEMBER,2,6) = 'DATA01' THEN
          v_dbf_filesystem  := v_dbf_filesystem_1;
          v_redofile_suffix := 'b';
        ELSE
          v_dbf_filesystem := v_dbf_filesystem_2;
          v_redofile_suffix := 'a';
        END IF;
        v_redofile_new := v_dbf_filesystem || 'redo_' || r_rdolog.group# || '_' || v_redofile_suffix || '.dbf';
        execute immediate 'alter database rename file '''|| r_rdolog.member || ''' to ''' || v_redofile_new || '''';
    end loop;
end;
/

select * from v$logfile;
```

**rename datafiles**
```sql
declare
    v_dbf_new VARCHAR2(200);
    cursor c_df is select name from v$datafile;
begin
    for r_df in c_df loop
        v_dbf_new := replace(r_df.name, 'DB_CLONE_01', 'EDUMA01_ATIN');
        dbms_output.put_line('alter database rename file ''' || r_df.name || ''' to ''' || v_dbf_new ||''' ;');
        execute immediate 'alter database rename file ''' || r_df.name || ''' to ''' || v_dbf_new ||'''';
    end loop;
end;
/
```
**recover database**
```bash
rman target /
recover database;
```

**open database**
alter database open resetlogs

Tags:
[[HowTo] - [[ZFS Skripttemplate für Massenänderungen]] - [[Snapshot]] - [[Clone]]
