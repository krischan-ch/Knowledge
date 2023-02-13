
# Tabellen prüfen

```sql
set lines 180 pages 50
col owner format a10

select '1',owner,count(1) from dba_tables where owner in ('X','K','JOB_EXEC','ZKB','SLB','ZKB','KOBL') group by owner
union all
select '2',owner,count(1) from [dba_tables@avaloq01.prod.zkb.ch](mailto:dba_tables@avaloq01.prod.zkb.ch) where owner in ('X','K','JOB_EXEC','ZKB','SLB','ZKB','KOBL') group by owner
order by 2,3,1;
```

# Constraints prüfen (via DB-Link)

```sql
set lines 180 pages 50
col owner format a10

select '1',owner,CONSTRAINT_TYPE,count(1) from dba_constraints where owner in ('X','K','JOB_EXEC','ZKB','SLB','ZKB','KOBL') group by owner,constraint_type
union all
select '2',owner,CONSTRAINT_TYPE,count(1) from [dba_constraints@avaloq01.prod.zkb.ch](mailto:dba_constraints@avaloq01.prod.zkb.ch) where owner in ('X','K','JOB_EXEC','ZKB','SLB','ZKB','KOBL') group by owner,constraint_type
order by 2,3,1;
```

## fehlende Constraints selektieren

(Constraint-Typ in where Klausel beachten)

```sql
set lines 180 pages 200
col owner format a30
col constraint_name format a50
col table_name format a30

select OWNER,CONSTRAINT_NAME,CONSTRAINT_TYPE,TABLE_NAME from [dba_constraints@avaloq01.prod.zkb.ch](mailto:dba_constraints@avaloq01.prod.zkb.ch) where owner in ('X','K','JOB_EXEC','ZKB','SLB','ZKB','KOBL') and constraint_type='P'
minus
select OWNER,CONSTRAINT_NAME,CONSTRAINT_TYPE,TABLE_NAME from dba_constraints where owner in ('X','K','JOB_EXEC','ZKB','SLB','ZKB','KOBL') and constraint_type='P';
```

# Indizes prüfen (via DB-Link)

## Differenz selektieren:

```sql
set lines 180 pages 200
col owner format a10
select '1',owner,index_type,count(1) from dba_indexes @avaloq01.prod.zkb.ch where owner in ('X','K','JOB_EXEC','ZKB','SLB','ZKB','KOBL') and index_type != 'LOB' group by owner,index_type
union all
select '2',owner,index_type,count(1) from dba_indexes where owner in ('X','K','JOB_EXEC','ZKB','SLB','ZKB','KOBL') and index_type != 'LOB' group by owner,index_type
order by 2,3,1;
```

## Fehlende Indizes selektieren:

```sql
set lines 180 pages 200
col owner format a30
col index_name format a50
col table_name format a30

select OWNER,INDEX_NAME,INDEX_TYPE,TABLE_NAME from [dba_indexes@avaloq01.prod.zkb.ch](mailto:dba_indexes@avaloq01.prod.zkb.ch) where owner in ('K') and index_type='NORMAL'
minus
select OWNER,INDEX_NAME,INDEX_TYPE,TABLE_NAME from dba_indexes where owner in ('K') and index_type='NORMAL';
```

# GLOBAL NAME prüfen


```sql
echo "select * from global_name;" | sqlplus -l / as sysdba
```

**Falls der nicht korrekt ist wie folgt konfigurieren:**

```bash
_GDBNAME=$(echo ${ORACLE_SID}.$(dnsdomainname) |tr '[:lower:]' '[:upper:]')

echo "alter database rename global_name to ${_GDBNAME};" | sqlplus / as sysdba
```

# Tags:

[[HowTo]] - [[Avaloq]]
