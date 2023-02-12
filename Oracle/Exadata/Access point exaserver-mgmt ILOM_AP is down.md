# Fehler

http://exa202s02-mgmt.st.zkb.ch/ Access point exa202s02-mgmt.st.zkb.ch_ILOM_AP2 is down

# Ursache

Dieser Fehler tritt auf, wenn der Access point, welcher in der Fehlermeldung erscheint, nicht korrekt im OEM erfasst ist.

Im Fall der oben gezeigten Fehlermeldung, stellte sich heraus, dass als Monitoring Agent der Agent [exa202s52-adm.st.zkb.ch](http://exa202s52-adm.st.zkb.ch/) (anstatt [exa202s02-adm.st.zkb.ch](http://exa202s02-adm.st.zkb.ch/)) definiert war.

Die Konfiguration eines Access Points kann wie folgt eingesehen werden:

[https://em13.prod.zkb.ch/em/faces/core-uifwk-console-overview](https://em13.prod.zkb.ch/em/faces/core-uifwk-console-overview)

Target auswählen (in diesem Fall [exa202s02-mgmt.st.zkb.ch](http://exa202s02-mgmt.st.zkb.ch/))

Monitoring => Access Points - Overview

# Fehlerbehebung

Falls in der Access Points - Overview eine fehlerhafte Konfiguration festgestellt wird, muss der entsprechende Access Point (ggf. beide) gelöscht werden.

Anschliessend muss der Access Point via emcli neu erstellt werden. Den emcli Befehl kann man z.B auf dem betroffenen Exadata Server ausführen, oder wo auch immer emcli konfiguriert ist.

Damit das Monitoring redundant an das entsprechende ILOM angebunden wird, müssen zwei verschiedene Agenten verwendet werden. Dies ist im untenstehenden Befehl rot gekennzeichnet.

# emcli Befehl 

```bash
/usr/local/db/emcli13/emclisrc13/emcli13.pl add_target -name=[exa202s02-mgmt.st.zkb.ch](http://exa202s02-mgmt.st.zkb.ch/) -type=oracle_si_server_map -host=[exa202s02-adm.st.zkb.ch](http://exa202s02-adm.st.zkb.ch/) -access_point_name=exa202s02-mgmt.st.zkb.ch_ILOM_AP1  -access_point_type=oracle_si_server_ilom -subseparator=properties== '-properties=SkipSnmpSubscription=true;dispatch.url=[ilom-ssh://10.125.63.24'](ilom-ssh://10.125.63.24') '-monitoring_cred=ilom_creds_set;oracle_si_server_ilom;ilom_creds;username:oemuser;password:<PW>'

/usr/local/db/emcli13/emclisrc13/emcli13.pl add_target -name=[exa202s02-mgmt.st.zkb.ch](http://exa202s02-mgmt.st.zkb.ch/) -type=oracle_si_server_map -host=[exa202s01-adm.st.zkb.ch](http://exa202s01-adm.st.zkb.ch/) -access_point_name=exa202s02-mgmt.st.zkb.ch_ILOM_AP2  -access_point_type=oracle_si_server_ilom -subseparator=properties== '-properties=SkipSnmpSubscription=true;dispatch.url=[ilom-ssh://10.125.63.24'](ilom-ssh://10.125.63.24') '-monitoring_cred=ilom_creds_set;oracle_si_server_ilom;ilom_creds;username:oemuser;password:<PW>'

=> Die Passwörter sind im Keepass abgelegt ([\\DATENPOOL\Datenpl$\Matrix\DATABASE\Oracle\Exadata\Exadata_AT_PROD.kdbx](file://datenpool/Datenpl$/Matrix/DATABASE/Oracle/Exadata/Exadata_AT_PROD.kdbx)) => exaserver-mgmt:oemuser

Falls der server_os Eintrag fehlen würde:

/usr/local/db/emcli13/emclisrc13/emcli13.pl add_target -name=[exa202s01-mgmt.st.zkb.ch](http://exa202s01-mgmt.st.zkb.ch/) -type=oracle_si_server_map -host=[exa202s01-adm.st.zkb.ch](http://exa202s01-adm.st.zkb.ch/) -access_point_name=[exa202s01-mgmt.st.zkb.ch/server_os](http://exa202s01-mgmt.st.zkb.ch/server_os) -access_point_type=oracle_si_server_os -properties='dispatch.url![local://localhost'](local://localhost') -subseparator=properties='!'
```

Tags:
[[Exadata]] - [[HowTo]] - [[emcli13]] - [[Monitoring]]


