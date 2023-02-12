
Achtung da SQLNET.FALLBACK_AUTHENTICATION=true auf true gesetzt wird werden viele Fehler als ORA-01017 invalid username / password dargestellt.

Um das zu beheben muss getraced werden oder der Parameter temporär auf false zu setzen.

# Konfigurationsfehler

Um reibungslos zu funktionieren müssen in /app/oracle/admin bei Dataguard die Verzeichnisse korrekt verlinkt sein. Alle Dateien müssen im Admin Verzeichnis /dbms/oracle/<dbanme>/ sein. der Uniqname wird lediglich auf das adm Filesystem verlinkt

**Richtig:**

```bash
> ll
total 0
lrwxrwxrwx    1 oracle   dba              22 May 02 11:05 CLS02 -> /dbms/oracle/CLS02/adm
lrwxrwxrwx    1 oracle   dba              22 May 03 06:56 CLS02_A_IT -> /dbms/oracle/CLS02/adm
```

**Falsch:**

```bash
lrwxrwxrwx    1 oracle   dba              22 Mar 20 17:52 PZM03 -> /dbms/oracle/PZM03/adm
drwxr-xr-x    4 oracle   dba             256 Mar 21 16:55 PZM03_A_IT
```

**Exadata Login nicht möglich**

Wenn das Login Recht im IAM Vergeben ist und trotzdem kein Login möglich ist, fehlt vermutlich das Homeverezichnis des Users.

Darin muss eine File *.k5login* existieren.

Fehlende Homeverzeichnisse können mit folgendem Befehl angelegt werden:

```bash
# erstellt das Homedir
asroot /usr/app/zkb-centrify/bin/krb5_homedir.pl 

# akualisiert die Rechte
adquery user -a tx00
```

# Fehlermeldungen

### ORA-01017: Benutzername/Kennwort ungültig; Anmeldung abgelehnt.  /  "kein OAV Zugriff moeglich" (bei auth.pl -list)

  - Mit "kerberize.sh -a" auf der DB prüfen ob das sqlnet.ora korrekt ist. Wenn nicht, mittels Script bereinigen (kerberize.sh -A)

  - Im sqlplus mit "show parameter ldap_directory_access" prüfen ob der Wert PASSWORD ist. Falls nicht mit "alter system set ldap_directory_access=PASSWORD scope=both;" den Wert setzen.

> Achtung: Der Parameter muss auf allen beteiligten Instanzen (Primary+Standby) gesetzt werden!

- Prüfen, ob der DB-User SHARED_OID_USER existiert
  - wenn nicht: mit "auth.pl -crshuser" erstellen und "auth.pl -refresh"​ laufen lassen.

### Spezialfall (1x aufgetreten)

- Fehlerbild unklar (1017 beim Login, sowohl via LDAP/Kerberos als auch via LDAP/Passwort).

- Tracing aktivieren:
  - SQL> alter system set events '28033 trace name context forever, level 9';

  - Im Tracefile ganz normaler Ablauf drin, endet jedoch mitten drin mit folgender Linie:

```KZLD_ERR: failed the search 28304.
KZLD_ERR: 28304

kzld_search -s sub -b cn=OracleDefaultDomain,cn=OracleDBSecurity,cn=Products,cn=OracleContext,ou=orast,ou=zkbrollen,o=zkb,c=ch

search filter: (&(objectclass=orcldbSubtreelevelMapping)(orclDBDistinguishedName=ou=user,ou=orast,ou=zkbrollen,o=zkb,c=ch))

KZLD_ERR: 0

kzld found userschema SHARED_OID_USER

KZLD is doing LDAP unbind

<<< EOF >>>>
```

- Tracing wieder deaktivieren:

```sql
SQL> alter system set events '28033 trace name context off';
```

- Da dies auf  Probleme mit dem SHARED_OID_USER hindeutete, diesen mal neu als "globally identified" setzen.

VORHER BITTE noch folgende Query absetzen für VORHER/NACHER vergleich

```sql
SQL> select * from SYS.user$ where name = 'SHARED_OID_USER';
SQL> alter user SHARED_OID_USER identified globally;
```

Damit konnte das Problem gelöst werden.

>ORA-01045: Benutzer SHARED_OID_USER hat keine CREATE SESSION-Berechtigung; Anmeldung abgelehnt.

Der Benutzer hat keine Berechtigungen auf dieser Datenbank. Kontrollieren mit ```auth.pl -list```

Falls der Benutzer dort aufgelistet ist mit auth.pl -list -verbose prüfen ob die DB in der entsprechenden Rolle registriert ist

Eventuell müssten auch die Rechte(bez. Rollen) erst geladen werden. auth.pl -refresh.

```sql
ORA-01017: invalid username/password; logon denied
```

Evtl. ist der Parameter *LDAP DIRECTORY_ACCESS* nicht auf Password gesetzt

```sql
alter system set ldap_directory_access=password;
```

Im */etc/hosts* muss der Eintrag des Servers zuerst mit der fqdn Adresse drin stehen, da Oracle den ersen Eintrag zum Aufbau der Service Principles verwendet

also 10.48.147.18 exa302s51.eng.zkb.ch und nicht 10.48.147.18 exa302s51 

**Zitat:**

>When a user wants to authenticate to the database it will build in the ticket the database principal in the following way:
<VALUE_OF_AUTHENTICATION_KERBEROS5_SERVICE>/<first_entry_from_etc_hosts_file>@REALM_NAME which in your case translates to [orakerb/exa302s01@PROD.ZKB.CH](mailto:orakerb/exa302s01@PROD.ZKB.CH) and this corresponds to the following line from client trace:

> 18-JAN-2017 13:25:56:918] nauk5aegetservcred: Client is [t874@PROD.ZKB.CH](mailto:t874@PROD.ZKB.CH), Service provider is [orakerb/exa302s01@PROD.ZKB.CH](mailto:orakerb/exa302s01@PROD.ZKB.CH)

>The service provider constructed by the client will be matched with the service provider from the keytab and in case there is no mismatch the client fails to authenticate with 12638 and in client trace we see the following error:

>[18-JAN-2017 13:25:56:925] nauk5a0sendclientauth: Couldn't get service credentials.

> The error "Couldn't get service credentials" it ALWAYS means there is something wrong with the database server principal and we have to check if the keytab file is correctly matching the entry from /etc/hosts file.

> Desweiteren kann es sein das der SHARED_OID_USER kein globaler User ist (identified globally). Dies kann mit seluser geprüft werden. Hier muss als Passwort GLOBAL stehen. Wenn dies nicht gegeben ist muss der User gelöscht und neu angelegt werden:

```sql
sqlplus / as sysdba <<EOSQL
  drop user shared_oid_user cascade;
  create user shared_oid_user identified globally;
EOSQL

ORA-01017: Benutzername/Kennwort ungültig; Anmeldung abgelehnt.
```

### Ausgangslage

Verbindung auf DB über sqlplus und Kerberos User ab PCMZ schlägt mit ORA-1017 fehl. Alle Verbindungen zu irgendwelche DB's schlagen fehl.

Dieselbe Verbindungen funktionieren auf anderem PC / Laptop  (Redfox bspw.) mit demselben User

Das bedeutet:  
- Kerberos auf DB Seite ok
- User Berechtigungen ok

Naheliegendstes Problem in diesem Fall:  
- Konfiguration / Kerberos auf Client Seite falsch

Lösung:
- Falscher Client wird verwendet (bspw. xe statt instan client)

## Sicherstellen, dass iclient verwendet wird:
- dazu sqlplus vollqualifiziert aufrufen:  
```sqlc:\app\oracle\o64\iclient\sqlpus /@<DB>```

Kerberos Konfig nicht up-to-date

### Prüfen der Konfig in :
```sqlc:\Program Files\DataLink\Oracle\sqlnet.ora```

### Kerberos Einträge drin und korrekt?

```sql
ORA-01882 : timezone region not found bei Aufruf von eusm
```

Das ist ein Bug im EUSM. Workaround:
```bash
cd $ORACLE_HOME
cd bin

save.sh eusm

vi eusm

# Aufruf erstetze

$JRE_HOME/bin/java -classpath $JRE_HOME/lib/rt.jar:$JDBCLIBDIR/ojdbc6.jar:$RDBMSJLIBDIR/eusm.jar oracle.security.eus.util.ESMdriver "$@"

# durch

$JRE_HOME/bin/java -Duser.timezone=GMT -classpath $JRE_HOME/lib/rt.jar:$JDBCLIBDIR/ojdbc6.jar:$RDBMSJLIBDIR/eusm.jar oracle.security.eus.util.ESMdriver "$@"

ORA-12631: Username retrieval failed
```

Das kann an der keytab liegen. Also oracle mit *kerberize.sh -l* kontrollieren
Muss so aussehen mit allen Encrypten Methoden:

Als oracle:
```bash
kerberize.sh -l
*I* START listing Keytab entries
ktutil:  ktutil:  slot KVNO Principal

---- ---- ---------------------------------------------------------------------
   1    3  [orakerb/exa301s01.eng.zkb.ch@ENG.ZKB.CH](mailto:orakerb/exa301s01.eng.zkb.ch@ENG.ZKB.CH) (aes256-cts-hmac-sha1-96)
   2    3  [orakerb/exa301s01.eng.zkb.ch@ENG.ZKB.CH](mailto:orakerb/exa301s01.eng.zkb.ch@ENG.ZKB.CH) (aes128-cts-hmac-sha1-96)
   3    3  [orakerb/exa301s01.eng.zkb.ch@ENG.ZKB.CH](mailto:orakerb/exa301s01.eng.zkb.ch@ENG.ZKB.CH) (arcfour-hmac)
   4    3  [orakerb/exa301s01.eng.zkb.ch@ENG.ZKB.CH](mailto:orakerb/exa301s01.eng.zkb.ch@ENG.ZKB.CH) (des-cbc-md5)
   5    3  [orakerb/exa301s01.eng.zkb.ch@ENG.ZKB.CH](mailto:orakerb/exa301s01.eng.zkb.ch@ENG.ZKB.CH) (des-cbc-crc)
   6    3           [adminora_exa301s01$@ENG.ZKB.CH](mailto:adminora_exa301s01$@ENG.ZKB.CH) (aes256-cts-hmac-sha1-96)
   7    3           [adminora_exa301s01$@ENG.ZKB.CH](mailto:adminora_exa301s01$@ENG.ZKB.CH) (aes128-cts-hmac-sha1-96)
   8    3           [adminora_exa301s01$@ENG.ZKB.CH](mailto:adminora_exa301s01$@ENG.ZKB.CH) (arcfour-hmac)
   9    3           [adminora_exa301s01$@ENG.ZKB.CH](mailto:adminora_exa301s01$@ENG.ZKB.CH) (des-cbc-md5)
  10    3           [adminora_exa301s01$@ENG.ZKB.CH](mailto:adminora_exa301s01$@ENG.ZKB.CH) (des-cbc-crc)
```

**ktutil**

Falls Methoden fehlen muss ​die keytab neu erstellt werden.
Vorher Einträge in 2 Konfig Files prüfen:

```bash
grep   aes256 /etc/centrifydc/centrifydc.conf | grep -v '#'

adclient.krb5.tkt.encryption.types: aes256-cts aes128-cts arcfour-hmac-md5 des-cbc-md5 des-cbc-crc

adclient.krb5.permitted.encryption.types: aes256-cts aes128-cts arcfour-hmac-md5 des-cbc-md5 des-cbc-crc

adminora_exa301s01.krb5.tkt.encryption.types.orakerb/[exa301s01.eng.zkb.ch](http://exa301s01.eng.zkb.ch/): aes128-cts aes256-cts arcfour-hmac-md5 des-cbc-crc des-cbc-md5
```
```bash
grep   aes256 /app/oracle/etc/krb5/krb5.conf | grep -v '#'

default_tgs_enctypes = aes256-cts aes128-cts arcfour-hmac-md5 des-cbc-md5 des-cbc-crc

default_tkt_enctypes = aes256-cts aes128-cts arcfour-hmac-md5 des-cbc-md5 des-cbc-crc

permitted_enctypes = aes256-cts aes128-cts arcfour-hmac-md5 des-cbc-md5 des-cbc-crc
```

Diese 2 Linien anpassen (Fehlende Methoden ergänzen):

```bash
default_tgs_enctypes = aes256-cts aes128-cts arcfour-hmac-md5 des-cbc-md5 des-cbc-crc

default_tkt_enctypes = aes256-cts aes128-cts arcfour-hmac-md5 des-cbc-md5 des-cbc-crc
```
​
​Änderung in /etc/centrifydc/centrifydc.conf vornehmen (adminora_exa301s01.krb5.tkt Teil nicht machen das kommt automatish):

Centrify agent restarten damit /app/oracle/etc/krb5/krb5.conf richtig updated wird

```bash
/etc/rc.centrify-agent restart
```

Der Update kann einige Zeit dauern.

In Oracle Keytab prüfen mit:

```bash
grep   aes256 /app/oracle/etc/krb5/krb5.conf | grep -v '#'
```

Falls es zu lange dauert können die Entries manuell angepasst werden (ACHTUNG Später prüfen dass es nicht wieder überschrieben wurde)

Das muss drin stehen​:

```bash
default_tgs_enctypes = aes256-cts aes128-cts arcfour-hmac-md5 des-cbc-md5 des-cbc-crc

default_tkt_enctypes = aes256-cts aes128-cts arcfour-hmac-md5 des-cbc-md5 des-cbc-crc

permitted_enctypes = aes256-cts aes128-cts arcfour-hmac-md5 des-cbc-md5 des-cbc-crc
```

Wenn *krb5.conf* ok ist, dann kann die keytab neu erstellt werden:

```bash
# als root

cd  /app/oracle/etc/krb5
mv krb5.keytab  krb5.keytab.save

# neu erstellen keytab
aes_keytab.sh -f

# muss in etwas so aussehen
root@exa202s01 krb5]# aes_keytab.sh -f
 *I* Generate /app/oracle/etc/krb5/krb5.keytab with Force

Success: New Account: adminora_exa202s01
 *I* Generate /app/oracle/etc/krb5/krb5.keytab finished

ktutil:  ktutil:  slot KVNO Principal
---- ---- --------------------------------------------------------------------1   6    [orakerb/exa202s01.st.zkb.ch@ST.ZKB.CH]mailto:orakerb/exa202s01.st.zkb.ch@ST.ZKB.CH) (aes256-cts-hmac-sha1-96)
2    6    [orakerb/exa202s01.st.zkb.ch@ST.ZKB.CH](mailto:orakerb/exa202s01.st.zkb.ch@ST.ZKB.CH) (aes128-cts-hmac-sha1-96)
3    6    [orakerb/exa202s01.st.zkb.ch@ST.ZKB.CH](mailto:orakerb/exa202s01.st.zkb.ch@ST.ZKB.CH) (arcfour-hmac)
4    6    [orakerb/exa202s01.st.zkb.ch@ST.ZKB.CH](mailto:orakerb/exa202s01.st.zkb.ch@ST.ZKB.CH) (des-cbc-md5)
5    6    [orakerb/exa202s01.st.zkb.ch@ST.ZKB.CH](mailto:orakerb/exa202s01.st.zkb.ch@ST.ZKB.CH) (des-cbc-crc)
6    6            [adminora_exa202s01$@ST.ZKB.CH](mailto:adminora_exa202s01$@ST.ZKB.CH) (aes256-cts-hmac-sha1-96)
7    6            [adminora_exa202s01$@ST.ZKB.CH](mailto:adminora_exa202s01$@ST.ZKB.CH) (aes128-cts-hmac-sha1-96)
8    6            [adminora_exa202s01$@ST.ZKB.CH](mailto:adminora_exa202s01$@ST.ZKB.CH) (arcfour-hmac)
9    6            [adminora_exa202s01$@ST.ZKB.CH](mailto:adminora_exa202s01$@ST.ZKB.CH) (des-cbc-md5)
10    6            [adminora_exa202s01$@ST.ZKB.CH](mailto:adminora_exa202s01$@ST.ZKB.CH) (des-cbc-crc)
```

```bash
ktutil:  [root@exa202s01 krb5]# aes_keytab.sh -f ^C
```

> ACHTUNG bei RAC muss das auf beiden Nodes geprüft werden

Vor dem Testen im DOS muss mit ```bashklist purge``` der Credential Cache geleert werden

```dos
u:\> klist purge

Aktuelle Anmelde-ID ist 0:0x166d1b
        Alle Tickets werden gelöscht:
        Ticket(s) gelöscht.

ORA-12638: Credential retrieval failed
```

Client überprufen ob er das File *C:\Program Files\rfTools\Install\W_TNS* und *SQLini Sync 1.0.002.pkg* besitzt! Wenn nicht soll der User ein Ticket aufmachen.

Das Kerberos System konnte nicht erfolgreich geladen werden. Falschkonfiguration auf der Client oder Server Seite.

**Serverseite:**

Als oracle ```kerberize.sh -a``` im Environment der DB laufen lassen und eventuelle Fehler korrigieren. (prüfen ob db auch mit diesem TNS_ADMIN var läuft

Keytab prüfen mit ```kerberize.sh -t``` im Environment der DB

*sqlnet.ora* der DB muss als user *grid* lesbar sein (bei Exadata)

Wenn das nicht hilft, dann bei der Fehlermeldung *ORA-12631* schauen, es könnte einen Server Keytab Issue sein.

Bei Dataguard kann auch untenstehender Grund vorkommen:

[[ORA-609 ORA-12638 Credential retrieval failed]]

> ORA-28030: Server ist beim Zugriff auf LDAP-Verzeichnisdienst auf Probleme gestossen

Die Verbindung zum LDAP ist fehlgeschlagen. Entweder ist ldap.ora falsch oder das Wallet. auth.pl -list zur Diagnose

Falls *auth.pl* fehlerlos läuft kann es sein dass auf SK DB's das Wallet an der Falschen Lokation gesucht wird.

Als Lösung kann folgender Eintrag in *sqlnet.ora* erstellt werden (Auf beiden DG-Seiten)

```bash
WALLET_LOCATION= (SOURCE= (METHOD=file) (METHOD_DATA= (DIRECTORY =  /app/oracle/admin/<DBNAME>/wallet )))
```
Danach sollte er das Wallet finden Mit *ls -al* bitte prüfen.

Kann gut sein dass z.B **/app/oracle/admin/MLDS01_A_IT* kein Link auf */dbms/oracle/MLDS01/adm* ist.

Bei EXA oder AIX unter srvctl Kontrolle, muss mit *svrctl getenv database -db <DB_UNIQUE_NAME>* geprüft werden ob *TNS_ADMIN* richtig gesetzt ist

**Trace des LDAP Zugriffs**

```sql
alter system set events '28033 trace name context forever, level 9';
```

**Deaktivieren Trace**

```sql
alter system set events '28033 trace name context off';
```
​
#### ​​ORA-28293: No matched Kerberos Principal found in any user entry.

der entsprechende Benutzer hat auf keiner Datenbank in dieser Umgebung ein Recht und ist somit im ora.... Tree im LDAP als Benutzer nicht hinterlegt.

#### ORA-28043: invalid bind credentials for DB-OID connection

Meistens ein Wallet Problem, zuerst Mal mit einem Reset probieren:

```bash
auth.pl -resetwal
```

#### ORA-NICHTS: Wenn sonst nichts hilft

Wenn die Registrierung ok ist, *auth.pl -list -verbose* zeigt alles i.o, dann besteht die Wahrscheinlichkeit, dass es sich um einen Zwischenzustand handelt. Prüfen ob nicht ORA-28030 im Alert.log ersichtlich sind. ​Die DB restarten hilft dann.

Meistens geschieht das im Zusammenhang mit dem Setzen von *TNS_ADMIN* (vorallem auf AIX).

#### [FATAL] [DBT-16001] Database configuration operation cannot be performed.

Unter 12.2 gibt es einen Bug der diesen Fehler hervorruft. Dieser Bug ist gefixt mit Patch  26980745

**Kontrolle:**

```bash
> opatch lsinventory | grep 26980745

Patch  26980745     : applied on Wed Jul 11 11:12:34 CEST 2018
     26980745
```

#### Wallets

Wallets werden verwendet um das Password und den Namen des Benutzers zu speichert der verwendet wird um auf das LDAP zuzugreifen

Bei *auth.pl -list* wird der Benutzer angezeigt (Beispiel unten) Wenn erfolgreich verifizert angezeigt wird dann ist der Benutzer und das Passwort gültig.

```bash
INFO: Verwende SRV:[lnx3631.eng.zkb.ch](http://lnx3631.eng.zkb.ch/) Port 2389 Sport 2636 Context ou=oraeng,ou=zkbRollen,o=zkb,c=ch

INFO: cn=eusadm,ou=user,ou=oraeng,ou=zkbrollen,o=zkb,c=ch, password=>########## [lnx3631.eng.zkb.ch:2389](http://lnx3631.eng.zkb.ch:2389/)

INFO: DN:cn=TEST84,cn=OracleContext,ou=oraeng,ou=zkbrollen,o=zkb,c=ch erfolgreich verifiziert
```

Bei Dataguard sind die Wallets unter */app/oracle/admin/DB_UNIQE_NAME/wallet* gespeichert. Bei allen anderen unter */app/oracle/admin/DBNAME/wallet*.   Falls */app/oracle/admin/DB_UNIQE_NAME/wallet* zur Zeit der Installation noch nicht existiert wird von *auth.pl -install* ein Link auf von */app/oracle/admin/DB_UNIQE_NAME/wallet* auf */app/oracle/admin/DBNAME/wallet* erstellt und das Wallet dort gespeichert.

Falls das Passwort verloren ging kann mit auht.pl  -resetwallet versucht werden ein neues Passwort zu generieren

```bash
auth.pl

ERROR: EUSException: Database is not a member of this domain
```

Bei 11 DB's wird der DB Uniquename zur Registrierung verwendet. umd den Namen der Registrierung rauszufinden muss der Parameter -verbose angehängt werden. 

Also zum Beispiel: ```auth.pl -refresh -verbose``` anstelle von ```auth.pl -refresh​

ERROR: Registrierung fehlgeschlagen
```

prüfen, dass die Passwörter für sys und system korrekt hinterlegt sind

```bash
ERROR: Keine oder zuviele Hinterlegte Wallet Passwoerter
```

Das Wallet Verzeichnis löschen/renamen und erneut versuchen. Das Wallet Verzeichnis befindet sich darunter */app/oracle/admin/<DB_UNIQE_NAME>/wallet* oder **/app/oracle/admin/<DB_NAME>/wallet*.

eusm hängt oder sehr langsam Timeouts oder Broken Pipe

Falls das passiert ist es sehr wahrscheinlich dass LDAP so eingestellt ist, dass zusätzlich auch das Changelog zursucht wird. Dann muss für den ganzen Server das Changelog deaktiviert werden.

#### TNS_ADMIN

Das *TNS_ADMIN* Verzeichnis muss auf */dbms/oracle/<SID>/pfile* zeigen.

Mit untenstehendem Befehl kann als Benutzer "oracle" mit gesourcter DB <SID> unter AIX und LINUX die Variable kontrolliert werden welche zur Startzeit der DB verwendet wurde:

```bash
ps ewww `ps -ef|grep pmon | grep -v grep| grep $ORACLE_SID | awk -F" " '{print $2}'` | tr ' ' '\n' | grep = | sort | grep TNS_ADMIN
```

Auf EXA als root mit gesetztem basenv und Instanz

```bash
export pid=$(ps -ef | grep -i $ORACLE_SID | grep pmon | awk -F" " '{print $2}');xargs --null --max-args=1 echo </proc/$pid/environ | grep TNS_ADMIN
```
```sql
/proc/$pid/environ | grep TNS_ADMIN
Um TNS_ADMIN zu setzen muss man folgenden srvctl command ausführen & anschliessend die DB bouncen--> srvctl setenv database -db <DB_UNIQUE_NAME> -env "TNS_ADMIN=/app/oracle/admin/<ORACLE_SID>/pfile"​
```
#### Zugang testen auf PCMZ

Auf einem PCMZ eine DOS-Command Box öffnen und ein DB Login via SqlPlus und Kerberos-Authentifizierung vornehmen:

```dos
CMD> sqlplus [/@SICADP01.prod.zkb.ch](mailto:/@SICADP01.prod.zkb.ch)

SQL*Plus: Release 11.2.0.4.0 Production on Thu Mar 9 14:15:13
Copyright (c) 1982, 2013, Oracle.  All rights reserved.
Connected to:

Oracle Database 12c Enterprise Edition Release 12.1.0.2.0 - 6
With the Partitioning, OLAP, Advanced Analytics and Unified A

SQL> show user
USER is "SHARED_OID_USER"

-- Auflisten der zugewiesenen LDAP-Rollen:
SQL> select * from session_roles;
```

#### DBCA failed beim Registrieren der DB: *Could not find an appropriate listener*
Auf der DB show parameter listener eingeben

Der gleiche Eintrag mit der Adresse muss auch im listener.ora vorhanden sein [(Doc ID 2370412.1)](https://support.oracle.com/epmos/faces/DocContentDisplay?_afrLoop=84538837320670&id=2370412.1)]

##### ConnectString Beispiele

**ODBC**

statt wie bisher:

```
Provider=""OraOLEDB.Oracle"";Data Source="QC03.prod.zkb.ch";User Id="USERNAME";Password="PASSWORD";
```

müsste neu:

```
Provider=""OraOLEDB.Oracle"";Data Source="QC03.prod.zkb.ch";OSAuthent=1;
```

... benutzt werden.

#### DB Ldap CN: NODBDN oder DBT-08012

Wenn eine Datenbank nicht sauber im LDAP registriert werden kann, weil kein DBDN vorhanden ist, kann dies wie folgt manuell durchgeführt werden.

Beim Kerberos-DB-Login erscheint dabei der Fehler *ORA-28043: invalid bind credentials for DB-OID connection*

Zur Lösung ist ```auth.pl -resetwal``` zu verwenden. vorher ist zu überprüfen ob die admin Verzeichnisse korrekt verlinkt sind.

Folgendes macht *resetwal*

Beispiel-DB: FIMAS02

1. Bestehende Wallet-Daten entfernen (oracle):

```rm -rf /app/oracle/admin/FIMAS02/wallet/*```

2. Wallet & LDAP Passwort auslesen (oracle)

```
readpw -w FIMAS02_wallet -d /app/oracle/admin/FIMAS02/wallet -k /app/oracle/admin/FIMAS02/wallet/.id_wallet

readpw -l eusadm
```

3. Leeres Wallet erstellen (oracle)

```orapki wallet create -wallet /app/oracle/admin/FIMAS02/wallet -pwd <WALLET-Passwort> -auto_login```

4. Einträge für *ORACLE.SECURITY.PASSWORD* und *ORACLE.SECURITY.DN* im Wallet erfassen (oracle)

```bash
echo "<WALLET-Passwort>" | mkstore -wrl /app/oracle/admin/FIMAS02/wallet -createEntry ORACLE.SECURITY.PASSWORD <LDAP Password> -createEntry ORACLE.SECURITY.DN  "cn=FIMAS02,cn=OracleContext,ou=oraeng,ou=zkbrollen,o=zkb,c=ch"
```

Wenn der Fehler *No locks available* erscheint, ist der NFS Share ohne *nolock* Option gemounted. Dies kann wie folgt behoben werden (root):

```bash
# beim entsprechenden NFS Share die Option **nolock** hinzufügen
vi /etc/fstab 

# prüfen, welche Prozesse auf dem NFS Share aktiv sind
lsof /app/oracle/admin/FIMAS02 

kill -9 <Prozesse auf NFS>
umount -f /app/oracle/admin/FIMAS02
mount /app/oracle/admin/FIMAS02
mount | grep FIMAS02
```

5. Einträge im Wallet überprüfen (oracle)

```bash
echo "<WALLET-Passwort>" | mkstore -wrl /app/oracle/admin/FIMAS02/wallet -viewEntry ORACLE.SECURITY.PASSWORD -nologo -viewEntry ORACLE.SECURITY.DN
```

6. auth.pl Install durchführen (oracle)

```bash
auth.pl -install
```

Falls der Fehler [FATAL] [DBT-08012] A database with name (xxxxxx) is already registered with Directory service erscheint (oracle):

```bash
mkpw -w FIMAS02_wallet -d /app/oracle/admin/FIMAS02/wallet -k /app/oracle/admin/FIMAS02/wallet/.id_wallet

auth.pl -install
```

7. Prüfung (oracle)

```auth.pl -list```

DB im LDAP mauell deregistrieren weil sie sich nicht regisitrieren lässt

siehe [[OUD - manuelles löschen einer DB im LDAP]]



[Troubleshooting Guide von Oracle](https://support.oracle.com/epmos/faces/SearchDocDisplay?_afrLoop=364675662889984&_afrWindowMode=0&_adf.ctrl-state=11agywp5ap_4)


# Tags:

[[HowTo]] - [[LDAP]] - [[EUS]] - [[OUD]]