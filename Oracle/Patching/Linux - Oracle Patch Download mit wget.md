
# Abstract

Diese Seite beschreibt das Herunterladen von Patches vom Oracle Supportportal direkt auf die Oracle-Softwareablage.

## Repo Server
orarepo.eng.zkb.ch
orarepo.st.zkb.ch
orarepo.prod.zkb.ch

## Neues RU Unterverzeichnis anlegen

@orarepo.eng.zkb.ch​ als oracle

```bash
mount /oramnt/orarepo
cd /oramnt/orarepo/ORACLE_LINUX/patch/
mkdir ./19.9.0.0/2020-10
cd ./19.9.0.0/2020-10
```

## Download Link finden

Patch auswählen -> Download -> "WGET Options"

Im heruntregeladenen wget.sh befindet sich die URL des Patches:

# Verify if authentication is successful

```bash
if [ $? -ne 0 ]
then
echo "Authentication failed with the given credentials." | tee -a "$LOGFILE"
echo "Please check logfile: $LOGFILE for more details."
else
echo "Authentication is successful. Proceeding with downloads..." >> "$LOGFILE"

 $WGET  --load-cookies="$COOKIE_FILE" [https://updates.oracle.com/Orion/Services/download/p31771877_190000_Linux-x86-64.zip?aru=23869227&patch_file=p31771877_190000_Linux-x86-64.zip](https://updates.oracle.com/Orion/Services/download/p31771877_190000_Linux-x86-64.zip?aru=23869227&patch_file=p31771877_190000_Linux-x86-64.zip) -O "$OUTPUT_DIR/p31771877_190000_Linux-x86-64.zip"   >> "$LOGFILE" 2>&1 
```

​Sämtliche URL's der im neuen Home zu installierenden Patches werden in eine Datei urls.txt kopiert. Alle in der Datei enthaltenen URL's werden später heruntergeladen.

## Bessere Variante (ab 07/2022):

Es werden alle wget.sh Skripte heruntergeladen und im selben Verzeichnis auf dem orarepo.eng.zkb.ch deponiert.

Anschliessend kann man mit folgendem Kommando das urls.txt generieren:

```bash
grep 'WGET  --load-cookies' wget* | awk -F\" '{print $4}' >urls.txt​
```


## MoS Logindaten konfigurieren

Das persönliche Passwort für den My Oracle Support Login wird temporär auf dem Server als OS-Passwort abgelegt und nach dem Download wieder gelöscht:

## Mailadresse generieren

```bash
_oralogin="$(/usr/bin/adquery user `/usr/local/db/bin/getlogin.sh` --display | awk -F" " '{print $2 "." $1 "@zkb.ch"}')"

echo ${_oralogin}
```

## persönliches MoS Passwort ablegen
```bash
/usr/local/db/bin/mkpw -a ${_oralogin}
```

## Passwort überprüfen

```bash
/usr/local/db/bin/readpw -a ${_oralogin}
```

## Internetproxy konfigurieren

Als Proxycredentials wurde für das DB-Team der User oradownload eingerichtet.

### proxy config variablen setzen

```bash
. /usr/local/db/bin/set_proxy.sh
```

Das Skript verwendet ein im Vault hinterlegtes Passwort. Ausserdem ist das Passwort des oradownload Users im LIEUD KeePass File zu finden.

## Dateien herunterladen

```bash
for _FILE in $(grep ^https urls.txt)
do
  _FILENAME=$(echo ${_FILE} | awk -F"=" '{print $NF}')
  wget --http-user=${_oralogin} --http-password=`/usr/local/db/bin/readpw -a ${_oralogin}` "${_FILE}" -O ${_FILENAME} -o ${_FILENAME%.zip}.download.log && echo "${_FILENAME}: OK" || echo "${_FILENAME}: NOK"
done
```

## Zip-Files auspacken

**!!! OPATCH (6880880​) NICHT ENTPACKEN !!!**

```bash
for _ZIP in $(ls *zip)
do
  unzip -o ${_ZIP}
done
```

## Aufräumen

```bash
rm -f *.zip PatchSearch.xml
rm -f /app/oracle/etc/.ocrypt/oracle/${_oralogin}.*
```


### Tags:

[[HowTo]] - [[OraclePatching]] - [[MoS]]

