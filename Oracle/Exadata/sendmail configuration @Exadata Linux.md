# Abstract

Diese Seite beschreibt wie man den ZKB Mailserver im Sendmail Konfigurationsfile konfiguriert.

Die ZKB Mailserver folgen dabei der Namenskonvention: mailserver.<domainname>

# Konfiguration

Der ZKB-Mailserver wird im Exadata Linux wie folgt konfiguriert (dcli auf allen Servern):

```bash
dcli -l root -g /u01/patches/dbs_group 'grep ^DS /etc/mail/sendmail.cf'

dcli -l root -g /u01/patches/dbs_group 'cp -p /etc/mail/sendmail.cf /etc/mail/sendmail.cf_save; sed "/^DS/s/DS.*/DS mailserver.$(dnsdomainname)/" /etc/mail/sendmail.cf_save > /etc/mail/sendmail.cf'

dcli -l root -g /u01/patches/dbs_group 'service sendmail restart'

dcli -l root -g /u01/patches/dbs_group 'grep ^DS /etc/mail/sendmail.cf'
```


### Tags:

[[HowTo]] - [[Exadata]] - [[sendmail]] - [[mail]]