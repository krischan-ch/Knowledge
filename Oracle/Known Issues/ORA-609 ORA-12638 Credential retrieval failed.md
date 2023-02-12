
# Fehler

Im Alertlog ist tritt mindestens einer der untenstehenden Fehler regelmässig auf:

- TNS-12638: Credential retrieval failed
- opiodr aborting process unknown ospid (320687) as a result of ORA-609

# Analyse

Aus den Tracefiles kann lediglich folgende Information entnommen werden:

- opiino: Attach failed due to ORA-12638

Aufgrund des Fehlercodes sollte man das listener.log basierend auf den Timestamps der Errors im Alertlog analysieren:

```bash
31-OCT-2017 15:43:21 * (CONNECT_DATA=(SID=SLZ03G1)(CID=(PROGRAM=perl)(HOST=[exa102s01.prod.zkb.ch](http://exa102s01.prod.zkb.ch/))(USER=oracle))(CLIENT_REGISTRATION=exa102s01.prod.zkb.ch_12102170117)) * (ADDRESS=(PROTOCOL=tcp)(HOST=10.35.16.104)(PORT=63058)) * establish * SLZ03G1 * 0

31-OCT-2017 15:42:06 * (CONNECT_DATA=(SID=SLZ03G1)(CID=(PROGRAM=perl)(HOST=[exa102s01.prod.zkb.ch](http://exa102s01.prod.zkb.ch/))(USER=oracle))(CLIENT_REGISTRATION=exa102s01.prod.zkb.ch_12102170117)) * (ADDRESS=(PROTOCOL=tcp)(HOST=10.35.16.104)(PORT=62988)) * establish * SLZ03G1 * 0
```

# Lösung

PERL könnte auf den Agent hinweisen.

Also: Guckst du sqlnet.ora vom agent13c und ob dem Kerberos ist wie unten aufgeführt konfiguriert:

```bash
vi /app/oracle/network/admin/sqlnet_agent12.ora

#BEGIN_KRB-CONF
SQLNET.AUTHENTICATION_SERVICES=(BEQ)
#END_KRB-CONF
```

# Tags:

[[HowTo]] - [[Kerberos]]