
# Ausgangslage

Data Pump kann bis und mit Oracle 12.2 Tabellen mit LOB's (Basicfiles oder auch Securefiles) nicht parallel exportieren und importieren. Siehe dazu Doc Id 1467662.1.

Bei Oracle 19 werden nur Securefile LOB's parallel verarbeitet, Basicfiles LOB's werden seriell verarbeitet.

# Check

Check, wieviele, wie grosse LOB's und welche Art der LOB's in der DB vorhanden sind:

```sql
@sellob
```

# Workaround

Als Workaround hilft bei Version <=12.2 nur, den Export von grossen LOB-Tabelle(n) manuell zu parallelisieren, d.h. die Daten mit der Data Pump QUERY Klausel aufzuteilen und mit mehreren Export-Jobs und Files parallelisiert abzuarbeiten. Auch der Import dieser Dumpfiles kann normalerweise nicht parallelisiert ablaufen, da beim Daten-Import defaultmässig die ganze Tabelle gelockt wird, sodass jeweils nur ein Import-Job laufen kann - als Workaround muss dafür der Data Pump Parameter DATA_OPTIONS=DISABLE_APPEND_HINT verwendet werden.

Ein Ire Namens Morten Jensen hat sich dazu ein geniales Vorgehen ausgedacht, um die LOB-Tabellen-Exporte dynamisch und effizient aufzuteilen, siehe Blog-Eintrag.

# Ablauf

Paralelisierung: pro Prozess sollte ein CPU-Thread und einige 100mb Shared_pool zur Verfügung stehen.

​Ausserdem sollte der STREAMS_POOL_SIZE auf min 256mb gesetzt werden:

```sql
alter system set shared_pool_size = 8g  scope = memory;
alter system set STREAMS_POOL_SIZE=256m scope = memory;
```

## EXPDP

- full-ex​port ohne Lob-Tabelle(n), vorzugsweise auf den einen zfs Node
- gleichzeitiger export der Daten der Lob Tabelle(n), vorzugsweise auf den anderen zfs Node

## IMPDB

- disable Admin Tasks
- Schema-import
- sobald Lob-Tabellen ohne Daten importiert wurden, gleichzeitiger import der Daten der Lob-Tabellen
- import der Constraints und Indizes​
- berechnen der Statistiken
- enable Admin Tasks

# Vorgehen

Export (alte DB)

Beim Full- (oder Schema-) Export  die grossen LOB-Tabellen ohne Daten exportieren, Bsp:

### Single Table

echo "QUERY=COM_CONTAINER:\"where 1=2\"" > excl_lob_tabs.par

### Multiple Table mit Owner

```bash
echo "QUERY=RSEDBA.COM_CONTAINER:\"where 1=2\",RSEDBA.COM_INPUT_STORAGE​:\"where 1=2\"" > excl_lob_tabs.par

# belegte Jobnamen auflisten

selsess -S seldptab
expdp_full_db.sh -c -P excl_lob_tabs.par -p 10
```

Gleichzeitig zum Export ein Perl-Script bereitstellen (adaptiert von Morten Jensen), Bsp für eine LOB-Tabelle mit 10 parallelen Jobs:

```bash
vi exp_LOBTAB.pl

#!/usr/bin/perl
use strict;
my $PARALLEL=10;
my $OWNER="RSEDBA";
my $TABLE="COM_CONTAINER";
my $PFILE="parallel_table_exp.par";
my @row;
my $sp_flash_time=`sqlplus -S / as sysdba <<EOF
set pages 0 lines 172
col x format a172
set trimout on
set trimspool on
select 'row:' || TO_CHAR(SYSDATE, 'dd/mon/yyyy hh24:mi:ss') x from dual
/
EOF`;

my $rc = $?;
if ($rc) {
  print "sqlplus failed - exiting";
exit $?;
}

my $flash_time;
foreach ( split (/\n/, $sp_flash_time) ) {
chomp;
if ( /^row:/ ) {
  s/^[row://g](row://g);
  $flash_time = $_;
  }
}

if ( !defined ($flash_time) ) {
  print "Unable to get database time - exiting";
}
print "Using flash time of: " . $flash_time . "\n";

open(PARFILE, ">", $PFILE);

print PARFILE "flashback_time=\"to_timestamp('${flash_time}', 'dd/mon/yyyy hh24:mi:ss')\"\n";
print PARFILE "parallel=1\n";
print PARFILE "exclude=statistics\n";
close(PARFILE);
$SIG{CHLD} = 'IGNORE';
my $shard = 0;
my $cmd;

foreach ($shard = 0 ; $shard < $PARALLEL ; $shard++) {
 $cmd = "expdp \\\'/ as sysdba\\\' tables=${OWNER}.${TABLE} dumpfile=DMP_DIR:${TABLE}_${shard}.dmp logfile=LOG_DIR:${TABLE}_${shard}.log JOB_NAME=${TABLE}_${shard} COMPRESSION=ALL CONTENT=DATA_ONLY"; 
 $cmd .= " parfile=${PFILE}";
 $cmd .= " query=${OWNER}.${TABLE}:\'\"where mod(dbms_rowid.rowid_block_number(rowid), ${PARALLEL}) = " . $shard . "\"\'";
 $cmd .= " &";
 print "Starting: $cmd\n";
 my $cpid = system($cmd);
}

oder mit mehreren Lob Tabellen gleichzeitig:

#!/usr/bin/perl

use strict;
my $PARALLEL=8;
my $DMPNAME="RSE_LOB_TABS";
my $TABLES="RSEDBA.COM_CONTAINER,RSEDBA.COM_INPUT_STORAGE";
my $PFILE="parallel_tables_exp.par";
my @row;
my $sp_flash_time=`sqlplus -S / as sysdba <<EOF
set pages 0 lines 172
col x format a172
set trimout on
set trimspool on
select 'row:' || TO_CHAR(SYSDATE, 'dd/mon/yyyy hh24:mi:ss') x from dual
/
EOF`;

my $rc = $?;
if ($rc) {
 print "sqlplus failed - exiting";
 exit $?;
}

my $flash_time;

foreach ( split (/\n/, $sp_flash_time) ) {
chomp;

 if ( /^row:/ ) {
  s/^[row://g](row://g);
  $flash_time = $_;
 }
}

if ( !defined ($flash_time) ) {
 print "Unable to get database time - exiting";
}
print "Using flash time of: " . $flash_time . "\n";

open(PARFILE, ">", $PFILE);

print PARFILE "flashback_time=\"to_timestamp('${flash_time}', 'dd/mon/yyyy hh24:mi:ss')\"\n";
print PARFILE "parallel=1\n";
print PARFILE "exclude=statistics\n";
close(PARFILE);
$SIG{CHLD} = 'IGNORE';
my $shard = 0;
my $cmd;

foreach ($shard = 0 ; $shard < $PARALLEL ; $shard++) {
 $cmd = "expdp \\\'/ as sysdba\\\' tables=${TABLES} dumpfile=DMP_DIR:${DMPNAME}_${shard}.dmp logfile=LOG_DIR:${DMPNAME}_${shard}.log JOB_NAME=${DMPNAME}_${shard} COMPRESSION=ALL CONTENT=DATA_ONLY"; 
 $cmd .= " parfile=${PFILE}";
 $cmd .= " query=\'\"where mod(dbms_rowid.rowid_block_number(rowid), ${PARALLEL}) = " . $shard . "\"\'";
 $cmd .= " &";

 print "Starting: $cmd\n";
 my $cpid = system($cmd);
}

oder mit einer Tabelle und einer query Bedingung:

#!/usr/bin/perl
use strict;
my $PARALLEL=10;
my $OWNER="FIMASDBA";
my $TABLE="SECURITY_QUOTE";
my $PFILE="parallel_table4_exp.par";
my @row;
my $sp_flash_time=`sqlplus -S / as sysdba <<EOF
set pages 0 lines 172
col x format a172
set trimout on
set trimspool on
select 'row:' || TO_CHAR(SYSDATE, 'dd/mon/yyyy hh24:mi:ss') x from dual
/
EOF`;

my $rc = $?;

if ($rc) {
 print "sqlplus failed - exiting";
exit $?;
}

my $flash_time;

foreach ( split (/\n/, $sp_flash_time) ) {
chomp;
if ( /^row:/ ) {
 s/^[row://g](row://g);
 $flash_time = $_;
}
}

if ( !defined ($flash_time) ) {
 print "Unable to get database time - exiting";
}

print "Using flash time of: " . $flash_time . "\n";
open(PARFILE, ">", $PFILE);
print PARFILE "flashback_time=\"to_timestamp('${flash_time}', 'dd/mon/yyyy hh24:mi:ss')\"\n";
print PARFILE "parallel=1\n";
print PARFILE "exclude=statistics\n";
close(PARFILE);
$SIG{CHLD} = 'IGNORE';

my $shard = 0;
my $cmd;

foreach ($shard = 0 ; $shard < $PARALLEL ; $shard++) {
 $cmd = "expdp \\\'/ as sysdba\\\' tables=${OWNER}.${TABLE} dumpfile=DMP_DIR:SECQUOTE2_${shard}.dmp logfile=LOG_DIR:SECQUOTE2_${shard}.log JOB_NAME=:SECQUOTE2_${shard} COMPRESSION=ALL CONTENT=DATA_ONLY"; 
 $cmd .= " parfile=${PFILE}";
 $cmd .= " query=${OWNER}.${TABLE}:\'\"where security_quote_id >= 500000000 and  mod(dbms_rowid.rowid_block_number(rowid), ${PARALLEL}) = " . $shard . "\"\'"; 
 $cmd .= " &";
 print "Starting: $cmd\n";
 my $cpid = system($cmd);
}
```

LOB-Tabelleninhalt mit mehreren Export-Jobs parallelisiert exportieren (gleichzeitig zum Full-Export), Bsp:

```bash
chmod u+x exp_LOBTAB.pl
./exp_LOBTAB.pl
```

Import (neue DB)

unter Umständen füllen die Tuningtasks den TEMP TBS, desshalb vorher disablen

```bash
sqlplus "/ as sysdba" << EOSQL
 BEGIN dbms_auto_task_admin.disable( client_name => 'sql tuning advisor', operation => NULL, window_name => NULL); END;
/
EOSQL
```

User-Import des Full-Exports starten, ohne Indizes und Constraints

```bash
echo "EXCLUDE=TABLE_STATISTICS" > imp_wo_stats.par
echo "EXCLUDE=INDEX_STATISTICS" >> imp_wo_stats.par
impdp_user_db.sh -o <USERS> -P imp_wo_stats.par -N -C -X -p 10     # falls Trigger vorhanden: -T
```

LOB-Tabellen-Inhalt mit gleichzeitig laufenden Import-Jobs importieren

Sobald die LOB-Tabellen ohne Inhalt importiert sind, den LOB-Import starten, Bsp:

```bash
. . imported "RSEDBA"."COM_CONTAINER"                    4.976 KB       0 rows
```

```bash
cd <Export-Dump-Pfad>
ll COM_CONTAINER*.dmp   # LOB-Filenamen anhand Script

for impfile in COM_CONTAINER*.dmp ; do
  impfile=${impfile##/*/}
  job=${impfile%.dmp}
  echo "---------- importing file ${impfile}"
  impdp USERID=\"/ as sysdba\" DUMPFILE=DMP_DIR:${impfile} LOGFILE=LOG_DIR:${ORACLE_SID}_$(date '+%Y%m%d')_impdp_${job}.log FULL=Y JOB_NAME=${job} PARALLEL=1 TABLE_EXISTS_ACTION=APPEND METRICS=Y DATA_OPTIONS=DISABLE_APPEND_HINT  &
  sleep 5
done
```

Constraints und Indizes nachimportieren

Sobald alle Daten importiert sind, können nun alle Constraints, Indizes und ev. Trigger nachimportiert werden

Bsp:

```bash
echo "INCLUDE=CONSTRAINT"         > imp_index_cons.par
echo "INCLUDE=REF_CONSTRAINT"    >> imp_index_cons.par
echo "INCLUDE=INDEX"             >> imp_index_cons.par
```

falls Trigger:

```bash
echo "INCLUDE=TRIGGER"             >> imp_index_cons.par
impdp_user_db.sh -o <USER> -P imp_index_cons.par -p 8
```

Im Anschluss müssen die Objekt-Statistiken neu berechnet werden, Bsp. mit 8 parallelen Prozessen:

```sql
sqlplus "/ as sysdba" << EOSQL
begin
dbms_stats.gather_schema_stats(
     ownname          => 'G4ZKB',
     estimate_percent => dbms_stats.auto_sample_size,
     cascade          => true,
     force          => true,
     degree           => 8
   );
end;
/
EOSQL
```

Tuningtasks wieder enablen

```sql
sqlplus "/ as sysdba" << EOSQL
BEGIN dbms_auto_task_admin.enable( client_name => 'sql tuning advisor', operation => NULL, window_name => NULL); END;
/
EOSQL
```


### Tags:

[[HowTo]] - [[DataPump]] - [[OracleLobs]]

