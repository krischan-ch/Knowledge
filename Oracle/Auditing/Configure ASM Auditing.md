# Abstract

Diese Seite beschreibt die Funktionsweise des ASM Auditing auf Exadata und AIX DB-Server (nur mit installierter Grid Infrastructure).

# Konfiguration
## ASM Instanz Parameter

In der ASM Instanz werden die folgenden Parameter gesetzt:

```sql
alter system set audit_sys_operations=TRUE scope=spfile;
alter system set audit_syslog_level='local0.info' scope=spfile;
```

## Konfiguration SYSLOG-Facility
Der Wert für AUDIT_SYSLOG_LEVEL stellt eine Logging Facility im SYSLOG dar. Diese Facility muss im Konfigurationsfile des SYSLOG-Daemons konfiguriert werden.

**Beispiele:**

### Exadata & AIX

```bash
mkdir /var/local/oracle/asm/audsyslog

# cat /etc/syslog-ng.d/20_oracle_asm_audit.conf
destination d_oracle_asm_audit    { file("/var/local/oracle/asm/audsyslog/asmaudit.log"         template(t_syslog)); };

filter f_oracle_asm_audit { facility(local0) and level(info); };

log { source(s_local); filter(f_oracle_asm_audit); destination(d_oracle_asm_audit); flags(final); };
```

Diese Settings definieren den entsprechenden Pfad als Ziel für die SYSLOG-Einträge der ASM-Instanz.
Damit die neue Konfiguration aktiv wird muss der Daemon neu gestartet werden:

| OS | Syntax |
| --- | --- |
| AIX | /etc/rc.syslog-ng restart |
| Exadata | service syslog-ng restart |

Bitte prüfen ob das Verzeichnis für das audit-File auch existiert! 

## Housekeeping
Das Housekeeping dieser Dateien wird per LOGROTATE Daemon realisiert. Der Logrotate ist wie folgt definiert:

**Exadata**
	
```bash
[root@exa301s01 logrotate.d]# cat /etc/logrotate.d/asmaudit
/var/log/asmaudit.log {
weekly
rotate 4
compress
copytruncate
delaycompress
notifempty
}

AIX
# cat /etc/logrotate.d/asmaudit
/var/local/oracle/asm/audsyslog/asmaudit.log {
  weekly
  rotate 4
  compress
  copytruncate
  delaycompress
  notifempty
} 
```

Es wird jede Woche ein neues File erstellt. Die bestehenden Dateien werden rotiert und es werden vier Generationen aufbewahrt.

# Ansible-Playbook

Das Ansible-Playbook configure-asm-audit ist im Sharepoint Wiki separat dokumentiert.


Tags:
[[ASM]] - [[Auditing]] - [[SYSLOG]] - [[Exadata]] - [[HowTo]]

