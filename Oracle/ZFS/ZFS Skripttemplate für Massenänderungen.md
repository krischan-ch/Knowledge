
# Abstract

Diese Seite beschreibt ein Skripttemplate, welches für Massenänderungen von ZFS-Projekten verwendet werden kann. Für den ssh-Login auf die ZFS ist es von Vorteil wenn man auf der ZFS einen public key hinterlegt um nicht jedes Mal das Passwort eingeben zu müssen.

# Skripttemplate

Der erste Teil selektiert alle Projekte mit _DB_ im Namen und erstellt ein File mit allen gefundenen Werten. Der zweite Teil nimmt das File als Basis und führt für jedes Project den Code aus.

```bash
ssh t874@zfs301 <<EOT 2>/dev/null | grep '^ *[A-Z0-9]*_DB_.*[0-9]*$' | tr -d ' ' | grep -v '^$' >projects.list
shares
ls
EOT

for _PRJ in $(cat projects.list)
do
ssh t874@zfs301 <<EOT
shares
select ${_PRJ}
get space_total
EOT
done
```


Tags:

[[ZFS]] - [[Skripte]] - [[HowTo]]