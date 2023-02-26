
# Abstract

Auf neu installierten Servern muss der User oraansi hinzugefügt werden, sodass der ansible Server sich passwortlos und mittels ssh-key auf dem neuen Server anmelden kann.

# Playbook

## Ablage

ansible-ora.eng.zkb.ch]:/usr/app/ansible/dev/plays/add_oraansi.yml

## Tasks

- Gruppe oraansi mit GID 1188 erstellen​
- User oraansi mit UID 1188 und GID 1188 erstellen
- ssh-public-key des lokalen oraansi-Users auf den remote-server transferieren
- sudo Eintrag im /etc/sudoers hinzufügen
- yum Repositories konfigurieren (ol6: latest, addons, uekr3 & OEL base versionsspezifisch
- rpm Pakete installieren

## Hinweis

> Da es sich bei diesem Task um eine nicht wiederkehrende Aktion handelt, wurden die Schritte zum Installieren des oraansi Users in ein extra Playbook ausgelagert und sind nicht Bestandteil der in der best practice Struktur enthaltenen Playbooks.

## Ausführung

Das Playbook wird direkt nach der Installation des betreffenden Servers ausgeführt und fragt bei Ausführung nach dem root-Passwort des remote-Servers. 

Im Gegensatz zu den anderen Playbooks wird dieses Playbook mit dem root User auf dem Remote-Server ausgeführt. Alle anderen Playbook verwenden den remote-User oraansi für den Login.  

Das Playbook wird mit dem Commandline-Parameter --limit ausgeführt. Damit wird der entsprechende remote-Server aus dem angegebenen Repository selektiert, sodass das Playbook nicht auf allen Servern ausgeführt wird.

## Command Line

ansible-playbook /usr/app/ansible/dev/plays/add_oraansi.yml -u root -i ../repository/dev/servers.list --limit [exa301s51.eng.zkb.ch](http://exa301s51.eng.zkb.ch/) --ask-pass

# Tags:

[[Ansible]] - [[Playbook]] - [[ZKB]]
