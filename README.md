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
dnf install -y libnsl
dnf install -y libnsl.i686
dnf install -y libnsl2
dnf install -y libnsl2.i686
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

### สร้าง Folder ต่อไปนี้
```bash
mkdir -p /u01/app/oracle/product/19.0.0/dbhome_1
mkdir -p /u01/oradata
chown -R oracle:oinstall /u01 
chmod -R 775 /u01
```

### ตั้งค่า Enveroinment ของ oracle
```bash
su - oracle
cat >> .bash_profile <<EOF

# Oracle Settings
export TMP=/tmp
export TMPDIR=\$TMP
export ORACLE_HOSTNAME=`hostname`
export ORACLE_UNQNAME=ora19c
export ORACLE_BASE=/u01/app/oracle
export ORACLE_HOME=\$ORACLE_BASE/product/19.0.0/dbhome_1
export ORA_INVENTORY=/u01/app/oraInventory
export ORACLE_SID=ora19c
export PATH=/usr/sbin:/usr/local/bin:\$PATH
export PATH=\$ORACLE_HOME/bin:\$PATH
export LD_LIBRARY_PATH=\$ORACLE_HOME/lib:/lib:/usr/lib
export CLASSPATH=\$ORACLE_HOME/jlib:\$ORACLE_HOME/rdbms/jlib
EOF
exit
```

### แก้ไขไฟล์ `/etc/hosts`
```bash
HOSTS_ENTRY=`ifconfig eth0 | grep 'inet ' | awk '{ print $2 }'`" "`hostname`
sed -i "0,/^$/ s/^\$/$HOSTS_ENTRY\n/" /etc/hosts
```

## เริ่มติดตั้ง Oracle ได้

### Donwload ไฟล์ Oracle Linux
(https://mega.nz/file/SpoG2TjK#ZzSNxc0ty_KVYWy28kwvqXwgvr58ZDo1Qu1RoUjrs7w)[>>Download here<<]

ทำการ upload file วางบน server 
```bash
## ทำการ unzip ไฟล์ ติดตั้ง
cd /root
chown oracle:oinstall /path/to/software/LINUX.X64_193000_db_home.zip
mv /path/to/software/LINUX.X64_193000_db_home.zip /u01/app/oracle/product/19.0.0/dbhome_1/LINUX.X64_193000_db_home.zip
```
ติดตั้ง
```bash
su - oracle
cd $ORACLE_HOME
unzip -oq LINUX.X64_*_db_home.zip

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


## 4. สร้าง. Database

start listner
```bash
su - oracle
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
- [Oracle Database 19c Installation On Oracle Linux 8](https://blog.tuningsql.com/how-to-install-oracle-19c-on-a-digitalocean-droplet-without-asm/)
- [How to Install Oracle 19c on a DigitalOcean Droplet Without ASM (Easiest Install Ever With Tons of Pictures](https://oracle-base.com/articles/19c/oracle-db-19c-installation-on-oracle-linux-8#Top)

