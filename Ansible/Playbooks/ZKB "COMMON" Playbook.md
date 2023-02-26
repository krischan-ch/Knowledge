

## DEFAULTS Directoty

### main.yml

```bash
---
  # common base filesystems
  common_base_filesystems:
    - {name: 'app_lv', vg: '{{ default_oravg }}', size: 1024, state: 'present', mntstate: 'mounted', fstype: 'ext4', mountpoint: '/app'}
  # common logical volumes for all plattforms
  common_filesystems:
    #- {name: app_oracle, vg: "{{ default_oravg }}", size: 1024, state: 'present', mntstate: 'mounted', fstype: "{{ default_fstype }}", mountpoint: '/app/oracle'}
    #- {name: app_grid, vg: "{{ default_oravg }}", size: 1024, state: 'present', mntstate: 'mounted', fstype: "{{ default_fstype }}", mountpoint: '/app/grid'}
    - {name: db_lv, vg: "{{ default_oravg }}", size: 1024, state: 'present', mntstate: 'mounted', fstype: 'ext4', mountpoint: '/usr/local/db'}
    - {name: zkb_lv, vg: "{{ default_oravg }}", size: 1024, state: 'present', mntstate: 'mounted', fstype: 'ext4', mountpoint: '/usr/local/zkb'}
    - {name: tvd_lv, vg: "{{ default_oravg }}", size: 1024, state: 'present', mntstate: 'mounted', fstype: 'ext4', mountpoint: '/usr/local/tvd'}
    - {name: oralog_lv, vg: "{{ default_oravg }}", size: 51200, state: 'present', mntstate: 'mounted', fstype: 'ext4', mountpoint: '/var/local/oracle'}
    - {name: oracle_lv, vg: "{{ default_oravg }}", size: 256, state: 'present', mntstate: 'mounted', fstype: 'ext4', mountpoint: '/usr/local/oracle'}

  # common os groups
  common_groups:
    - {name: 'oracleav', gid: '1209', state: 'present'}
    - {name: 'dba', gid: '1210', state: 'present'}
    - {name: 'osoper', gid: '1211', state: 'present'}
    - {name: 'oinstall', gid: '1222', state: 'present'}
    - {name: 'dgdba', gid: '1223', state: 'present'}
    - {name: 'bckdba', gid: '1224', state: 'present'}
    - {name: 'kmdba', gid: '1225', state: 'present'}
    - {name: 'asmdba', gid: '1221', state: 'present'}
    - {name: 'asmadmin', gid: '1220', state: 'present'}
    - {name: 'racdba', gid: '1254', state: 'present'}

  # common os users
  common_users:
    - {name: 'oracleav', uid: "{{oracleav_uid}}", desc: 'OracleAV_{{inventory_hostname}}', shell: '/bin/bash', pgrp: "{{oracleav_defgrp_id}}", ogrp: "{{oracleav_defgrp_id}}", state: 'present'}
    - {name: 'oracle', uid: "{{ oracle_uid }}", desc: 'oracle_{{inventory_hostname}}', shell: '/bin/{% if WSA is defined %}ksh{% else %}bash{% endif %}', pgrp: "{{oracle_defgrp_id}}", ogrp: '1210,{{oracleav_defgrp_id}},1211,1221,1223,1224,1225,1254', state: 'present'}

  # mount options for database filesystems
  mntopts_dbfiles: "nolock,rw,bg,hard,rsize=32768,wsize=32768,nointr,timeo=600,tcp,actimeo=0,noac,vers=3"

  # mount options for backup filesystems
  mntopts_backupfiles: "nolock,rw,bg,hard,nointr,rsize=1048576,wsize=1048576,tcp,timeo=600,noac,noauto,vers=3"

  # OEM public key file location
  oem_pubkey_file: "files/oem_keys.pub.{{_DOMAIN.stdout|lower}}"

  # ORACLE_HOME filesystems
  common_orahomes:
    - {name: ora12102181016_lv, vg: '{{ default_oravg }}', size: 30720, state: 'present', mntstate: 'mounted', fstype: 'ext4', mountpoint: '/app/oracle/product/12.1.0.2.181016'}

  # Directories
  common_directories:
    - {path: '/app', owner: "root", group: "{{ oracle_defgrp_id }}", mode: "0755", state: 'directory'}
    - {path: '/app/oracle', owner: "{{ oracle_uid }}", group: "{{ oracle_defgrp_id }}", mode: "0775", state: 'directory'}
    - {path: '/app/oracle/admin', owner: "{{ oracle_uid }}", group: "{{ oracle_defgrp_id }}", mode: "0775", state: 'directory'}
    - {path: '/app/oracle/product', owner: "{{ oracle_uid }}", group: "{{ oracle_defgrp_id }}", mode: "0775", state: 'directory'}
    - {path: '/app/grid', owner: "root", group: "{{ grid_defgrp_id }}", mode: "0755", state: 'directory'}
    - {path: '/app/grid/product', owner: "root", group: "{{ grid_defgrp_id }}", mode: "0755", state: 'directory'}
    - {path: '/app/oracle/etc', owner: "{{ oracle_uid }}", group: "{{ oracle_defgrp_id }}", mode: "0775", state: 'directory'}
    - {path: '/app/oracle/etc/.ssh', owner: "{{ oracle_uid }}", group: "{{ oracle_defgrp_id }}", mode: "0700", state: 'directory'}
    - {path: '/app/oracle/network', owner: "{{ oracle_uid }}", group: "{{ oracle_defgrp_id }}", mode: "0775", state: 'directory'}
    - {path: '/app/oracle/network/admin', owner: "{{ oracle_uid }}", group: "{{ oracle_defgrp_id }}", mode: "0755", state: 'directory'}
    - {path: '/app/base/oracle', owner: "{{ oracle_uid }}", group: "{{ oracle_defgrp_id }}", mode: "{{ oracle_defperm }}", state: 'directory'}
    - {path: '/app/base/grid', owner: "{{ grid_uid }}", group: "{{ grid_defgrp_id }}", mode: "0775", state: 'directory'}
    - {path: '/dbms', owner: "{{ oracle_uid }}", group: "dba", mode: "{{ oracle_defperm }}", state: 'directory'}
    - {path: '/dbms/oracle', owner: "{{ oracle_uid }}", group: "dba", mode: "{{ oracle_defperm }}", state: 'directory'}
    - {path: '/usr/local/db', owner: "{{ oracle_uid }}", group: "dba", mode: "{{ oracle_defperm }}", state: 'directory'}
    - {path: '/usr/local/db/bin', owner: "{{ oracle_uid }}", group: "dba", mode: "{{ oracle_defperm }}", state: 'directory'}
    - {path: '/usr/local/oracle', owner: "{{ oracle_uid }}", group: "dba", mode: "{{ oracle_defperm }}", state: 'directory'}
    - {path: '/usr/local/tvd', owner: "{{ oracle_uid }}", group: "{{ oracle_defgrp_id }}", mode: "{{ oracle_defperm }}", state: 'directory'}
    - {path: '/var/local/oracle', owner: "{{ oracle_uid }}", group: "dba", mode: "{{ oracle_defperm }}", state: 'directory'}
    - {path: '/var/local/oracle/diag', owner: "{{ oracle_uid }}", group: "dba", mode: "{{ oracle_defperm }}", state: 'directory'}
    - {path: '/var/local/oracle/rman', owner: "{{ oracle_uid }}", group: "dba", mode: "{{ oracle_defperm }}", state: 'directory'}
    - {path: '/var/local/oracle/secaud', owner: "{{ oracle_uid }}", group: "dba", mode: "{{ oracle_defperm }}", state: 'directory'}
    - {path: '/var/local/oracle/log', owner: "{{ oracle_uid }}", group: "dba", mode: "{{ oracle_defperm }}", state: 'directory'}
    - {path: '/var/local/oracle/trace', owner: "{{ oracle_uid }}", group: "dba", mode: "{{ oracle_defperm }}", state: 'directory'}
    - {path: '/app/oracle/product/av_agent12', owner: "{{ oracleav_uid }}", group: "{{ oracleav_defgrp_id }}", mode: "{{ oracleav_defperm }}", state: 'directory'}

  # common links

  common_links:
    - { src: '/app/oracle/etc/dbsrvname.txt', dest: '/etc/dbsrvname.txt', owner: "{{ oracle_uid }}", group: "{{ oracle_defgrp_id }}", mode: "{{ oracle_defperm }}"}
    - { src: '/var/local/oracle', dest: '/var/local/logs/oracle', owner: "{{ oracle_uid }}", group: "{{ oracle_defgrp_id }}", mode: "{{ oracle_defperm }}"}
    - { src: '/var/local/oracle/log', dest: '/app/oracle/network/log', owner: "{{ oracle_uid }}", group: "{{ oracle_defgrp_id }}", mode: "{{ oracle_defperm }}"}
    - { src: '/var/local/oracle/trace', dest: '/app/oracle/network/trace', owner: "{{ oracle_uid }}", group: "{{ oracle_defgrp_id }}", mode: "{{ oracle_defperm }}"}
    - { src: '/var/local/oracle/diag', dest: '/app/oracle/diag', owner: "{{ oracle_uid }}", group: "{{ oracle_defgrp_id }}", mode: "{{ oracle_defperm }}"}
    - { src: '/app/oracle/etc/oratab', dest: '/etc/oratab', owner: "{{ oracle_uid }}", group: "{{ oracle_defgrp_id }}", mode: "{{ oracle_defperm }}"}
    - { src: '/app/oracle/etc/oraInst.loc', dest: '/etc/oraInst.loc', owner: "{{ oracle_uid }}", group: "{{ oracle_defgrp_id }}", mode: "{{ oracle_defperm }}"}
    - { src: '/usr/local/oracle/mkdb12', dest: '/home/oracle/mkdb12', owner: "{{ oracle_uid }}", group: "{{ oracle_defgrp_id }}", mode: "{{ oracle_defperm }}"}

```

## META Directory

### main.yml

```bash
---
  dependencies:
    - read-zkb-facts
    #- create_zfs_oramnt
    #- set-variables
```

  
## TASKS Directory

### main.yml

```bash
---
  - debug:
      msg: "Distri:{{ ansible_distribution }}"
    tags: test

  - debug:
      msg: "exa-number: {{ exa_nr }}"
    tags: test

  - debug:
      msg: "uniq_id: {{ uniq_id }}"
    tags: test

  # create os groups
  - name: os groups
    become: true
    group:
      name: '{{ item.name }}'
      gid: '{{ item.gid }}'
      state: '{{ item.state }}'
    with_items: '{{ common_groups }}'
    tags: os_groups

  # create os users
  - name: os users
    become: true
    become_method: sudo
    become_user: root
    user:
      name: '{{ item.name }}'
      uid: '{{ item.uid }}'
      comment: '{{ item.desc }}'
      shell: '{{ item.shell }}'
      group: '{{ item.pgrp }}'
      groups: '{{ item.ogrp }}'
      state: '{{ item.state }}'
    with_items: '{{ common_users }}'
    when: not ansible_check_mode and ansible_distribution != 'AIX'     # because it will fail when groups are missing
    tags: os_users

  # create ssh keys for users
  - name: create ssh keys
    become: true
    become_method: sudo
    become_user: root
    user:
      name: '{{ item.name }}'
      generate_ssh_key: yes
      ssh_key_bits: 2048
      ssh_key_file: .ssh/id_rsa
    with_items: '{{ common_users }}'
    when: not ansible_check_mode        # because it will fail when groups are missing
    tags:
      - os_users
      - cr_user_sshkeys

  # remove user password
  - name: remove os user passwords
    become: true
    shell: '/usr/bin/passwd -d {{ item.name }}'
    with_items: '{{ common_users }}'
    when: not ansible_check_mode and ansible_distribution != 'AIX'
    tags: os_user_pws

  # create base filesystems
  - name: create base logical volumes
    become: true
    lvol:
      vg: "{{ item.vg }}"
      lv: "{{ item.name }}"
      size: "{{ item.size }}"
      state: "{{ item.state }}"
    with_items: "{{ common_base_filesystems }}"
    when: ansible_distribution != "AIX"
    failed_when: false
    tags: cr_base_fs

  - name: create base filesystems
    become: true
    filesystem:
      dev: '/dev/{{ default_oravg }}/{{ item.name }}'
      fstype: '{{ item.fstype }}'
      opts: -c
    with_items: '{{ common_base_filesystems }}'
    when: ansible_distribution != "AIX"
    tags: cr_base_fs

  - name: mount base filesystems
    become: true
    mount:
      src: '/dev/{{ default_oravg }}/{{ item.name }}'
      name: '{{ item.mountpoint }}'
      fstype: '{{ item.fstype }}'
      state: '{{ item.mntstate }}'
    with_items: '{{ common_base_filesystems }}'
    when: ansible_distribution != "AIX"
    tags: cr_base_fs

  # create filesystems
  - name: create logical volumes
    become: true
    lvol:
      vg: "{{ item.vg }}"
      lv: "{{ item.name }}"
      size: "{{ item.size }}"
      state: "{{ item.state }}"
      force: yes # --> nur bei Groessenaenderungen verwenden
    with_items: "{{ common_filesystems }}"
    when: ansible_distribution != "AIX"
    tags: cr_fs

  - name: create filesystems
    become: true
    filesystem:
      dev: '/dev/{{ default_oravg }}/{{ item.name }}'
      fstype: '{{ item.fstype }}'
      opts: -c
    with_items: '{{ common_filesystems }}'
    when: ansible_distribution != "AIX"
    tags: cr_fs

  - name: mount filesystems
    become: true
    mount:
      src: '/dev/{{ default_oravg }}/{{ item.name }}'
      name: '{{ item.mountpoint }}'
      fstype: '{{ item.fstype }}'
      state: '{{ item.mntstate }}'
    with_items: '{{ common_filesystems }}'
    when: ansible_distribution != "AIX"
    tags: cr_fs

  # create common ORACLE_HOME filesystems
  - name: create logical volumes
    become: true
    lvol:
      vg: "{{ item.vg }}"
      lv: "{{ item.name }}"
      size: "{{ item.size }}"
      state: "{{ item.state }}"
      force: no # --> nur bei Groessenaenderungen verwenden
    with_items: "{{ common_orahomes }}"
    when: ansible_distribution != "AIX"
    failed_when: false
    tags: cr_orahomes

  - name: create orahome filesystems
    become: true
    filesystem:
      dev: '/dev/{{ default_oravg }}/{{ item.name }}'
      fstype: '{{ item.fstype }}'
      opts: -c
    with_items: '{{ common_orahomes }}'
    when: ansible_distribution != "AIX"
    tags: cr_orahomes

  - name: mount filesystems
    become: true
    mount:
      src: '/dev/{{ default_oravg }}/{{ item.name }}'
      name: '{{ item.mountpoint }}'
      fstype: '{{ item.fstype }}'
      state: '{{ item.mntstate }}'
    with_items: '{{ common_orahomes }}'
    when: ansible_distribution != "AIX"
    tags: cr_orahomes

  # create common directories
  - name: create directories
    become: true
    file:
      path: '{{ item.path }}'
      state: '{{ item.state }}'
      mode: '{{ item.mode }}'
      owner: '{{ item.owner }}'
      group: '{{ item.group }}'
    with_items: '{{ common_directories }}'
    tags:
      - common_dirs

  # creation of dbsrvname.txt
  - name: get dbsrvname from /etc/hosts
    become: true
    lineinfile:
      dest: /app/oracle/etc/dbsrvname.txt
      line: "{{ ansible_fqdn }}"
      create: yes
      backup: yes
      mode: '{{ oracle_defperm }}'
      owner: '{{ oracle_uid }}'
      group: '{{ oracle_defgrp_id }}'
    tags:
      - cr_dbsrvname

  # renaming .bash_profile if existing
  #- name: rename .bash_profile if existing
    #shell: '[ -x "/home/{{ item.name }}/.bash_profile" ] && mv /home/{{ item.name }}/.bash_profile /home/{{ item.name }}/.bash_profile.inact || exit 0'
    #with_items: '{{ common_users }}'
    #when: install_os_users and item.name|lower == "oracleav"
    #tags:
      #- user_profiles

  # create .bash_profile for oracleav user
  #- name: create user profile (only for user 'oracleav')
    #template:
      #src: 'profile.{{ item.name }}.j2'
      #dest: "/home/{{ item.name }}/.profile"
      #owner: '{{ item.uid }}'
      #group: '{{ item.pgrp }}'
      #mode: 0644
      #force: yes
      #backup: yes

    #with_items: '{{ common_users }}'
    #when: install_os_users and item.name|lower == "oracleav"
    #tags:
      #- user_profiles

  # provide reporead keys
  - name: provide reporead public key
    become: true
    become_method: sudo
    become_user: oracle
    copy:
      src: 'files/reporead.pub'
      dest: '/app/oracle/etc/.ssh/id_rsa.pub'
      owner: '{{ oracle_uid }}'
      group: '{{ oracle_defgrp_id }}'
      mode: 0644
      backup: yes
    ignore_errors: True
    tags:
      - cr_repo_keys

  - name: provide reporead private key
    become: true
    become_method: sudo
    become_user: oracle
    copy:
      src: 'files/reporead.key'
      dest: '/app/oracle/etc/.ssh/id_rsa'
      owner: '{{oracle_uid}}'
      group: '{{oracle_defgrp_id}}'
      mode: 0600
      backup: yes
    ignore_errors: True
    tags:
      - cr_repo_keys

  # provide OEM public keys
#  - name: provide oem public keys
#    authorized_key:
#      user: oracle
#      path: /home/oracle/.ssh/authorized_keys2
#      state: present
#      key: "{{lookup('file', '{{oem_pubkey_file}}')}}"
#    tags:
#      - cr_oem_keys

  # script provisioning
  - name: copy check_dependent_servers.sh
    become: true
    copy:
      src: 'files/provide_scripts.sh'
      dest: '/tmp/provide_scripts.sh'
      mode: 0777
    failed_when: false
    tags:
      - provide_scripts

  - name: execute check_dependent_servers.sh
    become: true
    shell: '/tmp/provide_scripts.sh >/tmp/provide_scripts.log 2>&1'
    failed_when: false
    tags:
      - provide_scripts

  # creation of oratab file
  - name: check for oratab file
    become: true
    become_method: sudo
    become_user: oracle
    stat:
      path: /app/oracle/etc/oratab
    register: otab_stat
    tags:
      - cr_ora_cfg_files

#  - name: create oratab if not existing
#    become: true
#    become_method: sudo
#    become_user: oracle
#    lineinfile:
#      dest: /app/oracle/etc/oratab
#      create: yes
#      line: 'grid:{{ gihome }}:D'
#      owner: '{{oracle_uid}}'
#      group: '{{oracle_defgrp_id}}'
#      mode: 0644
#    when: otab_stat.stat.exists == False
#    tags:
#      - cr_ora_cfg_files

#  # creation of oraInst.loc file
#  - name: check for oraInst.loc file
#    become: true
#    become_method: sudo
#    become_user: oracle
#    stat:
#      path: /app/oracle/etc/oraInst.loc
#    register: oinstloc_stat
#    tags:
#      - cr_ora_cfg_files

  - name: create or replace empty oraInst.loc
    become: true
    become_method: sudo
    become_user: oracle
    copy:
      content: ""
      dest: '/app/oracle/etc/oraInst.loc'
      force: yes
      mode: 0644
      owner: '{{oracle_uid}}'
      group: '{{oracle_defgrp_id}}'
    tags:
     - cr_ora_cfg_files

  - name: create content of oraInst.loc
    become: true
    become_method: sudo
    become_user: oracle
    blockinfile:
      dest: '/app/oracle/etc/oraInst.loc'
      create: yes
      mode: 0644
      owner: '{{oracle_uid}}'
      group: '{{oracle_defgrp_id}}'
      block: |
        inventory_loc=/app/oraInventory
        inst_group=oinstall
    tags:
      - cr_ora_cfg_files

  - name: create sid._DEFAULT_.conf
    become: true
    become_method: sudo
    become_user: oracle
    template:
      src: sid.default.conf
      dest: /app/oracle/etc/sid._DEFAULT_.conf
      owner: "{{ oracle_uid }}"
      group: "{{ oracle_gid }}"
      mode: "{{ oracle_defperm }}"
      backup: yes
    tags:
      - cr_sidconf
```

# TEMPLATES Directory

## profile.oracleav.j2

```bash
# {{ ansible_managed }}
# .bash_profile
# Get the aliases and functions

if [ -f ~/.bashrc ]; then
            . ~/.bashrc
fi

# User specific environment and startup programs
# Aenderungen fuer User oracleav fuer Audit Vault Agent
PATH=$PATH:$HOME/bin::/app/oracle/product/av_agent12/bin
export PATH
HOST=`hostname -s` ; export HOST
#Prompt mit PS1 Variable einzeilig definieren
export PS1='#${ORACLE_SID:+{${ORACLE_SID}:${ORACLE_VERSION}\}}oracleav@${HOST}:${PWD}# '
export ORACLEAV_HOME=/app/oracle/product/av_agent12
#OVM ORA-01882: timezone region not found
export JAVA_OPTS="$JAVA_OPTS -Duser.timezone=GMT"
```

## sid.default.conf

```bash
[_DEFAULT_]
AIXTHREAD_SCOPE=S
BE_ORA_ADMIN=/app/oracle/admin/$ORACLE_SID
BE_ORA_ADMIN_SID=/app/oracle/admin/$ORACLE_SID
BE_ORA_DIAG_SID=/dbms/oracle/$ORACLE_SID/adm
BE_ORA_DIAGNOSTIC_DEST=/dbms/oracle/$ORACLE_SID/adm
NLS_LANG=AMERICAN_SWITZERLAND.WE8ISO8859P1
NLS_NUMERIC_CHARACTERS='.'''
PATH + /usr/local/db/bin B
LD_LIBRARY_PATH=${ORACLE_HOME}/ctx/lib:${LD_LIBRARY_PATH}
SQLPATH=/usr/local/db/sql
TNS_ADMIN=$ORACLE_HOME/network/admin
ORA_TWO_TASK=false
ORACLE_TRACE=/var/local/oracle/trace
ORA_CSM_MODE=line
ORACLE_TERM=lft

# POST_IFILE exportiert ZKB spezifische Variablen
POST_IFILE=$TVDZKB_BASE/bin/zkb_env.sh
ZKB_ORA_LOG=/var/local/oracle/log

# default ZKB RMAN parameters
ZKB_RMAN_LOG_DIR=/var/local/oracle/rman
ZKB_RMAN_NUM_CH_INC=4
ZKB_RMAN_NUM_CH_ARC=1
ZKB_RMAN_NUM_CH_MNT=1
ZKB_RMAN_NUM_OBSOLET={% if domshort == 'prod' %}14{% else %}7{% endif %}
ZKB_RMAN_NUM_FPS=0
BE_DB_STATUS=none
DB_BASE={% if ansible_system == 'AIX' %}/dbms/oracle{% else %}/app/oracle/admin{% endif %}
```

# VARS Directory

## main.yml

```bash
# variable declaration
install_os_packages: false
install_os_groups: true
install_os_users: true
install_lvols: true
short_domain: "{{ ansible_domain.split('.')[0] }}"

# defaults for oracle user
oracle_uid: 1210
oracle_defgrp_id: 1222
oracle_defperm: '0755'

# defaults for oracleav user
oracleav_uid: 1209
oracleav_defgrp_id: 1209
oracleav_defperm: '0755'

# defaults for grid user
grid_uid: 1220
grid_defgrp_id: 1222
grid_defperm: '0755'

# name of default oravg volumegroup
default_oravg: "{% if inventory_hostname_short | regex_search('exa')  %}{{ 'VGExaDb' }}{% else %}{{ 'oravg0' }}{% endif %}"
default_fstype: "{% if ansible_system == 'Linux' %}{{ 'ext4' }}{% else %}{{ 'jfs2' }}{% endif %}"

# default_oravg: "oravg0"

# calculate exa-nr
host_nr: "{{ ansible_hostname | regex_replace('^[a-zA-Z]*[0-9]{3}s([0-9]*)$','\\1') }}"
exa_nr: "{% if host_nr | int is divisibleby 2  %}{{ 'EVEN' }}{% else %}{{ 'ODD' }}{% endif %}"
uniq_id: "{% if exa_nr == 'EVEN' %}{{ 'G' }}{% else %}{{ 'H' }}{% endif %}"
```

# Tags:

[[Ansible]] - [[Playbook]] - [[ZKB]]
