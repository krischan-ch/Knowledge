
# Abstract

Diese Seite beschreibt wie man eine Datenbank manuell aus dem LDAP löscht. Es muss das DB-Env gesetzt sein! Alternativ ZKB_DBNAME durch den namen der DB ersetzen.

# DB Objekt löschen

```bash
_DOMSHORT=$(hostname | awk -F\. '{print $2}')
_FNsearch=/tmp/${ZKB_DBNAME}.ldapsearch
_FNdel=/tmp/${ZKB_DBNAME}.ldapdel

echo ${_DOMSHORT}
echo ${_FNsearch}
echo ${_FNdel}

ldapsearch -h [oud.prod.zkb.ch](http://oud.prod.zkb.ch/) -p 389 -D "cn=eusadm,ou=user,ou=ora${_DOMSHORT},ou=zkbrollen,o=zkb,c=ch" -w $(pwfetch -l eusadm) -s sub -b "cn=${ZKB_DBNAME},cn=OracleContext,ou=ora${_DOMSHORT},ou=zkbrollen,o=zkb,c=ch" "(objectClass=*)" '*' | awk NF | perl -e "print reverse <>" >${_FNsearch}
```

!!! FILE KONTROLLIEREN VOR DEM AUSFÜHREN VON ldapdelete !!!

```bash
ldapdelete -h [oud.prod.zkb.ch](http://oud.prod.zkb.ch/) -p 389 -D "cn=eusadm,ou=user,ou=ora${_DOMSHORT},ou=zkbrollen,o=zkb,c=ch" -w $(pwfetch -l eusadm) -f ${_FNsearch}
```

# UniqueMember Eintrag entfernen

```bash
_CN=$(ldapsearch -h [oud.prod.zkb.ch](http://oud.prod.zkb.ch/) -p 389 -D "cn=eusadm,ou=user,ou=ora${_DOMSHORT},ou=zkbrollen,o=zkb,c=ch" -w $(pwfetch -l eusadm) -s sub -b "cn=OracleDefaultDomain,cn=OracleDBSecurity,cn=Products,cn=OracleContext,ou=ora${_DOMSHORT},ou=zkbrollen,o=zkb,c=ch" "(&(objectClass=*)(uniqueMember=cn=${ZKB_DBNAME},cn=OracleContext,ou=ora${_DOMSHORT},ou=zkbrollen,o=zkb,c=ch))" '*')

echo "dn: ${_CN}
changetype: modify
delete: uniqueMember
uniqueMember: cn=${ZKB_DBNAME},cn=OracleContext,ou=ora${_DOMSHORT},ou=zkbrollen,o=zkb,c=ch" >${_FNdel}

ldapmodify -h [oud.prod.zkb.ch](http://oud.prod.zkb.ch/) -p 389 -D "cn=eusadm,ou=user,ou=ora${_DOMSHORT},ou=zkbrollen,o=zkb,c=ch" -w $(pwfetch -l eusadm) -f ${_FNdel}
```

### Tags:

[[HowTo]] - [[OUD]] - [[LDAP]]