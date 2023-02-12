
### Zurücksetzen eines Benutzerpassworts im LDAP

Passwörter zurücksetzen geht am einfachsten mittles .ldiff file und dem ldap modify befehl.
Der LDAP-modify Befehl benötigt dafür ein File mit der folgenden Struktur/Inhalt

```
dn: cn=IAMAgent,ou=zkbApplicationProcess,o=zkb,c=ch
changetype: modify
replace: userPassword
userPassword: {NEUES_PASSWORT}
```

der *dn:*  muss natürlich entsprechend dem gewünschten User angepasst werden (hier im Beispiel würde das PAsswort des Users für den IAM Agenten geändert werden.

Dann mittels ldap-modify das ganze ausführen:

```bash
ldapmodify -h [oidsrv1.prod.zkb.ch](http://oidsrv1.prod.zkb.ch/) -p 3060 -D "cn=orcladmin" -w $(readpw -l orcladmin_prod) -f change_userpw.ldiff
```

quittiert sollte das ganze etwa wie folgt werden:

> modifying entry cn=IAMAgent,ou=zkbApplicationProcess,o=zkb,c=ch

Danach noch verifizieren das der User mit dem entsprechen Passwort auch zugreifen kann:

```
ldapbind -h [oidsrv2.prod.zkb.ch](http://oidsrv2.prod.zkb.ch/) -p 3060 -D "cn=IAMAgent,ou=zkbApplicationProcess,o=zkb,c=ch" -w {NEUES_PASSWORT}
```

Test ist erfolgreich wenn der Befehl mit *bind successful* quittiert wird.

### Tags:

[[HowTo]] - [[OUD]] - [[LDAP]]

