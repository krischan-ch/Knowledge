
**Ziel**: Tabelleninhalte resp. Werte in einer Zeile statt in einer Spalte per SQL darstellen

**Methode**: mit der SQL Funktion LISTAGG lassen sich einzelne Werte in einer Zeile aggregieren (ab 11g)

Beispiel:

```sql
SQL> select ENAME from SCOTT.EMP where DEPTNO=20 order by ENAME;
ENAME
----------
ADAMS
FORD
JONES
SCOTT
SMITH

SQL> select listagg (ENAME, ',') within group (order by ENAME) "NAMES" from SCOTT.EMP where DEPTNO=20;

NAMES
--------------------------------------------------
ADAMS,FORD,JONES,SCOTT,SMITH
```

Tags:

[[HowTo]] - [[SQLPLUS]] - [[SQLFunctions]]
