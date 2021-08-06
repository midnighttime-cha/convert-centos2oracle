# วิธีเปลี่ยน CentOS 8 เป็น Oracle Linux 8 และติดตั้ง Oracle database 19c enterprise บน Digital Ocean

## ทรัพยากรที่ต้องการ
- CPU - 3.4 Ghz (2 cores)
- Memory - 8 GB
- Storage - 80 GB

## เปลี่ยน CentOS 8 ให้เป็น Oracle Linux 8

ติดตั้ง package ที่ต้องใช้งาน
```bash
yum update -y
yum install git -y
yum install nano -y
```
ทำการ clone ไฟล์ และทำการติดตั้ง
```bash
git clone https://github.com/oracle/centos2ol.git
cd centos2ol
sudo bash centos2ol.sh
yum distro-sync -y
yum upgrade -y
```
จากนั้น reboot เครื่อง
```bash
sudo reboot
```
Disable SELINUX
```bash
sed -i 's/SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config
```
เท่านี้ก็เสร็จสิ้นการเปลี่ยน CentOS 8 เป็น Oracle Linux 8 แล้ว

## ขั้นตอนการติดตั้ง Oracle database

ตรวจสอบชื่อของ. host
```bash
cat /etc/hostname

# Ouput example
centos2oracle.local
```

ติดตั้ง Oracle Installation Prerequisites
```bash
dnf install -y oracle-database-preinstall-19c
yum update -y
```

นำคำสั่งต่อไปนี้ไปแก้ไขในไฟล์ชื่อ `/etc/sysctl.conf`
```bash
# File: /etc/sysctl.conf
...
fs.file-max = 6815744
kernel.sem = 250 32000 100 128
kernel.shmmni = 4096
kernel.shmall = 1073741824
kernel.shmmax = 4398046511104
kernel.panic_on_oops = 1
net.core.rmem_default = 262144
net.core.rmem_max = 4194304
net.core.wmem_default = 262144
net.core.wmem_max = 1048576
net.ipv4.conf.all.rp_filter = 2
net.ipv4.conf.default.rp_filter = 2
fs.aio-max-nr = 1048576
net.ipv4.ip_local_port_range = 9000 65500
...
```
จากนั้น run คำสั่ง
```bash
/sbin/sysctl -p
```

ตรวจสอบไฟล์ `/etc/security/limits.d/oracle-database-preinstall-19c.conf` ว่าเป็นไปตามคำสั่งด้านล่างหรือไม่
```bash
# File: /etc/security/limits.d/oracle-database-preinstall-19c.conf
...
oracle   soft   nofile    1024
oracle   hard   nofile    65536
oracle   soft   nproc    16384
oracle   hard   nproc    16384
oracle   soft   stack    10240
oracle   hard   stack    32768
oracle   hard   memlock    134217728
oracle   soft   memlock    134217728
...
```

ติดตั้ง package ที่จำเป็น
```bash
dnf install -y bc    
dnf install -y binutils
dnf install -y compat-libstdc++-33
dnf install -y elfutils-libelf
dnf install -y elfutils-libelf-devel
dnf install -y fontconfig-devel
dnf install -y glibc
dnf install -y glibc-devel
dnf install -y ksh
dnf install -y libaio
dnf install -y libaio-devel
dnf install -y libXrender
dnf install -y libXrender-devel
dnf install -y libX11
dnf install -y libXau
dnf install -y libXi
dnf install -y libXtst
dnf install -y libgcc
dnf install -y librdmacm-devel
dnf install -y libstdc++
dnf install -y libstdc++-devel
dnf install -y libxcb
dnf install -y make
dnf install -y net-tools # Clusterware
dnf install -y nfs-utils # ACFS
dnf install -y python # ACFS
dnf install -y python-configshell # ACFS
dnf install -y python-rtslib # ACFS
dnf install -y python-six # ACFS
dnf install -y targetcli # ACFS
dnf install -y smartmontools
dnf install -y sysstat
dnf install -y unixODBC

# New for OL8
dnf install -y libnsl
dnf install -y libnsl.i686
dnf install -y libnsl2
dnf install -y libnsl2.i686
```

สร้าง user แล้วกลุ่มต่อไปนี้
```bash
groupadd -g 54321 oinstall
groupadd -g 54322 dba
groupadd -g 54323 oper

useradd -u 54321 -g oinstall -G dba,oper oracle
```

กำหนด Password ให้กับ user oracle
```bash
passwd oracle
```

ตั้งค่า SELINUX เป็น permissive ในไฟล์ /etc/selinux/config
```bash
# Out File: /etc/selinux/config
...
SELINUX=permissive
...
````

ถ้าระบบมี firewall ให้ทำการปิดการใช้งานก่อน
```bash
systemctl stop firewalld
systemctl disable firewalld
```

สร้าง Folder ต่อไปนี้
```bash
mkdir -p /u01/app/oracle/product/19.0.0/dbhome_1
mkdir -p /u02/oradata
chown -R oracle:oinstall /u01 /u02
chmod -R 775 /u01 /u02
```

ทำการเปลี่ยนโหมดใช้. user: oracle
```bash
su - oracle
```

สร้าง. Folder
```bash
mkdir /home/oracle/scripts
```

ตั้งค่า Enveroinment ของ oracle
```bash
cat > /home/oracle/scripts/setEnv.sh <<EOF
# Oracle Settings
export TMP=/tmp
export TMPDIR=\$TMP

export ORACLE_HOSTNAME=ol8-19.localdomain
export ORACLE_UNQNAME=cdb1
export ORACLE_BASE=/u01/app/oracle
export ORACLE_HOME=\$ORACLE_BASE/product/19.0.0/dbhome_1
export ORA_INVENTORY=/u01/app/oraInventory
export ORACLE_SID=cdb1
export PDB_NAME=pdb1
export DATA_DIR=/u02/oradata

export PATH=/usr/sbin:/usr/local/bin:\$PATH
export PATH=\$ORACLE_HOME/bin:\$PATH

export LD_LIBRARY_PATH=\$ORACLE_HOME/lib:/lib:/usr/lib
export CLASSPATH=\$ORACLE_HOME/jlib:\$ORACLE_HOME/rdbms/jlib
EOF

echo ". /home/oracle/scripts/setEnv.sh" >> /home/oracle/.bash_profile
```

สร้าง ไฟล์ในการ start/stop service
```bash
cat > /home/oracle/scripts/start_all.sh <<EOF
#!/bin/bash
/home/oracle/scripts/setEnv.sh

export ORAENV_ASK=NO
oraenv
export ORAENV_ASK=YES

dbstart \$ORACLE_HOME
EOF


cat > /home/oracle/scripts/stop_all.sh <<EOF
#!/bin/bash
/home/oracle/scripts/setEnv.sh

export ORAENV_ASK=NO
oraenv
export ORAENV_ASK=YES

dbshut \$ORACLE_HOME
EOF

chown -R oracle:oinstall /home/oracle/scripts
chmod u+x /home/oracle/scripts/*.sh

~/scripts/start_all.sh
~/scripts/stop_all.sh
```

เริ่มติดตั้ง Oracle ได้
```bash
cd $ORACLE_HOME
## ทำการ unzip ไฟล์ ติดตั้ง
unzip -oq /path/to/software/LINUX.X64_193000_db_home.zip

# หลอกว่านี้คือ Oracle 7
export CV_ASSUME_DISTID=OEL7.6

#ติดตั้ง
./runInstaller -ignorePrereq -waitforcompletion -silent \
    -responseFile ${ORACLE_HOME}/install/response/db_install.rsp \
    oracle.install.option=INSTALL_DB_SWONLY \
    ORACLE_HOSTNAME=${ORACLE_HOSTNAME}\
    UNIX_GROUP_NAME=oinstall \
    INVENTORY_LOCATION=${ORA_INVENTORY} \
    SELECTED_LANGUAGES=en,en_GB \
    ORACLE_HOME=${ORACLE_HOME} \
    ORACLE_BASE=${ORACLE_BASE} \
    oracle.install.db.InstallEdition=EE \
    oracle.install.db.OSDBA_GROUP=dba \
    oracle.install.db.OSBACKUPDBA_GROUP=dba \
    oracle.install.db.OSDGDBA_GROUP=dba \
    oracle.install.db.OSKMDBA_GROUP=dba \
    oracle.install.db.OSRACDBA_GROUP=dba \
    SECURITY_UPDATES_VIA_MYORACLESUPPORT=false \
    DECLINE_SECURITY_UPDATES=true 
```

ออกจากการ ser oracle ไปยัง user root และรันไฟล์ต่อไปนี้
```bash
exit
/u01/app/oraInventory/orainstRoot.sh
/u01/app/oracle/product/19.0.0/dbhome_1/root.sh
```


## สร้าง. Database

start listner
```bash
lsnrctl start
```
สร้าง Database
```bash
dbca -silent -createDatabase \
     -templateName General_Purpose.dbc \
     -gdbname ${ORACLE_SID} -sid  ${ORACLE_SID} -responseFile NO_VALUE \
     -characterSet AL32UTF8 \
     -sysPassword SysPassword1 \
     -systemPassword SysPassword1 \
     -createAsContainerDatabase true \
     -numberOfPDBs 1 \
     -pdbName ${PDB_NAME} \
     -pdbAdminPassword PdbPassword1 \
     -databaseType MULTIPURPOSE \
     -memoryMgmtType auto_sga \
     -totalMemory 2000 \
     -storageType FS \
     -datafileDestination "${DATA_DIR}" \
     -redoLogFileSize 50 \
     -emConfiguration NONE \
     -ignorePreReqs
```
แก้ไขไฟล์ `/etc/oratab` ให้เป็น 'Y'
```bash
cdb1:/u01/app/oracle/product/19.0.0/db_1:Y
```

เปิดใช้งาน `Oracle Managed Files (OMF)` และเช็คให้แน่ใจว่า `PDB` ทำงานอยู่
```bash
sqlplus / as sysdba <<EOF
alter system set db_create_file_dest='${DATA_DIR}';
alter pluggable database ${PDB_NAME} save state;
exit;
EOF
```

เป็นอันเสร็จสิ้นการเปลี่ยน CentOS 8 เป็น Oracle Linux 8 และการติดตั้ง Oracle Database

## อ้างอิง
(https://blog.tuningsql.com/how-to-install-oracle-19c-on-a-digitalocean-droplet-without-asm/)[https://blog.tuningsql.com/how-to-install-oracle-19c-on-a-digitalocean-droplet-without-asm/]
(https://oracle-base.com/articles/19c/oracle-db-19c-installation-on-oracle-linux-8#Top) [https://oracle-base.com/articles/19c/oracle-db-19c-installation-on-oracle-linux-8#Top]


















