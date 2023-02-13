# Allgemeines

Das zkb_ Wrapper Script master.zkb_wrap setzt bei Multi-Home-/Multi-DB-Environments das richtige Environment für die entsprechende Datenbank und ruft das mitangegebene Script oder Executable auf. Es stellt sicher, dass die Oracle Environment Variablen ORACLE_SID, ORACLE_HOME, PATH, etc. für die angegebene Instanz und die entsprechende Umgebung (mittels Basenv) gesetzt sind und damit das angegebene Script aufgerufen wird.

Wenn das DB-Environment schon vorher gesetzt ist, muss das Wrapper-Script für eine einzelnde DB nicht verwendet werden. Bei mehreren Instanzen auf einem Server erleichtert es die Handhabung (ohne vorher das Environment zu setzen) oder auch beim Ausführen von Scripts auf mehreren Instanzen eines Servers.

# Funktionsweise

Die meisten in der ZKB entwickelten Oracle Shell- und Perl-Scripts (oder auch Oracle Binaries wie sqlplus, sqlldr, rman etc.) sind mit diesem Wrapper-Script verlinkt, über welches das eigentliche Script (evtl. mit eigenen Parametern mittels "-arg <par>" oder "--<par>") aufgerufen wird, Bsp:

```bash
zkb_chpwd.sh -s TEST11 -arg -c
```

*/usr/local/db/bin/zkb_chpwd.sh* ist ein symbolischer Link auf */usr/local/db/data/master.zkb_wrap*

*master.zkb_wrap* setzt das Environment für die DB TEST11 anhand ORATAB und ruft anschliessend das eigentliche Script ohne "zkb_" auf, d.h. chpwd.sh, mit dem Parameter "-c" auf.

Die Liste der mit dem zkb_-Wrapper verlinkten Scripts wird im File */usr/local/db/data/zkb_wrap.dat* definiert und das Verlinken selbst auf dem DB-Server übernimmt das Script *cr_wrap_ln.sh*.

# Parameter

Die wichtigsten Parameter von *master.zkb_wrap*:

-s <SID> [,<SID2>...]

    Script auf angegebener SID ausführen

    bei Angabe mehrerer SID's (kommagetrennt) wird das entsprechende Script für jede SID separat ausgeführt

-s all

    es werden alle in ORATAB auf Y gesetzten SID's abgearbeitet

-e

   alle mit -s angegebenen SIDs sind extern (remote via TWO_TASK)

-E <ENV>

   alle mit -s angegebenen SIDs sind extern (remote via TWO_TASK), mit Environment <ENV> (Bsp -E dummy11204)

-p

   bei Angabe mehrerer SID's wird das Script nicht sequenziell sondern möglichst parallelisiert aufgerufen (abhängig von der CPU-Last)

-t

    das Startflag Y in ORATAB wird für die entsprechende SID ignoriert

-T

    unset TWO_TASK (nur fuer profile.zkb)

-m

   bei nicht erfolgreicher Beendigung des aufgerufenen Scripts (RC<>0) wird ein Mail an occ@zkb.ch versendet

-r

  bei nicht erfolgreicher Beendigung des aufgerufenen Scripts (RC<>0) wird ein SNMP Trap generiert

Für alle Parameter siehe z.B. "zkb_chpwd.sh -help"

# Variante selsess

Das Script selsess resp. zkb_selsess ist auch einre Art Wrapper-Script, welches ein SQL-Script resp. die angegebenen Select-SQL-Scripts auf der Datenbank via SQL-Plus ausführt.

Standardmässig wird das SQL-Script selsess.sql ausgeführt. Mittels Parameter -S <script1>[,<script2>...] werden alle angegebenen SQL-Scripts (mit "sel" anfangend) hineterinander ausgeführt.

Für alle Parameter siehe"selsess -h"

# Beispiele

**Starten einer Datenbank:**

zkb_dbstart.zkb -s TEST81

**Neustarten von zwei Datenbanken, inkl. Listener und Blackout:**

zkb_dbrestart.zkb -s TEST81,TEST82 -arg -Bin

**paralleles Herunterfahren aller Datenbanken auf einem DB-Server:**

zkb_dbshut.zkb -s all -p -arg -Birn

**Ausführen von dbharden.sh auf allen Datenbanken auf einem DB-Server:**

zkb_dbharden.sh -s all

**Aufruf von mehreren Select-Scripts auf mehreren Datenbanken:**

zkb_selsess -s TEST81,TEST82 -arg -S seldb,selreg,seldg -i

**Aufruf von SQL-Plus mit einem SQL-Script auf einer lokalen DB:**

zkb_sqlplus -s TEST82 -arg scott/tiger @update_xy.sql

**Aufruf von SQL-Plus mit einem SQL-Script auf einer remote DB (sie muss über SQL-Net auflösbar sein):**

zkb_sqlplus -s TEST83 -e -arg scott/tiger @update_xy.sql

# Tags:

[[HowTo]] - [[Skripte]]
