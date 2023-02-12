
# Abfrage Unix bzw. Linux System-Typ

## Abstract

Es soll abgefragt werden, um welchen Typ von System es sind handelt (AIX DB-Server, Exadata DB-Server, ZSI Linux OVM etc.).

## Checks

Folgende Checks werden durchgeführt. Abhängig vom Ergebnis wird die Variable ZKB_SYSTEM auf den entsprechenden Ergebniswert gesetzt:

| Prüfungen | Ergebnis |
|-----------|----------|
|Verzeichnis /opt/oracle.SupportTools/exachk vorhanden|EXA|
|Verzeichnis /EXAVMIMAGES/conf vorhanden|OVMHYP|
|Prozess ovmd läuft|OVM|
|Befehl uname liefert AIX|AIX|
|Prozess vmtoolsd läuft und Befehl uname liefert Linux|VMWARE_ZSILNX|
|Befehl uname liefert Linux|LNX|
|alles andere|NIL|

## Skript

In der Subroutine systemType  in dbfunc.zkb wird mittels folgender Logik der Parameter ZKB_SYSTEM ermittelt und in der shell exportiert.
Mit der gleichen Logik gibt es für perl im /usr/local/db/lib/ZKB/Util.pm die sub getSystemType, welche als return Wert die gleichen Strings liefert.
Hier die Shell Logik
```
if [ -d "/opt/oracle.SupportTools/exachk" ] && [ -x "/etc/rc.d/init.d/cellirqbalance" ]
then
  ZKB_SYSTEM=EXA
elif [ -d "/EXAVMIMAGES/conf" ]
then
  ZKB_SYSTEM=OVMHYP
elif ps -ef | grep [o]vmd >/dev/null 2>&1
then
  ZKB_SYSTEM=OVM
elif [ "$(uname)" == "AIX" ]
then
  ZKB_SYSTEM=AIX
elif ps -ef | grep [v]mtoolsd >/dev/null 2>&1 && [ "$(uname)" == "Linux" ]
then
  ZKB_SYSTEM=VMWARE_ZSILNX
elif [ "$(uname)" == "Linux" ]
then
  ZKB_SYSTEM=LNX
else
  ZKB_SYSTEM=NIL
fi
```

Somit kann in den Shell Scripten nach dem sourcen von dbfunc.zkb  und nach Aufruf der sub systemType  die Variable  ZKB_SYSTEM  abgefragt werden:

```
. dbfunc.zkb
systemType

if [ "${ZKB_SYSTEM}" == "EXA" ]  ; then
..echo "ist ne Exadata"
fi
```

Für die Scripte, die auf Exadata Systemen nicht laufen sollen und in der Exadata_Script_liste als obsolet@exa gelistet sind, kann folgender Code verwendet werden:

```
. /usr/local/db/bin/dbfunc.zkb 
if systemType  ; then
  if  [ "${ZKB_SYSTEM}" == "EXA"  -o  "${ZKB_SYSTEM}" == "OVM"  -o  "${ZKB_SYSTEM}" == "OVMHYP" ] ; then
    doLog E " Script $0 should not run on a ${ZKB_SYSTEM} system"
    exit 1
  fi
else
  doLog W "Sub systemType failed with Error $? "
fi
```

Tags:
[[Exadata]] - [[OVM]] - [[HowTo]]
