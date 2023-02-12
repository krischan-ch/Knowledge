
# Abstract
Diese Seite gibt einen Einstieg in das ldapsearch Tool, zeigt dessen Möglichkeiten und bietet einige Beispiele und HowTo's.

### User-Infos anzeigen

Das folgende Kommando sucht nach einem User im Tree cn=Users,dc=eng,dc=zkb,dc=ch dessen uid dem Wert t159 entspricht.

```bash
ldapsearch -h [oidsrv1.eng.zkb.ch](http://oidsrv1.eng.zkb.ch/) -p 3060 -D "cn=orcladmin" -w $(readpw -l orcladmin) -b "ou=oraeng,ou=zkbrollen,o=zkb,c=ch" "uid=t159"
```

### Passwort-Policies anzeigen

```bash
ldapsearch -h [oidsrv1.eng.zkb.ch](http://oidsrv1.eng.zkb.ch/) -p 3060 -D "cn=orcladmin" -w $(readpw -l orcladmin) -s sub "objectclass=pwdpolicy"
```

### Passwort-Locktime und -Failuretime eines User verifizieren

```bash
ldapsearch -h [oidsrv1.eng.zkb.ch](http://oidsrv1.eng.zkb.ch/) -p 3060 -D "cn=orcladmin" -w $(readpw -l orcladmin) -b "cn=gisuser,ou=funktionaleuser,ou=gis,ou=zkbrollen,o=zkb,c=ch" -s sub "objectclass=*" pwdfailuretime pwdaccountlockedtime 
```

### Enterpriserollen für eine Datenbank finden

```bash
ldapsearch -h [oidsrv1.prod.zkb.ch](http://oidsrv1.prod.zkb.ch/) -p 3060 -D "cn=orcladmin" -w $(readpw -l orcladmin) -b "ou=oraeng,ou=zkbrollen,o=zkb,c=ch" -s sub "(&(objectclass=zkbrecht)(cn=*IAM71*))" dn
```

### UniqueMember einer Enterpriserolle finden

... zeigt alle, der Rolle zugewiesenen, User an (uniquemember Attribut)

```bash
ldapsearch -h [oidsrv1.prod.zkb.ch](http://oidsrv1.prod.zkb.ch/) -p 3060 -D "cn=orcladmin" -w $(readpw -l orcladmin) -b "ou=oraeng,ou=zkbrollen,o=zkb,c=ch" -s sub "(&(objectclass=zkbrecht)(cn=ORA_IAM71_ENG_RW))" uniquemember
```

### Tags:

[[HowTo]] - [[LDAP]] - [[OUD]]

