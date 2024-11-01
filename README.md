# วิธีเปลี่ยน CentOS 8 เป็น Oracle Linux 8 และติดตั้ง Oracle database 19c enterprise บน Digital Ocean

## 1. ทรัพยากรที่ต้องการ
- CPU - 3.4 Ghz (2 cores)
- Memory - 8 GB
- Storage - 80 GB

## 2. เปลี่ยน CentOS 8 ให้เป็น Oracle Linux 8

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

## 3. ขั้นตอนการติดตั้ง Oracle database

### ติดตั้ง Oracle Installation Prerequisites
```bash
yum install -y oracle-database-preinstall-19c
yum update -y
```

### ติดตั้ง package ที่จำเป็น
```bash
yum install -y bc \
   binutils \
   compat-libstdc++-33 \
   elfutils-libelf \
   elfutils-libelf-devel \
   fontconfig-devel \
   glibc \
   glibc-devel \
   ksh \
   libaio \
   libaio-devel \
   libXrender \
   libXrender-devel \
   libX11 \
   libXau \
   libXi \
   libXtst \
   libgcc \
   librdmacm-devel \
   libstdc++ \
   libstdc++-devel \
   libxcb \
   make \
   net-tools \
   nfs-utils \
   python \
   python-configshell \
   python-rtslib \
   python-six \
   targetcli \
   smartmontools \
   sysstat \
   unixODBC \
   libnsl \
   libnsl.i686 \
   libnsl2 \
   libnsl2.i686
```

### สร้าง user แล้วกลุ่มต่อไปนี้
```bash
groupadd -g 54321 oinstall
groupadd -g 54322 dba
groupadd -g 54323 oper

useradd -u 54321 -g oinstall -G dba,oper oracle
```

### กำหนด Password ให้กับ user oracle
```bash
passwd oracle
```

### ตั้งค่า SELINUX เป็น permissive ในไฟล์ /etc/selinux/config
```bash
# Out File: /etc/selinux/config
...
SELINUX=permissive
...
```

ถ้าหากมี Firewall ให้ทำการ disable เสียก่อน
```bash
systemctl stop firewalld
systemctl disable firewalld
```

restart server
```bash
sudo reboot
```



### สร้าง Folder ต่อไปนี้เพื่อทำการติดตั้ง
```bash
mkdir -p /u01/app/oracle/product/19.0.0/dbhome_1
mkdir -p /u02/oradata
chown -R oracle:oinstall /u01 /u02
chmod -R 775 /u01 /u02
mkdir /home/oracle/scripts
chown -R oracle:oinstall /home/oracle/scripts
```

### ตั้งค่า Enveroinment ของ oracle
```bash
cat > /home/oracle/scripts/setEnv.sh <<EOF
# Oracle Settings
export TMP=/tmp
export TMPDIR=\$TMP

export ORACLE_HOSTNAME=ol8-19.localdomain
export ORACLE_UNQNAME=cdb1
export ORACLE_BASE=/u01/app/oracle
export ORACLE_HOME=$ORACLE_BASE/product/19.0.0/dbhome_1
export ORA_INVENTORY=/u01/app/oraInventory
export ORACLE_SID=cdb1
export PDB_NAME=pdb1
export DATA_DIR=/u02/oradata

export PATH=/usr/sbin:/usr/local/bin:\$PATH
export PATH=\$ORACLE_HOME/bin:\$PATH

export LD_LIBRARY_PATH=\$ORACLE_HOME/lib:/lib:/usr/lib
export CLASSPATH=\$ORACLE_HOME/jlib:\$ORACLE_HOME/rdbms/jlib
EOF

echo ". /home/oracle/scripts/setEnv.sh" >> .bash_profile
echo ". /home/oracle/scripts/setEnv.sh" >> /home/oracle/.bash_profile
```
สร้างไฟล์ start/stop สำหรับ application
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
```
*** วิธีใช้งานงาน
```bash
su - oracle
~/scripts/start_all.sh
~/scripts/stop_all.sh
```

## เริ่มติดตั้ง Oracle ได้

### ตั้งค่า DISPLAY environmental
```bash
DISPLAY=<machine-name>:0.0; export DISPLAY
# <machine-name> ex. --> cenos2oracle.local
```

### Donwload ไฟล์ Oracle Linux และทำการติดตั้ง
[>>Download Here<<](https://mega.nz/file/SpoG2TjK#ZzSNxc0ty_KVYWy28kwvqXwgvr58ZDo1Qu1RoUjrs7w)

ทำการ upload file วางบน server 
```bash
## ทำการ unzip ไฟล์ ติดตั้ง
cd /root
chown oracle:oinstall ./LINUX.X64_193000_db_home.zip
mv ./LINUX.X64_193000_db_home.zip /u01/app/oracle/product/19.0.0/dbhome_1/LINUX.X64_193000_db_home.zip
```
ติดตั้ง
```bash
su - oracle
cd $ORACLE_HOME
unzip -oq LINUX.X64_*_db_home.zip

# หลอกว่านี้คือ Oracle 7
export CV_ASSUME_DISTID=OEL7.6

#ติดตั้ง
./runInstaller -ignorePrereq -waitforcompletion -silent                        \
    -responseFile ${ORACLE_HOME}/install/response/db_install.rsp               \
    oracle.install.option=INSTALL_DB_SWONLY                                    \
    ORACLE_HOSTNAME=${ORACLE_HOSTNAME}                                         \
    UNIX_GROUP_NAME=oinstall                                                   \
    INVENTORY_LOCATION=${ORA_INVENTORY}                                        \
    SELECTED_LANGUAGES=en,en_GB                                                \
    ORACLE_HOME=${ORACLE_HOME}                                                 \
    ORACLE_BASE=${ORACLE_BASE}                                                 \
    oracle.install.db.InstallEdition=EE                                        \
    oracle.install.db.OSDBA_GROUP=dba                                          \
    oracle.install.db.OSBACKUPDBA_GROUP=dba                                    \
    oracle.install.db.OSDGDBA_GROUP=dba                                        \
    oracle.install.db.OSKMDBA_GROUP=dba                                        \
    oracle.install.db.OSRACDBA_GROUP=dba                                       \
    SECURITY_UPDATES_VIA_MYORACLESUPPORT=false                                 \
    DECLINE_SECURITY_UPDATES=true
```

ออกจากการ user oracle ไปยัง user root และรันไฟล์ต่อไปนี้
```bash
exit
/u01/app/oraInventory/orainstRoot.sh
/u01/app/oracle/product/19.0.0/dbhome_1/root.sh
```


## 4. สร้าง. Database

ทำการ start listner
```bash
su - oracle
lsnrctl start
```
สร้าง Instance
```bash
dbca -silent -createDatabase                                                   \
     -templateName General_Purpose.dbc                                         \
     -gdbname ${ORACLE_SID} -sid  ${ORACLE_SID} -responseFile NO_VALUE         \
     -characterSet AL32UTF8                                                    \
     -sysPassword SysPassword1                                                 \
     -systemPassword SysPassword1                                              \
     -createAsContainerDatabase true                                           \
     -numberOfPDBs 1                                                           \
     -pdbName ${PDB_NAME}                                                      \
     -pdbAdminPassword PdbPassword1                                            \
     -databaseType MULTIPURPOSE                                                \
     -memoryMgmtType auto_sga                                                  \
     -totalMemory 2000                                                         \
     -storageType FS                                                           \
     -datafileDestination "${DATA_DIR}"                                        \
     -redoLogFileSize 50                                                       \
     -emConfiguration NONE                                                     \
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

In the file you put in /etc/init.d/ you have to set it executable with:

chmod +x /etc/init.d/start_my_app
Thanks to @meetamit, if this does not run you have to create a symlink to /etc/rc.d/

ln -s /etc/init.d/start_my_app /etc/rc.d/

เป็นอันเสร็จสิ้นการเปลี่ยน CentOS 8 เป็น Oracle Linux 8 และการติดตั้ง Oracle Database



## Uninstall oracle
พิมพ์คำสั่งต่อไปนี้
```bash
$ORACLE_HOME/deinstall/deinstall
```


## อ้างอิง
- [Oracle Database 19c Installation On Oracle Linux 8](https://blog.tuningsql.com/how-to-install-oracle-19c-on-a-digitalocean-droplet-without-asm/)
- [How to Install Oracle 19c on a DigitalOcean Droplet Without ASM (Easiest Install Ever With Tons of Pictures](https://oracle-base.com/articles/19c/oracle-db-19c-installation-on-oracle-linux-8#Top)

