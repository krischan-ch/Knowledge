
ACHTUNG VAULT (PasswortVAULT) ist noch in der Spielphase

Nötige Pfad Änderungen

```bash
export PATH=/usr/share/centrifydc/bin:/usr/share/centrifydc/kerberos/bin:$PATH

Token generieren ACHTUNG curl aus Centrify verwenden! Reighenfolge ist wichtig

curl -u : --negotiate --data '{"applicationId": 3101, "databaseName": "CIAM64", "clusterName": "CIAM64"}'  -H "Content-Type: application/json" [https://vaultgw-20.eng.zkb.ch/krb/eng/token](https://vaultgw-20.eng.zkb.ch/krb/eng/token)
```

keytab anhängen

```bash
kinit -kt /app/oracle/etc/krb5/krb5.keytab adminora_aix2737$
```

Info Philipp

Lesen

diff, resp. Update der Values von gleichnamigen Keys

Schreiben
```bash
/usr/share/centrifydc/kerberos/bin/klist

Ticket cache: FILE:/var/krb5/security/creds/krb5cc_1210

Default principal: [adminora_aix2056$@ENG.ZKB.CH](mailto:adminora_aix2056$@ENG.ZKB.CH)

Valid starting     Expires            Service principal

11/14/19 10:17:26  11/14/19 20:17:26  [krbtgt/ENG.ZKB.CH@ENG.ZKB.CH](mailto:krbtgt/ENG.ZKB.CH@ENG.ZKB.CH)

        renew until 11/15/19 10:17:26

{dummy12102180116:[12.1.0.2.180116}oracle@aix2056.eng.zkb.ch](mailto:12.1.0.2.180116%7Doracle@aix2056.eng.zkb.ch):/app/oracle/etc/krb5

> # Einmal ohne BootGrant, also ohne initialen JSON Payload: GEHT NICHT IMMER MIT PAYLOAD

{dummy12102180116:[12.1.0.2.180116}oracle@aix2056.eng.zkb.ch](mailto:12.1.0.2.180116%7Doracle@aix2056.eng.zkb.ch):/app/oracle/etc/krb5

> /usr/share/centrifydc/bin/curl -XGET --negotiate -u : [https://vaultgw-20.eng.zkb.ch/krb/eng/token](https://vaultgw-20.eng.zkb.ch/krb/eng/token)

{
  "lease_id": "",
  "request_id": "eb318602-0480-ec08-eea6-c6847d70bd0d",
  "lease_duration": "0",
  "renewable": false,
  "auth": {
    "accessor": "RreWTXJkTciY7fwN2mAf89F5",
    "client_token": "s.b27RDLfCcEiOUh2VSDhhtX8h",
    "lease_duration": "3600",
    "renewable": true,
    "entity_id": "",
    "policies": [
      "default",
      "[server-eng.zkb.ch](http://server-eng.zkb.ch/)",
      "server-self-adminora_aix2056$"
    ]
  }
}

{dummy12102180116:[12.1.0.2.180116}oracle@aix2056.eng.zkb.ch](mailto:12.1.0.2.180116%7Doracle@aix2056.eng.zkb.ch):/app/oracle/etc/krb5

> # Einmal mit BootGrant für CIAM66, man beachte die zusätzliche Rolle: vault_3103_secret_eng_eng_ciam66

{dummy12102180116:[12.1.0.2.180116}oracle@aix2056.eng.zkb.ch](mailto:12.1.0.2.180116%7Doracle@aix2056.eng.zkb.ch):/app/oracle/etc/krb5

> /usr/share/centrifydc/bin/curl -XPOST -d'{"applicationId" : 3103, "databaseName" : "CIAM66"}' -H "Content-Type: application/json" --negotiate -u : [https://vaultgw-20.eng.zkb.ch/krb/eng/token](https://vaultgw-20.eng.zkb.ch/krb/eng/token)

{
  "lease_id": "",
  "request_id": "0294505d-d166-b17d-fbfa-cef0df1d6355",
  "lease_duration": "0",
  "renewable": false,
  "auth": {
    "accessor": "Ls7QM5oEkFR7acpCQiRMWKJh",
    "client_token": "s.ZkZmEfjJo5qP9oDeeauAbHvz",
    "lease_duration": "3600",
    "renewable": true,
    "entity_id": "",
    "policies": [
      "default",
      "[server-eng.zkb.ch](http://server-eng.zkb.ch/)",
      "server-self-adminora_aix2056$",
      "vault_3103_secret_eng_eng_ciam66"
    ]
  }
}

{dummy12102180116:[12.1.0.2.180116}oracle@aix2056.eng.zkb.ch](mailto:12.1.0.2.180116%7Doracle@aix2056.eng.zkb.ch):/app/oracle/etc/krb5

> # Und die Berechtigungen für das Anlegen von secrets funktionieren auch.

{dummy12102180116:[12.1.0.2.180116}oracle@aix2056.eng.zkb.ch](mailto:12.1.0.2.180116%7Doracle@aix2056.eng.zkb.ch):/app/oracle/etc/krb5

> # Einmal unter dem eigenen Oracle-only Pfad secret/zkb/eng/oracle* wo Clustersecrets sind:

{dummy12102180116:[12.1.0.2.180116}oracle@aix2056.eng.zkb.ch](mailto:12.1.0.2.180116%7Doracle@aix2056.eng.zkb.ch):/app/oracle/etc/krb5

> curl -L -H "X-Vault-Token: s.ZkZmEfjJo5qP9oDeeauAbHvz" -H "Content-Type: application/json" -X POST -d'{"db_password":"secret"}' [https://vaultserver3-20.eng.zkb.ch/v1/secret/zkb/eng/oracle/CIAM66/host1](https://vaultserver3-20.eng.zkb.ch/v1/secret/zkb/eng/oracle/CIAM66/host1)

{dummy12102180116:[12.1.0.2.180116}oracle@aix2056.eng.zkb.ch](mailto:12.1.0.2.180116%7Doracle@aix2056.eng.zkb.ch):/app/oracle/etc/krb5

> # Und einmal unter dem Pfad, wo der RUNUSER dann mal hinkommt und wir, web, auch lesen können:

{dummy12102180116:[12.1.0.2.180116}oracle@aix2056.eng.zkb.ch](mailto:12.1.0.2.180116%7Doracle@aix2056.eng.zkb.ch):/app/oracle/etc/krb5

> curl -L -H "X-Vault-Token: s.ZkZmEfjJo5qP9oDeeauAbHvz" -H "Content-Type: application/json" -X POST -d'{"db_password":"secret"}' [https://vaultserver3-20.eng.zkb.ch/v1/secret/zkb/eng/app/database/eng/3103/host1](https://vaultserver3-20.eng.zkb.ch/v1/secret/zkb/eng/app/database/eng/3103/host1)
```

Einfach immer mit /usr/share/centrifydc/kerberos/bin/klist prüfen, welcher principal du bist. Der Ablauf ist wie folgt:

Prüfen, welcher principal man gerade ist:

```bash
$ /usr/share/centrifydc/kerberos/bin/klist

Ticket cache: FILE:/tmp/krb5cc_12209_EdGtNJWAhQ

Default principal: [tl09@PROD.ZKB.CH](mailto:tl09@PROD.ZKB.CH)
```

Ein vault bearer Token holen mit dem aktuellen principal:

```bash
curl -XGET --negotiate -u : [https://vaultgw-20.eng.zkb.ch/krb/eng/token](https://vaultgw-20.eng.zkb.ch/krb/eng/token)

{
  "lease_id": "",
  "request_id": "6298421c-2c8a-fe75-11f0-b84ce5fce2c0",
  "lease_duration": "0",
  "renewable": false,
  "auth": {
    "accessor": "9e31f0fa-e1e0-f589-9cec-e1e4345f7458",
    "client_token": "95afbe96-9515-caa3-69d8-925c71d93eed",
    "lease_duration": "3600",
    "renewable": true,
    "entity_id": "",
    "policies": [
      "default",
      "user-self-tl09",
      "vault_3103_secret_eng_eng",
      "vault_3360_secret_eng_eng",
      "vault_3360_secret_it_eng",
      "vault_web_secret_eng_eng"
    ]
  }
}
```

Relevant aus dem obigen JSON ist das client_token wie auch die policies. Aktuell musst man die policies "händisch" vergleichen mit dem "name" Attribut aus der angehängten Datei RolesSTD.json. als Beispiel nehme ich die Policy "vault_web_secret_eng_eng" aus dem obigen JSON. Diese passt auf den regex: "name": "vault_web_secret_(eng|it|st|prod)_(eng|st|prod)". Dies resultiert danach auf die nachfolgenden Rechte in vault, gem dem RolesSTD.json:

```bash
    "GWPolicy": [
      {
        "path": "secret/*",
        "capabilities": ["list"]
      },
      {
        "path": "secret/zkb/global/*",
        "capabilities": ["read", "create", "list"]
      },
      {
        "path": "secret/zkb/{%2}/app/web/{%1}/*",
        "capabilities": ["create", "read", "update", "delete", "list"]
      },
      {
        "path": "secret/zkb/{%2}/app/database/{%1}/*",
        "capabilities": ["create", "read", "update", "delete", "list"]
      },
      {
        "path": "auth/token/renew-self",
        "capabilities": ["create", "read", "update", "delete", "list"]
      }
    ]
```

Ich kann jetzt also mit dem bearer Token 95afbe96-9515-caa3-69d8-925c71d93eed unter dem Pfad secret/zkb/eng/app/web/eng/1744 ein Secret anlegen/updaten mit dem Key db_password und dem value secert wie folgt:

```bash
$ curl -L -H "X-Vault-Token: 95afbe96-9515-caa3-69d8-925c71d93eed" -H "Content-Type: application/json" -X POST -d'{"db_password":"secret"}' [https://vaultserver3-20.eng.zkb.ch/v1/secret/zkb/eng/app/web/eng/1744](https://vaultserver3-20.eng.zkb.ch/v1/secret/zkb/eng/app/web/eng/1744)
```

Jetzt noch zum Feature, dass man die appId und den Datenbanknamen direkt beim holen des Tokens angeben kann und somit weitere, höhere Rechte erhält (gilt nur für String principal == $principal.startsWith("adminora_")). Hierzu muss man einen POST Request machen gegen den vaultgw und auch noch JSON Payload mitgeben. Die Authentifizierung bleibt Kerberos. Z.B. für die appId = 3103 und die db = CIAM66. Der JSON Payload wäre: {"applicationId" : 3103, "databaseName" : "CIAM66"}

```bash
$ curl -XPOST -d'{"applicationId" : 3103, "databaseName" : "CIAM66"}' --negotiate -u : [https://vaultgw-20.eng.zkb.ch/krb/eng/token](https://vaultgw-20.eng.zkb.ch/krb/eng/token)
```

Nach diesem Call müsste eine Policy enthalten sein wie folgt: vault_3103_secret_eng_eng_CIAM66. Dieses Feature konnte ich noch nie wirklich "in the wild" testen, darum wird es mit grosser Sicherheit nicht funktionieren.

Aktuelle Policy Settings (Könen jederzeit ändern ) Referenz bei Philipp


```bash
[
  {
    "name": "vault_web_secret_(eng|it|st|prod)_(eng|st|prod)",
    "GWPolicy": [
      {
        "path": "secret/*",
        "capabilities": ["list"]
      },
      {
        "path": "secret/zkb/global/*",
        "capabilities": ["read", "create", "list"]
      },
      {
        "path": "secret/zkb/{%2}/app/web/{%1}/*",
        "capabilities": ["create", "read", "update", "delete", "list"]
      },
      {
        "path": "secret/zkb/{%2}/app/database/{%1}/*",
        "capabilities": ["create", "read", "update", "delete", "list"]
      },
      {
        "path": "auth/token/renew-self",
        "capabilities": ["create", "read", "update", "delete", "list"]
      }
    ]
  },
  {
    "name": "vault_db_secret_(eng|it|st|prod)_(eng|st|prod)",
    "GWPolicy": [
      {
        "path": "secret/*",
        "capabilities": ["list"]
      },
      {
        "path": "secret/zkb/global/*",
        "capabilities": ["read", "create", "list"]
      },
      {
        "path": "secret/zkb/{%2}/oracle/*",
        "capabilities": ["create", "read", "update", "delete", "list"]
      },
      {
        "path": "secret/zkb/{%2}/app/database/{%1}/*",

        "capabilities": ["create", "read", "update", "delete", "list"]
      },
      {
        "path": "auth/token/renew-self",
        "capabilities": ["create", "read", "update", "delete", "list"]
     }
    ]
  },
  {
    "name": "vault_([0-9][0-9][0-9][0-9])_secret_(eng|it|st|prod)_(eng|st|prod)",
    "GWPolicy": [
      {
        "path": "secret/*",
        "capabilities": ["list"]
      },
      {
        "path": "secret/zkb/global/*",
        "capabilities": ["read", "list"]
      },
      {
        "path": "secret/zkb/{%3}/global/*",
        "capabilities": ["read", "list"]
      },
      {
        "path": "secret/zkb/{%3}/app/web/{%2}/{%1}/*",
        "capabilities": ["read", "list"]
      },
      {
        "path": "secret/zkb/{%3}/app/database/{%2}/{%1}/*",
        "capabilities": ["read", "list"]
      },
      {
        "path": "auth/token/renew-self",
        "capabilities": ["create", "read", "update", "delete", "list"]
      }
    ]
  },
  {
    "name": "vault_([0-9][0-9][0-9][0-9])_secret_(eng|it|st|prod)_(eng|st|prod)_([A-Z]*[0-9]*)",
    "GWPolicy": [
      {
        "path": "secret/*",
        "capabilities": ["list"]
      },
      {
        "path": "secret/zkb/global/*",
        "capabilities": ["read", "list"]
      },
      {
        "path": "secret/zkb/{%3}/global/*",
        "capabilities": ["read", "list"]
      },
      {
        "path": "secret/zkb/{%3}/app/database/{%2}/{%1}/*",
        "capabilities": ["create", "read", "update", "delete", "list"]
      },
      {
        "path": "secret/zkb/{%3}/oracle/{%4}/*",
        "capabilities": ["create", "read", "update", "delete", "list"]
      },
      {
        "path": "auth/token/renew-self",
        "capabilities": ["create", "read", "update", "delete", "list"]
      }
    ]
  },
  {
    "name": "[server-prod.zkb.ch](http://server-prod.zkb.ch/)",
    "GWPolicy": [
      {
        "path": "secret/zkb/global/*",
        "capabilities": ["read", "create", "list"]
      },
      {
        "path": "secret/zkb/prod/global/*",
        "capabilities": ["read", "create", "list"]
      }
    ]
  },
  {
    "name": "[server-st.zkb.ch](http://server-st.zkb.ch/)",
    "GWPolicy": [
      {
        "path": "secret/zkb/global/*",
        "capabilities": ["read", "list"]
      },
      {
        "path": "secret/zkb/at/global/*",
        "capabilities": ["read", "list"]
      }
    ]
  },
  {
    "name": "[server-eng.zkb.ch](http://server-eng.zkb.ch/)",
    "GWPolicy": [
      {
        "path": "secret/zkb/global/*",
        "capabilities": ["read", "list"]
      },
      {
        "path": "secret/zkb/eng/global/*",
        "capabilities": ["read", "list"]
      }
    ]
  },
  {
    "name": "server-self-.*",
    "GWPolicy": [
      {
        "path": "secret/zkb/{ENV}/systems/{SELF}/*",
        "capabilities": ["create", "read", "update", "delete", "list"]
      }
    ]
  },
  {
    "name": "user-self-.*",
    "GWPolicy": [
      {
        "path": "secret/zkb/{ENV}/users/{SELF}/*",
        "capabilities": ["create", "read", "update", "delete", "list"]
      }
    ]
  }
]
```


### Tags:

[[HowTo]] - [[Vault]] - [[Hashicorp]]