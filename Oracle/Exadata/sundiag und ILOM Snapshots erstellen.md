
# Abstract

Für Service Requests für Storagezellen wird ein sundiag Paket und/oder ein ILOM-Snapshot benötigt. Diese Seite erklärt wie man diese Bundles erstellt.

# sundiag

Einloggen auf entsprechender Storagezelle als root User.

Start sundiag-Collection mit folgendem Kommando:

```bash
/opt/oracle.SupportTools/sundiag.sh
```

Sundiag zeigt am Ende den Pfad eines tarball an. Dieses File kann mittels scp von einem Exa-DB-Server der gleichen Exadata zum Beispiel ins /tmp kopiert und von dort aus auf den sqltransfer geladen werden.

# ILOM-Snapshot

Diese Snapshots werden auf dem ILOM gestartet und via sftp auf den DB-Server geladen. Beschrieben ist das Vorgehen in der MoS Note [1674265.1](https://support.oracle.com/epmos/faces/DocContentDisplay?_afrLoop=45248160556119&id=1674265.1) beschrieben.

Dieses Beispiel erstellt einen ILOM-Snapshot der Storagezelle "exa102c01" und speichert dieses mittels sftp im /tmp Verzeichnis des exa102s01 Datenbankservers (Steps analog MoS note):

```bash
[root@exa102s01 ~]# ssh exa102c01-mgmt

The authenticity of host 'exa102c01-mgmt (10.125.63.65)' can't be established.
RSA key fingerprint is e8:cd:38:02:d2:ad:73:4a:85:b8:2b:67:9d:a1:eb:68.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added 'exa102c01-mgmt' (RSA) to the list of known hosts.

Password:

Oracle(R) Integrated Lights Out Manager
Version 4.0.2.26.a r123797
Copyright (c) 2018, Oracle and/or its affiliates. All rights reserved.
Warning: HTTPS certificate is set to factory default.

Hostname: exa102c01-mgmt

-> show /SYS
/SYS
    Targets:
       DBP
        FAN_FAULT
        INTSW
        LOCATE
        MB
        OK
        PS0
        PS1
        PS_FAULT
        PWRBS
        SERVICE
        SP
        TEMP_FAULT
        T_AMB
        VPS
        VPS_CPUS
        VPS_FANS
        VPS_MEMORY
    Properties:
        type = Host System
        ipmi_name = /SYS
        product_name = ORACLE SERVER X5-2L
        product_part_number = 7309556
        product_serial_number = 1546NM70JV
        product_manufacturer = Oracle Corporation
        fault_state = OK
        clear_fault_action = (none)
        power_state = On
    Commands:
        cd
        reset
        set
        show
        start
        stop
-> show /SP/network
/SP/network
    Targets:
        interconnect
        ipv6
        test
    Properties:
        commitpending = (Cannot show property)
        dhcp_clientid = none
        dhcp_server_ip = none
        ipaddress = 10.125.63.65
        ipdiscovery = static
        ipgateway = 10.125.48.1
        ipnetmask = 255.255.240.0
        macaddress = 00:10:E0:8D:9A:A6
        managementport = MGMT
        outofbandmacaddress = 00:10:E0:8D:9A:A6
        pendingipaddress = 10.125.63.65
        pendingipdiscovery = static
        pendingipgateway = 10.125.48.1
        pendingipnetmask = 255.255.240.0
        pendingmanagementport = MGMT
        pendingvlan_id = (none)
        sidebandmacaddress = 00:10:E0:8D:9A:A7
        state = ipv4-only
        state = ipv4-only
        vlan_id = (none)
    Commands:
        cd
        set
        show
-> show /SP/clients/dns
/SP/clients/dns
    Targets:
    Properties:
        auto_dns = enabled
        nameserver = 10.35.8.59
        retries = 1
        searchpath = [prod.zkb.ch](http://prod.zkb.ch/)
        timeout = 5
    Commands:
        cd
        set
        show
-> set /SP/diag/snapshot dataset=normal
Set 'dataset' to 'normal'
-> set /SP/diag/snapshot dump_uri=[sftp://root](sftp://root):<root-PW des exa102s01-adm>@10.125.63.61/tmp/
Set 'dump_uri' to '[sftp://root:cZHiU7inRggN_r2q4R5B@10.125.63.61/tmp/'](sftp://root:cZHiU7inRggN_r2q4R5B@10.125.63.61/tmp/')
-> cd /SP/diag/snapshot
/SP/diag/snapshot
-> show
/SP/diag/snapshot
    Targets:
    Properties:
        dataset = normal
        dump_uri = (Cannot show property)
        encrypt_output = false
        result = Running
    Commands:
        cd
        set
        show
-> show
/SP/diag/snapshot
    Targets:
    Properties:
        dataset = normal
        dump_uri = (Cannot show property)
        encrypt_output = false
        result = Collecting data into [sftp://root@10.125.63.61/tmp//exa102c01-mgmt_1546NM70JV_2018-10-31T09-06-09.zip](sftp://root@10.125.63.61/tmp//exa102c01-mgmt_1546NM70JV_2018-10-31T09-06-09.zip)
                 Snapshot Complete.
                 Done.
    Commands:
        cd
        set
        show
```

Tags:
[[HowTo]] - [[Exadata]] - [[ILOM]] - [[sundiag]]

