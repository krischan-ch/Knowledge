
# TNS Configuration files – Search order

Oracle clients (and programs that use Oracle clients/drivers) will search for TNS configuration files such as sqlnet.ora and tnsnames.ora in the following order.

Note that the first file to be found will be used -- so if there were files in both location 3 and location 5, then the one in location 3 would be found first and used.

# For UNIX systems

- $HOME (for hidden (dot) files only -- e.g. .sqlnet.ora and .tnsnames.ora)
- $TNS_ADMIN
- $HOME
- /etc or /var/opt/oracle (depending on platform)
- $ORACLE_HOME/network/admin


# For Windows systems

- current path (associated with the running client application)
- Environment variable TNS_ADMIN defined for the user/session
- Environment variable TNS_ADMIN defined for the system
- Windows Registry Key TNS_ADMIN (beware of virtualized applications that have their own private version of the registry which you cannot view).
- %ORACLE_HOME%\network\admin

# Tags:

[[HowTo]] - [[tns]] - [[TNS_ADMIN]]

