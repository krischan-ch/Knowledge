21cNF - Expressions in Initialization Parameters

[Oracle Doku](|https://docs.oracle.com/en/database/oracle/oracle-database/21/refrn/parameter-files.html#GUID-6C5C109F-B60E-4407-AB13-991B84B5F464)

### Example: diagnostic_dest Parameter

#### bevore chaning the parameter

```SQL
[oracle@itchy ~]$ sqlplus / as sysdba

SQL*Plus: Release 21.0.0.0.0 - Production on Sat Jul 23 08:04:14 2022
Version 21.3.0.0.0

Copyright (c) 1982, 2021, Oracle.  All rights reserved.


Connected to:
Oracle Database 21c Enterprise Edition Release 21.0.0.0.0 - Production
Version 21.3.0.0.0

SQL> show parameter diagnostic

NAME				     TYPE	 VALUE
------------------------------------ ----------- ------------------------------
diagnostic_dest 		     string	 /opt/oracle
diagnostics_control		     string	 IGNORE
```

#### creating a new admin directory

```SHELL
[oracle@itchy oradata]$ mkdir -p /u01/oracle/admin/CDB01
[oracle@itchy oradata]$ export ADMIN_DIR=/u01/oracle/admin/CDB01
```

#### changing diagnostic_dest parameter

```SQL
[oracle@itchy oradata]$ sqlplus / as sysdba

SQL*Plus: Release 21.0.0.0.0 - Production on Sat Jul 23 08:07:55 2022
Version 21.3.0.0.0

Copyright (c) 1982, 2021, Oracle.  All rights reserved.


Connected to:
Oracle Database 21c Enterprise Edition Release 21.0.0.0.0 - Production
Version 21.3.0.0.0

SQL> alter system set diagnostic_dest='$ADMIN_DIR/';

System altered.

SQL> show parameter diagnostic

NAME				     TYPE	 VALUE
------------------------------------ ----------- ------------------------------
diagnostic_dest 		     string	 $ADMIN_DIR/
diagnostics_control		     string	 IGNORE

SQL> create pfile='/tmp/initCDB01.ora' from spfile;

File created.

SQL> exit
Disconnected from Oracle Database 21c Enterprise Edition Release 21.0.0.0.0 - Production
Version 21.3.0.0.0
```

#### check the new admin dir

```SHELL
[oracle@itchy oradata]$ ls -la $ADMIN_DIR
total 0
drwxr-xr-x. 3 oracle oinstall 18 Jul 23 08:08 .
drwxr-xr-x. 3 oracle oinstall 19 Jul 23 08:05 ..
drwxrwxr-x. 3 oracle oinstall 19 Jul 23 08:08 diag

[oracle@itchy oradata]$ fgrep -i diagnostic_dest /tmp/initCDB01.ora 
*.diagnostic_dest='$ADMIN_DIR/'
```

**note**
* the new admin dir was immediately created when executing the alter system command because scope='spfile' was not used
* the variable name will not be resolved when executing alter system; variable names will be used as they where defined
* the directories needs to be existing

# Tags:

[[NewFeatures]] - [[21c]]