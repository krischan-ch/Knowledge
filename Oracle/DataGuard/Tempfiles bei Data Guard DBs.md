
# ​Fakt

Anpassungen (Ändern der maximalen File-Grösse, File-Vergrösserung oder Hinzufügen von Files) an TEMPFILES auf einer Data Guard Primary DB werden nicht auf die Standby DB repliziert, d.h. die Standby DB hat dann immer noch die alten TEMPFILE Definitionen und beim nächsten Switchover oder Failover ist dort das TEMP Tablespace evtl. zu klein dimensioniert!

Siehe auch Oracle Note "Data Guard Physical Standby - Managing temporary tablespace tempfiles (DOC ID ​[1514488.1](https://[support.oracle.com/epmos/faces/DocContentDisplay?_afrLoop=99529780274179&id=1514588.1](http://support.oracle.com/epmos/faces/DocContentDisplay?_afrLoop=99529780274179&id=1514588.1)))"

D.h. alle Tempfile-Anpassungen auf der Primary DB müssen auf der Standby DB manuell nachgeführt werden!

**Beispiel**

Wenn ein z.B. neues temporäres Tablespace auf der Primary DB erstellt wird und nicht auf der Standby nachgeführt wird, ist das Tablespace nach einem Switchover auf der neuen Primary leer (ohne Tempfiles) -> Fehlermeldung beim Zugriff:

ORA-00604: error occurred at recursive SQL level 1

ORA-25153: Temporary Tablespace is Empty​​

# Lösung

Die gleichen TEMPFILE Anpassungen auf der Primary müssen auch auf der Standby DB gemacht werden. Damit dies möglich ist, muss die Standby DB zumindest read-only aufgemacht werden.

## Variante 1 (ohne Appl.-Unterbruch, ohne Switchover):

**Primary DB**: TEMPFILE Anpassungen auf der Primary durchführen, Bsp:

```bash
PRIM-SQL> @seltempf
PRIM-SQL> alter database tempfile 1 autoextend on next 256m maxsize 16g;
PRIM-SQL> @seltempf
```

​**​​Standby DB**: Status checken

```bash
STBY-SQL> @seldg
```

Standby DB: Log-Apply stoppen, falls Active Data Guard nicht aktiv ist (Active DG: READ ONLY WITH APPLY)

```bash
dgcmd.sh -l

Standby DB: wenn OPEN_MODE = MOUNTED, Standby DB read-only öffnen
```
```sql
STBY-SQL> alter database open;
```

Bei Fehler (ORA-10458: standby database requires recovery oder ORA-10456: cannot open standby database; media recovery session may be in progress), muss die Standby DB durchgestartet werden, Bsp mit Clusterware:

```bash
srvctl stop database -d ${BE_ORA_DB_UNIQUE_NAME}
srvctl start database -d ${BE_ORA_DB_UNIQUE_NAME} -o "read only"

STBY-SQL> @seldg
```

-> OPEN_MODE = READ ONLY oder READ ONLY WITH APPLY

Standby DB: gleiche TEMPFILE Anpassungen wie auf der Primary durchführen, Bsp:

```sql
STBY-SQL> @seltempf
STBY-SQL> alter database tempfile 1 autoextend on next 256m maxsize 16g;
STBY-SQL> @seltempf
```

6. Standby DB: zurück in den Mount-Status bringen, d.h. Standby DB durchstarten

```bash
dbrestart.zkb -Bn
```

resp.

```bash
srvctl stop database -d ${BE_ORA_DB_UNIQUE_NAME} ; srvctl start database -d ${BE_ORA_DB_UNIQUE_NAME}
```

Standby DB: Log-Apply wieder starten, falls Active Data Guard nicht schon aktiv ist

```bash
dgcmd.sh -L
```

Standby DB: Status checken

```bash
seldg
```

-> OPEN_MODE = MOUNTED oder READ ONLY WITH APPLY​

## Variante 2 (mit Appl.-Unterbruch wegen 2 x Switchover):

TEMPFILE Anpassung auf Primary DB

```sql
PRIM-SQL> alter tablespace TEMP add tempfile ...
PRIM-SQL> @seltempf
```

Switchover zur Standby

```bash
dgcmd.sh -X
```

TEMPFILE Anpassung auf Ex-Standby DB (nun Primary DB)

```sql
PRIM-SQL> alter tablespace TEMP add tempfile ...
PRIM-SQL> @seltempf
```

Switchover zurück

```bash
dgcmd.sh -X
```

Tags:
[[HowTo]] - [[DataGuard]] - [[TempFile]]

