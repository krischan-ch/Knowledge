Autoupgrade 2.0

Documents


[AutoUpgrade Tool (Doc ID 2485457.1)](https://support.oracle.com/epmos/faces/DocContentDisplay?_afrLoop=463443714431785&id=2485457.1)

[Convert non-CDB to PDB using AutoUpgrade](https://docs.oracle.com/en/database/oracle/oracle-database/21/upgrd/understanding-non-cdb-to-pdb-upgrades-autoupgrade.html#GUID-D739E4A4-F1B9-45BE-B0E2-F213FE70F665)


Note:

When you run AutoUpgrade in Upgrade mode, no postupgrade operations are performed, so you must complete those steps separately. For example, the following postupgrade operations are not performed: 

-   Copy of network files (`tnsnames.ora`, `sqlnet.ora`, `listener.ora` and other listener files, LDAP files, `oranfstab`
-   Removal of the guaranteed restore point (GRP) created during the upgrade
-   Final restart of an Oracle Real Application Clusters database




Tags:

[[AutoUpgrade]]   -   [[MikeDietrich]]
