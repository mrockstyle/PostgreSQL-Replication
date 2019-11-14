
# PostgreSQL Replication

## Variables (***ปรับเปลี่ยนตัวแปรได้ตามต้องการ)

### User variables
> user = ${USER}  
> user_group = ${USER_GRP}  

## Master variables
> hostname = ${M_NAME}  
> ip = ${M_IP}  

## Slave varibles
> hostname = ${S_NAME}  
> ip = ${S_IP}



## Table of Contents
1. [repmgr Setup](#1-repmgr-setup)
2. [Master Setup](#2-master-setup)
3. [Slave Setup](#3-slave-setup)


## 1. repmgr Setup
### 1.1 Install repmgr
```bash
cd /Source
tar -xvf repmgr-4.3.tar
chown -R ${USER}:${USER_GRP} /Source/repmgr-4.3.0
chmod -R 777 /Source/repmgr-4.3.0
mkdir -p /app/repmgr
chown -R ${USER}:${USER_GRP} /app/repmgr

# Change user
su - ${USER}
cd /Source/repmgr-4.3.0
./configure --prefix=/app/repmgr
make install
```

---

## 2. Master Setup

### 2.1 Add repmgr user
```bash
createuser -s repmgr
createdb repmgr -O repmgr
```

### 2.2 Create repmgr.conf
```bash
touch /app/repmgr/repmgr.conf

echo "node_id=1
node_name=${M_NAME}
conninfo='host=${M_IP} user=repmgr dbname=repmgr port=5432'
data_directory=/DB/postgresql/${USER}/etc/data
pg_bindir=/DB/postgresql/${USER}/etc/bin
log_file=/app/repmgr/repmgr.log" > /app/repmgr/repmgr.conf
```

### 2.3 Edit pg_hba.conf
```bash
vi /DB/postgresql/${USER}/etc/data/pg_hba.conf

# เพิ่ม config ตามด้านล่าง

host	repmgr		    repmgr		    ${S_IP}/32		    trust
host	replication	    repmgr		    ${S_IP}/32		    trust
host	repmgr		    repmgr		    ${M_IP}/32		    trust
host    replication   	repmgr      	127.0.0.1/32            trust

# จะได้ผลดังนี้

# TYPE  DATABASE        USER            ADDRESS                 METHOD
host	repmgr		    repmgr		    ${S_IP}/32		    trust
host	replication	    repmgr		    ${S_IP}/32		    trust
host	repmgr		    repmgr		    ${M_IP}/32		    trust
host    replication   	repmgr      	127.0.0.1/32            trust
```

### 2.4 Edit postgresql.conf
```bash
vi /DB/postgresql/${USER}/etc/data/postgresql.conf

# แก้ไขตัวแปรด้านล่างดังนี้

wal_level = replica
archive_mode = on
archive_command = 'cp -i %p /DB/postgresql/insttxn/log/pg_archive/%f </dev/null'
max_wal_senders = 50
hot_standby = on
```

### 2.5 Register primary node
```bash
repmgr -f /app/repmgr/repmgr.conf primary register
```

### 2.6 Check cluster
```bash
psql -U repmgr -c 'select * from repmgr.nodes' -d repmgr
```

### 2.7 Restart PostgreSQL
```bash
/etc/init.d/etc/init.d/postgres_${USER} restart
```
---

## 3. Slave Setup
> **Install new postgresql instance without init and without start**

### 3.1 Add repmgr user
```bash
createuser -s repmgr
createdb repmgr -O repmgr
```

### 3.2 Create repmgr.conf
```bash
touch /app/repmgr/repmgr.conf

echo "node_id=2
node_name=${S_NAME}
conninfo='host=${S_IP} user=repmgr dbname=repmgr port=5432'
data_directory=/DB/postgresql/${USER}/etc/data
pg_bindir=/DB/postgresql/${USER}/etc/bin
log_file=/app/repmgr/repmgr.log" > /app/repmgr/repmgr.conf
```

### 3.3 Clone data from Primary
```bash
repmgr -h ${M_IP} -U repmgr -d repmgr -f /app/repmgr/repmgr.conf standby clone
```

### 3.4 Start secondary node
```bash
/etc/init.d/postgres_${USER} start
```

### 3.5 Register secondary node
```bash
repmgr -f /app/repmgr/repmgr.conf standby -F register
```

### 3.6 Check cluster
```bash
psql -U repmgr -c 'select * from repmgr.nodes' -d repmgr
```
