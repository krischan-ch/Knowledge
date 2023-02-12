
# dbca

dbca verlangt lediglich den zusätzlichen Parameter *-dirServiceCertificatePath /etc/pki/zkb/CA/certs/microsoft/ZKB-CA-PROD.pem* um auf SSL Verbindungen zu wechseln (File mit dem Root Zertifikat der ZKB).

# eusm

**eusm list enterprise roles**

```bash
eusm listEnterpriseRoles ldap_host='lnx3631.eng.zkb.ch' ldap_ssl_port=2636 keystore=/usr/local/db/data/eusmkeystore.jks  key_pass=Just_simple_4Cert    domain_name='OracleDefaultDomain'  realm_dn='ou=oraeng,ou
=zkbRollen,o=zkb,c=ch' ldap_user_dn='cn=eusadm,ou=user,ou=oraeng,ou=zkbrollen,o=zkb,c=ch'  ldap_user_password='<pw>'
```

**add global role**

```bash
eusm addGlobalRole enterprise_role='ORA_TEST95_ENG_ANALYSE' ldap_host='lnx3631.eng.zkb.ch' ldap_ssl_port=2636 keystore=/usr/local/db/data/eusmkeystore.jks  key_pass=Just_simple_4Cert  keystore_alias=oud_root_certificate   domain_name='OracleDefaultDomain'  realm_dn='ou=oraeng,ou=zkbRollen,o=zkb,c=ch' database_name='TEST95'  global_role='ZKB_OID_ANALYSE'  dbuser=SYSTEM dbuser_password=OoYk_DTiXNec4jFD  dbconnect_string='(DESCRIPTION = (LOAD_BALANCE=OFF)(FAILOVER=ON) (CONNECT_TIMEOUT=10)(RETRY_COUNT=3) (ADDRESS_LIST= (ADDRESS=(PROTOCOL = TCP)(Host = dbtest95h.eng.zkb.ch)(Port = 10048)) (ADDRESS=(PROTOCOL = TCP)(Host = dbtest95g.eng.zkb.ch)(Port = 10048))) (CONNECT_DATA=(SERVICE_NAME = TEST95_RW.eng.zkb.ch)))'     ldap_user_dn='cn=eusadm,ou=user,ou=oraeng,ou=zkbrollen,o=zkb,c=ch'  ldap_user_password='<pw>'
```

# keytool

Mit dem keytool wird der keyystore (Java Keystore) für eusm erstellt:

```bash
$ORACLE_HOME/jdk/bin/keytool -import -trustcacerts \
-alias oud_root_certificate \
-storetype pkcs12 \
-keystore /usr/local/db/data/eusmkeystore.jks \
-storepass <kspw> \
-import -file /etc/pki/zkb/CA/certs/microsoft/ZKB-CA-PROD.pem
```

Tags:
[[EUS]] - [[KEYTOOL]] - [[SSL]] - [[HowTo]]
