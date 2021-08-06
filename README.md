# วิธีเปลี่ยน ContOS8 to Oracle Linux

## ทรัพยากรที่ต้องการ
- CPU - 3.4 Ghz (2 cores)
- Memory - 8 GB
- Storage - 80 GB

## 1. ทำการ clone ไฟล์ติดตั้ง
```bash
git clone https://github.com/oracle/centos2ol.git
cd centos2ol
sudo sh centos2ol.sh
```

## 2. ทำการ reboot เครื่อง
```bash
sudo reboot
```

## 3. เปิดเครื่องกลับมา server จะกลายเป็น Oracle เรียบร้อย จากนั้นทำการอัพเดท
```bash
yum distro-sync -y
yum upgrade -y
```

## 4. ทำการ Disable SELINUX
```bash
sed -i 's/SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config
```

## 5. ติดตั้ง Oracle Prerequisites
```bash
yum install -y oracle-database-preinstall-19c
yum update -y
```
## 6. สร้าง Folder
```bash
mkdir -p /u01/app/oracle/product/19.0.0/dbhome_1
mkdir -p /u01/oradata
chown -R oracle:oinstall /u01 
chmod -R 775 /u01 
```

## 7. ปรับ Environment Variables ใน Bash Profile
```bash
su - oracle
```

```bash
vi ~/.bash_profile
```

```bash
# Oracle Settings
export TMP=/tmp
export TMPDIR=$TMP

export ORACLE_HOSTNAME=oracle-db-19c.centlinux.com
export ORACLE_UNQNAME=cdb1
export ORACLE_BASE=/u01/app/oracle
export ORACLE_HOME=$ORACLE_BASE/product/19.3.0/dbhome_1
export ORA_INVENTORY=/u01/app/oraInventory
export ORACLE_SID=cdb1
export PDB_NAME=pdb1
export DATA_DIR=/u01/oradata

export PATH=$ORACLE_HOME/bin:$PATH

export LD_LIBRARY_PATH=$ORACLE_HOME/lib:/lib:/usr/lib
export CLASSPATH=$ORACLE_HOME/jlib:$ORACLE_HOME/rdbms/jlib
```

```bash
source ~/.bash_profile
```

## 8. แก้ไขไฟล์ /etc/hosts
```bash
HOSTS_ENTRY=`ifconfig eth0 | grep 'inet ' | awk '{ print $2 }'`" "`hostname`
sed -i "0,/^$/ s/^\$/$HOSTS_ENTRY\n/" /etc/hosts
```

## 9. ดาวน์โหลดไฟล์ Oracle 19c
```bash
cd /root
```
(LINUX.X64_193000_db_home.zip)[https://www.oracle.com/database/technologies/oracle-database-software-downloads.html]
```bash
chown oracle:oinstall LINUX.X64_193000_db_home.zip

mv LINUX.X64_193000_db_home.zip /u01/app/oracle/product/19.0.0/dbhome_1/

su - oracle
cd $ORACLE_HOME
```

## 10. ทำการ unzip ไฟล์ติดตั้ง
```bash
unzip -oq LINUX.X64_*_db_home.zip
```

## 11. ทำการ Copy คำสั่งด้านล่าง
```bash

```

## 12. รันคำสั่งต่อไปนี้
```bash
vi ~/.bash_profile
```

```bash
/u01/app/oracle/product/19.0.0/dbhome_1/runInstaller \
-ignorePrereq -waitforcompletion -silent \
oracle.install.option=INSTALL_DB_SWONLY \
ORACLE_HOSTNAME=${ORACLE_HOSTNAME} \
UNIX_GROUP_NAME=oinstall \
INVENTORY_LOCATION=${ORA_INVENTORY} \
ORACLE_HOME=${ORACLE_HOME} \
ORACLE_BASE=${ORACLE_BASE} \
oracle.install.db.InstallEdition=EE \
oracle.install.db.OSDBA_GROUP=dba \
oracle.install.db.OSBACKUPDBA_GROUP=backupdba \
oracle.install.db.OSDGDBA_GROUP=dgdba \
oracle.install.db.OSKMDBA_GROUP=kmdba \
oracle.install.db.OSRACDBA_GROUP=racdba \
SECURITY_UPDATES_VIA_MYORACLESUPPORT=false \
DECLINE_SECURITY_UPDATES=true

/u01/app/oracle/product/19.0.0/dbhome_1/install/response/db_install.rsp
```

## 
```bash
dbca -silent -createDatabase \
-templateName General_Purpose.dbc \
-gdbname ${ORACLE_SID} -sid  ${ORACLE_SID} \
-responseFile NO_VALUE \
-characterSet AL32UTF8 \
-sysPassword V3ryStr@ng \
-systemPassword V3ryStr@ng \
-createAsContainerDatabase true \
-numberOfPDBs 1 \
-pdbName ${PDB_NAME} \
-pdbAdminPassword V3ryStr@ng \
-databaseType MULTIPURPOSE \
-automaticMemoryManagement false \
-totalMemory 800 \
-storageType FS \
-datafileDestination "${DATA_DIR}" \
-redoLogFileSize 50 \
-emConfiguration NONE \
-ignorePreReqs
```











