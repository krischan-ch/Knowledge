
# dbms_metadata - Role DDL generieren

## DDL für Rolle extrahieren

Um das DDL einer Rolle zu extrahieren sind vier Befehle notwendig.
Zuerst wird das DDL für die Rolle selbst generiert. Im Anschluss die Role-, die System und als letztes die Objekt-Grants.

### Die Rolle selbst

```sql
SQL> col ddl format a100
SQL> select dbms_metadata.get_ddl('ROLE','DBA') DDL from dual;

DDL
----------------------------------------------------------------------------------------------------
   CREATE ROLE "DBA"
```

### Role Grants

```sql
SQL> select dbms_metadata.get_granted_ddl('ROLE_GRANT','DBA') DDL from dual;
DDL
----------------------------------------------------------------------------------------------------
   GRANT "SELECT_CATALOG_ROLE" TO "DBA" WITH ADMIN OPTION
   GRANT "EXECUTE_CATALOG_ROLE" TO "DBA" WITH ADMIN OPTION
   GRANT "DELETE_CATALOG_ROLE" TO "DBA" WITH ADMIN OPTION
   GRANT "EXP_FULL_DATABASE" TO "DBA"
   GRANT "IMP_FULL_DATABASE" TO "DBA"
   GRANT "DATAPUMP_EXP_FULL_DATABASE" TO "DBA"
   GRANT "DATAPUMP_IMP_FULL_DATABASE" TO "DBA"
   GRANT "GATHER_SYSTEM_STATISTICS" TO "DBA"
   GRANT "SCHEDULER_ADMIN" TO "DBA" WITH ADMIN OPTION
   GRANT "PLUSTRACE" TO "DBA" WITH ADMIN OPTION
   GRANT "TKPROFER" TO "DBA" WITH ADMIN OPTION
```
### System Grants
```sql
SQL> select dbms_metadata.get_granted_ddl('SYSTEM_GRANT','DBA') DDL from dual;
DDL
----------------------------------------------------------------------------------------------------
  GRANT FLASHBACK ARCHIVE ADMINISTER TO "DBA" WITH ADMIN OPTION
  GRANT ADMINISTER SQL MANAGEMENT OBJECT TO "DBA" WITH ADMIN OPTION
  GRANT UPDATE ANY CUBE DIMENSION TO "DBA" WITH ADMIN OPTION
  GRANT UPDATE ANY CUBE BUILD PROCESS TO "DBA" WITH ADMIN OPTION
  GRANT DROP ANY CUBE BUILD PROCESS TO "DBA" WITH ADMIN OPTION
  GRANT CREATE ANY CUBE BUILD PROCESS TO "DBA" WITH ADMIN OPTION
  GRANT CREATE CUBE BUILD PROCESS TO "DBA" WITH ADMIN OPTION
…
```
### Object Grants
```sql
SQL> select dbms_metadata.get_granted_ddl('OBJECT_GRANT','DBA') DDL from dual;
DDL
----------------------------------------------------------------------------------------------------
  GRANT ALTER ON "SYS"."AWSEQ$" TO "DBA"
  GRANT ALTER ON "SYS"."MAP_OBJECT" TO "DBA"
  GRANT DELETE ON "SYS"."MAP_OBJECT" TO "DBA"
  GRANT INSERT ON "SYS"."MAP_OBJECT" TO "DBA"
  GRANT SELECT ON "SYS"."V_$DIAG_VTEST_EXISTS" TO "DBA"
  GRANT SELECT ON "SYS"."V_$DIAG_VNOT_EXIST_INCIDENT" TO "DBA"
  GRANT SELECT ON "SYS"."V_$DIAG_VIPS_PKG_INC_CAND" TO "DBA"
  GRANT SELECT ON "SYS"."V_$DIAG_V_SWPERRCOUNT" TO "DBA"
  GRANT SELECT ON "SYS"."V_$DIAG_V_ACTPROB" TO "DBA"
  GRANT SELECT ON "SYS"."V_$DIAG_V_ACTINC" TO "DBA"
  GRANT SELECT ON "SYS"."V_$DIAG_VIPS_PKG_MAIN_PROBLEM" TO "DBA"
  GRANT SELECT ON "SYS"."V_$DIAG_VIPS_PACKAGE_MAIN_INT" TO "DBA"
…
```
### Fehlermeldungen

Existiert für einen Objekttyp kein Eintrag (z. Bsp. kein System-Grant) wird folgender Fehler ausgegeben:
```sql
SQL> ERROR:
ORA-31608: Angegebenes Objekt vom Typ SYSTEM_GRANT nicht gefunden
ORA-06512: in "SYS.DBMS_METADATA", Zeile 5805
ORA-06512: in "SYS.DBMS_METADATA", Zeile 8492
ORA-06512: in Zeile 1

Tags:  
[[DDL]] - [[dbms_metadata]] - [[HowTo]]
```


# Tags:

[[HowTo]] - [[SQLPLUS]]