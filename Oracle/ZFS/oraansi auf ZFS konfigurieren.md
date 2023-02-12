# Abstract

Diese Seite beschreibt die Konfiguration des oraansi Users auf den ZFS Storage Appliances.

Der oraansi User soll von den Exadata DB-Servern mittels ssh-Key passwortlos auf den Appliances einloggen und Projects/Shares für Exadata DB's konfigurieren.

# Ablauf

Auf dem ersten DB-Server wird mittels root-Login auf der ZFS der oraansi User erstellt. Dieser Task muss nur einmal ausgeführt werden.

Anschliessend wird für den oraansi-Linux-User ein RSA-Key erstellt und beim oraansi User auf der ZFS hinterlegt. Dieser Task muss auf jedem DB-Server ausgeführt werden, da der SSH-Key pro Exa-DB-Server eindeutig ist.

# Kommandos

Alle Kommandos werden als root-User ausgeführt.

# oraansi User auf ZFS erstellen

(auf dem ersten DB-Server)

```bash
. /usr/local/db/bin/dbfunc.zkb

getExaZfsNr
_PW=$(/usr/local/db/bin/genpwd.sh)

if [ -n "${_PW}" ]
then
ssh root@zfs${_ZFSNR}-db <<EOT && echo "oraansi@zfs${_ZFSNR}: ${_PW}"
  cd /
  configuration users
  local oraansi
  set initial_password="${_PW}"
  set roles=basic,role_share_project
  set fullname="oraansi User for use with Playbook"
  commit
EOT

fi
```

Das angezeigte Passwort muss im nächsten Schritt (Key hinterlegen) auf allen beteiligten DB-Servern angegeben werden. Später wird es nicht mehr benötigt, da sich der oraansi-Linux-User mittels public-key einloggt. Das Passwort muss also nicht dauerhaft abgelegt werden und kann, wenn notwendig, neu gesetzt werden.

# Key hinterlegen

neuen Key generieren

```bash
rm -f /home/oraansi/.ssh/id_rsa*
su - oraansi -c "/usr/bin/ssh-keygen -b 2048 -t rsa -f /home/oraansi/.ssh/id_rsa -q -N \"\""

# neuen Key auf ZFS hinterlegen

. /usr/local/db/bin/dbfunc.zkb
getExaZfsNr

ssh oraansi@zfs${_ZFSNR}-db <<EOT
cd /
configuration preferences keys
create
set type=RSA
set key="$(su - oraansi -c "sed -e 's/^.* \(.*\) oraansi.*$/\1/g' /home/oraansi/.ssh/id_rsa.pub")"
set comment="oraansi_$(hostname)"
commit

cd /
configuration users
select oraansi
set initial_password=$(/usr/local/db/bin/genpwd.sh)
commit
EOT
```

Dieser Schritt muss auf allen beteiligten Exa-DB-Servern ausgeführt werden.

### Tags:

[[HowTo]] - [[ZFS]] - [[oraansi]]

