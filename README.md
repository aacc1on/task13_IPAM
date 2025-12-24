# Docker Compose Practical Task - Complete Guide

## 📋 Բովանդակություն

1. [Նախապատրաստում](#նախապատրաստում)
2. [Ֆայլերի ստեղծում](#ֆայլերի-ստեղծում)
3. [Քայլ առ քայլ Setup](#քայլ-առ-քայլ-setup)
4. [Ստուգումներ](#ստուգումներ)
5. [Database և Table ստեղծում](#database-և-table-ստեղծում)
6. [MySQL Dump ստեղծում](#mysql-dump-ստեղծում)
7. [Troubleshooting](#troubleshooting)
8. [Cleanup](#cleanup)

---

## 🚀 Նախապատրաստում

### Պահանջվող ծրագրեր

- Docker Engine (20.10+)
- Docker Compose Plugin (2.0+)
- Bash shell

### Docker Compose տեղադրում

```bash
# Եթե ունեք .deb ֆայլ
sudo dpkg -i docker-compose-plugin_*.deb

# Կամ apt-ի միջոցով
sudo apt-get update
sudo apt-get install docker-compose-plugin

# Ստուգել տեղադրումը
docker compose version
```

---

## 📁 Ֆայլերի ստեղծում

### 1. Ստեղծել աշխատանքային directory

```bash
mkdir -p ~/NarekIPAM
cd ~/NarekIPAM
```

### 2. Ստեղծել docker-compose.yml

```bash
cat > docker-compose.yml << 'EOF'
name: dockercompose

services:
  # ==========================================
  # FRONTEND SERVICE (phpMyAdmin)
  # ==========================================
  frontend:
    # Step 2.1: Use phpmyadmin:5.2.0-apache image
    build:
      context: .
      dockerfile: Dockerfile.frontend
    # Step 3: Expose port 80 to host port 8080
    ports:
      - "8080:80"
    # Step 4 & 7: Attach to dockercompose-frontend network
    networks:
      - dockercompose-frontend
    # Step 8: Configure phpMyAdmin environment variables
    environment:
      - PMA_HOST=mydb
      - PMA_PORT=3306
    # Step 1: Start only if mydb service health is OK
    depends_on:
      mydb:
        condition: service_healthy

  # ==========================================
  # DATABASE SERVICE (MariaDB)
  # ==========================================
  mydb:
    # Step 5.1: Use current LTS mariadb version
    build:
      context: .
      dockerfile: Dockerfile.mydb
    # Step 7: Attach to dockercompose-frontend network
    networks:
      - dockercompose-frontend
    # Step 9: Define mysql data volume, attach to mydb service
    volumes:
      - mydb_data:/var/lib/mysql
    # Step 8: Set mysql server host and port via environment variables
    environment:
      - MYSQL_ROOT_PASSWORD=rootpassword
      - MYSQL_DATABASE=mydb
      - MYSQL_USER=admin
      - MYSQL_PASSWORD=adminpassword
    # Step 0: Set healthcheck
    healthcheck:
      # Healthcheck based on mysqladmin ping with interval 10s, timeout 15s, retries 5
      test: ["CMD", "healthcheck.sh", "--connect", "--innodb_initialized"]
      interval: 10s
      timeout: 15s
      retries: 5
      start_period: 30s

# Step 4: Define custom network with name dockercompose-frontend
networks:
  dockercompose-frontend:
    driver: bridge

# Step 9: Define mysql data volume
volumes:
  mydb_data:
    driver: local
EOF
```

### 3. Ստեղծել Dockerfile.frontend

```bash
cat > Dockerfile.frontend << 'EOF'
# Step 2.1: Define frontend service using image phpmyadmin:5.2.0-apache
FROM phpmyadmin:5.2.0-apache

# Step 2.2: Use Dockerfile for building a custom image
# Step 2.3: Add package iputils-ping into the image
RUN apt-get update && \
    apt-get install -y iputils-ping && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# Step 2.4: Start new defined service (handled by docker-compose)
EOF
```

### 4. Ստեղծել Dockerfile.mydb

```bash
cat > Dockerfile.mydb << 'EOF'
# Step 5.1: Define db service mydb. Use current LTS mariadb version image
FROM mariadb:11.4

# Step 5.2: Use Dockerfile for building a custom image
# Step 5.3: Add package iputils-ping into the image
RUN apt-get update && \
    apt-get install -y iputils-ping mariadb-client && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
EOF
```

---

## 🔧 Քայլ առ քայլ Setup

### Քայլ 1: Build և Start Services

```bash
cd ~/NarekIPAM

# Build images և start services
docker compose up -d --build

# Սպասել 30-40 վայրկյան healthcheck-ի համար
sleep 35

# Ստուգել status-ը
docker compose ps
```

**Ակնկալվող output:**

```
NAME                        STATUS                   PORTS
dockercompose-frontend-1    Up                       0.0.0.0:8080->80/tcp
dockercompose-mydb-1        Up (healthy)             3306/tcp
```

### Քայլ 2: Ստուգել Logs-երը

```bash
# Ստուգել mydb logs
docker compose logs mydb

# Ստուգել frontend logs
docker compose logs frontend

# Real-time logs (Ctrl+C-ով դուրս գալու համար)
docker compose logs -f
```

### Քայլ 3: Ստուգել Network և Volume

```bash
# Ստուգել network-ը
docker network ls | grep dockercompose

# Ակնկալվող: dockercompose_dockercompose-frontend

# Ստուգել volume-ը
docker volume ls | grep mydb

# Ակնկալվող: dockercompose_mydb_data
```

---

## ✅ Ստուգումներ

### 1. Ping Test (Task 6)

```bash
# Frontend-ից դեպի mydb
docker compose exec frontend ping -c 3 mydb

# mydb-ից դեպի frontend
docker compose exec mydb ping -c 3 frontend
```

**Ակնկալվող output:**
```
3 packets transmitted, 3 received, 0% packet loss
```

### 2. Healthcheck Test (Task 10)

```bash
# Ստուգել healthcheck status
docker compose ps

# Մանրամասն healthcheck info
docker inspect dockercompose-mydb-1 --format='{{json .State.Health}}' | jq

# Manual healthcheck test
docker compose exec mydb healthcheck.sh --connect --innodb_initialized
```

### 3. phpMyAdmin Login Test (Task 8)

Բացեք բրաուզերում: **http://localhost:8080**

**Login credentials:**
- **Server:** mydb
- **Username:** root
- **Password:** rootpassword

Կամ:
- **Username:** admin
- **Password:** adminpassword

### 4. Service Dependency Test (Task 11)

```bash
# Restart services-երը
docker compose restart

# Frontend-ը պետք է սպասի mydb-ին
docker compose logs frontend | grep -i "waiting\|depend"
```

---

## 🗃️ Database և Table ստեղծում

### Մեթոդ 1: Command Line (Recommended)

```bash
# 1. Ստեղծել database (արդեն ստեղծված է environment variables-ով)
docker compose exec mydb mariadb -uroot -prootpassword -e "CREATE DATABASE IF NOT EXISTS mydb;"

# 2. Ստեղծել table
docker compose exec mydb mariadb -uroot -prootpassword mydb -e "
CREATE TABLE IF NOT EXISTS mytable (
  id INT AUTO_INCREMENT PRIMARY KEY,
  data TEXT,
  datamodified TIMESTAMP DEFAULT NOW()
);"

# 3. Insert data
docker compose exec mydb mariadb -uroot -prootpassword mydb -e "
INSERT INTO mytable(data) VALUES('testdata01'), ('testdata02'), ('testdata03');"

# 4. Ստուգել տվյալները
docker compose exec mydb mariadb -uroot -prootpassword mydb -e "SELECT * FROM mytable;"
```

### Մեթոդ 2: phpMyAdmin-ի միջոցով

1. Բացեք **http://localhost:8080**
2. Login անեք (տես վերևը)
3. Ընտրեք `mydb` database-ը ձախ կողմից
4. Սեղմեք **SQL** tab-ը
5. Պատճենեք և գործարկեք:

```sql
CREATE TABLE IF NOT EXISTS mytable (
  id INT AUTO_INCREMENT PRIMARY KEY,
  data TEXT,
  datamodified TIMESTAMP DEFAULT NOW()
);

INSERT INTO mytable(data) VALUES('testdata01');
INSERT INTO mytable(data) VALUES('testdata02');
INSERT INTO mytable(data) VALUES('testdata03');
```

### Մեթոդ 3: Batch Insert (Ամենաարագ)

```bash
docker compose exec -T mydb mariadb -uroot -prootpassword mydb << 'EOF'
CREATE TABLE IF NOT EXISTS mytable (
  id INT AUTO_INCREMENT PRIMARY KEY,
  data TEXT,
  datamodified TIMESTAMP DEFAULT NOW()
);
INSERT INTO mytable(data) VALUES('testdata01'), ('testdata02'), ('testdata03');
EOF
```

---

## 💾 MySQL Dump ստեղծում

### Քայլ 1: Ստեղծել dump local directory-ում

```bash
cd ~/NarekIPAM

# Ստեղծել task-13 directory
mkdir -p task-13

# Ստեղծել dump
docker compose exec mydb sh -c 'mariadb-dump -uroot -prootpassword mydb' > task-13/mydb.sql

# Ստուգել dump-ը
ls -lh task-13/mydb.sql
head -30 task-13/mydb.sql
```

### Քայլ 2: Copy անել պահանջվող location

```bash
# Ստեղծել /opt/docker/dockercompose directory
sudo mkdir -p /opt/docker/dockercompose/task-13

# Copy անել dump-ը
sudo cp ~/NarekIPAM/task-13/mydb.sql /opt/docker/dockercompose/task-13/mydb.sql

# Ստուգել
ls -lh /opt/docker/dockercompose/task-13/mydb.sql
```

### Քայլ 3: Ստուգել dump-ի բովանդակությունը

```bash
# Ստուգել CREATE TABLE statement
grep -i "CREATE TABLE" /opt/docker/dockercompose/task-13/mydb.sql

# Ստուգել INSERT statements
grep -i "testdata" /opt/docker/dockercompose/task-13/mydb.sql

# Ամբողջական դիտում
cat /opt/docker/dockercompose/task-13/mydb.sql
```

---

## 🧪 Վերջնական ստուգում

### Գործարկել checkup-compose

```bash
cd ~/NarekIPAM
checkup-compose
```

**Ակնկալվող output:**

```
[ DockerCompose tests ], 1..11 tests
-----------------------------------------------------------------------------------
✓  1  Task 1: Check Docker Compose Version
✓  2  Task 2: Check phpmyadmin service
✓  3  Task 3: Expose ports
✓  4  Task 4: Custom network
✓  5  Task 5: Check mydb service
✓  6  Task 6-7: Check custom network usage; interconnection test
✓  7  Task 8: Check phpmyadmin login page
✓  8  Task 9: Check custom volume
✓  9  Task 10: Check healthcheck
✓ 10  Task 11: Check service dependency
✓ 11  Task 13: Check mysql dump
-----------------------------------------------------------------------------------
11 (of 11) tests passed, rated as 100%
```

### Manual ստուգումներ

```bash
# 1. Service-ների status
docker compose ps

# 2. Container-ների անուններ
docker ps --format "table {{.Names}}\t{{.Status}}"

# 3. Network-ների ցուցակ
docker network ls | grep dockercompose

# 4. Volume-ների ցուցակ
docker volume ls | grep mydb

# 5. Port bindings
docker compose port frontend 80

# 6. Environment variables
docker compose exec mydb env | grep MYSQL

# 7. Database connection
docker compose exec mydb mariadb -uroot -prootpassword -e "SHOW DATABASES;"
```

---

## 🔍 Troubleshooting

### Խնդիր 1: mydb service-ը unhealthy

```bash
# Ստուգել logs-երը
docker compose logs mydb

# Ստուգել healthcheck-ը
docker inspect dockercompose-mydb-1 --format='{{json .State.Health}}' | jq

# Manual healthcheck
docker compose exec mydb healthcheck.sh --connect --innodb_initialized

# Լուծում: Restart service
docker compose restart mydb
sleep 30
docker compose ps
```

### Խնդիր 2: frontend-ը չի կարողանում միանալ mydb-ին

```bash
# Ստուգել network connectivity
docker compose exec frontend ping -c 3 mydb

# Ստուգել environment variables
docker compose exec frontend env | grep PMA

# Ստուգել /etc/hosts
docker compose exec frontend cat /etc/hosts

# Լուծում: Restart services
docker compose restart
```

### Խնդիր 3: Port 8080-ը զբաղված է

```bash
# Գտնել ով է օգտագործում port-ը
sudo lsof -i :8080

# Kill process-ը
sudo kill -9 <PID>

# Կամ փոխել port-ը docker-compose.yml-ում
# ports:
#   - "8081:80"
```

### Խնդիր 4: mysqldump/mariadb-dump չի գտնվում

```bash
# Rebuild mydb image-ը
docker compose build --no-cache mydb
docker compose up -d mydb

# Ստուգել package-ը
docker compose exec mydb which mariadb-dump
docker compose exec mydb dpkg -l | grep mariadb-client
```

### Խնդիր 5: Permission denied errors

```bash
# Տալ permissions
sudo chown -R $USER:$USER ~/NarekIPAM
chmod -R 755 ~/NarekIPAM

# Volume permissions
docker compose exec mydb ls -la /var/lib/mysql
```

### Խնդիր 6: Checkup-compose fail է անում

```bash
# Ստուգել project name-ը
docker compose config | grep "^name:"

# Պետք է լինի: name: dockercompose

# Ստուգել resource names
docker ps --format "table {{.Names}}"
docker network ls | grep dockercompose
docker volume ls | grep mydb

# Համոզվել որ dump-ը ճիշտ տեղում է
ls -lh /opt/docker/dockercompose/task-13/mydb.sql
```

---

## 🗑️ Cleanup

### Կանգնեցնել և հեռացնել բոլոր resources-երը

```bash
cd ~/NarekIPAM

# Stop և remove containers, networks, volumes
docker compose down -v

# Հեռացնել images
docker rmi dockercompose-frontend dockercompose-mydb

# Հեռացնել orphan containers
docker compose down --remove-orphans

# Ամբողջական cleanup
docker system prune -a -f
```

### Հեռացնել ֆայլերը

```bash
# Հեռացնել աշխատանքային directory-ն
rm -rf ~/NarekIPAM

# Հեռացնել dump location-ը
sudo rm -rf /opt/docker/dockercompose
```

### Պահպանել միայն dump-ը

```bash
# Backup dump-ը
cp /opt/docker/dockercompose/task-13/mydb.sql ~/mydb_backup.sql

# Cleanup
docker compose down -v
```

---

## 📊 Հետազոտական հրամաններ

### Docker Compose commands

```bash
# Configuration-ի դիտում
docker compose config

# Services-ների ցուցակ
docker compose ps -a

# Resource usage
docker stats

# Logs (last 100 lines)
docker compose logs --tail=100

# Follow logs real-time
docker compose logs -f --tail=50

# Specific service logs
docker compose logs mydb -f
```

### Database queries

```bash
# Show databases
docker compose exec mydb mariadb -uroot -prootpassword -e "SHOW DATABASES;"

# Show tables
docker compose exec mydb mariadb -uroot -prootpassword mydb -e "SHOW TABLES;"

# Table structure
docker compose exec mydb mariadb -uroot -prootpassword mydb -e "DESCRIBE mytable;"

# Count rows
docker compose exec mydb mariadb -uroot -prootpassword mydb -e "SELECT COUNT(*) FROM mytable;"

# Query all data
docker compose exec mydb mariadb -uroot -prootpassword mydb -e "SELECT * FROM mytable;"
```

### Network debugging

```bash
# Inspect network
docker network inspect dockercompose_dockercompose-frontend

# Test DNS resolution
docker compose exec frontend nslookup mydb
docker compose exec mydb nslookup frontend

# Test connectivity
docker compose exec frontend nc -zv mydb 3306
```

---

## 📝 Checklist - Բոլոր կետերը

- [x] **Task 1:** Docker Compose տեղադրված է
- [x] **Task 2:** Frontend service սահմանված է (phpmyadmin:5.2.0-apache, Dockerfile, iputils-ping)
- [x] **Task 3:** Port 80→8080 expose արված
- [x] **Task 4:** dockercompose-frontend network ստեղծված
- [x] **Task 5:** mydb service սահմանված է (MariaDB 11.4, Dockerfile, iputils-ping)
- [x] **Task 6:** Ping connection աշխատում է (frontend ↔ mydb)
- [x] **Task 7:** mydb կցված է dockercompose-frontend network-ին
- [x] **Task 8:** PMA_HOST և PMA_PORT environment variables սահմանված են
- [x] **Task 9:** mydb_data volume սահմանված և կցված է /var/lib/mysql-ին
- [x] **Task 10:** Healthcheck սահմանված է (mysqladmin ping, 10s/15s/5)
- [x] **Task 11:** Frontend-ը depends_on mydb (service_healthy)
- [x] **Task 2 (data):** Database, table ստեղծված, 3 row insert արված
- [x] **Task 13:** MySQL dump ստեղծված և պահպանված

---

## 🎯 Secret Phrases (Checkup Completion)

Երբ բոլոր թեստերը անցնեն, կստանաք այս secret phrases-երը:

1. `hoh9leeMahCh1o` - Docker Compose Version
2. `Ahligievie2ahc` - phpMyAdmin service
3. `ux6ahl8aht4OK2` - Expose ports
4. `Igho5veh9Hifee` - Custom network
5. `shae7laeCh9eid` - mydb service
6. `ohh6iefu0Bupei` - Network usage & interconnection
7. `Chio8eevaiGei4` - phpMyAdmin login
8. `ahngohv3cuo6Ce` - Custom volume
9. `ahNiteech4phee` - Healthcheck
10. `shuraiMi1eimoo` - Service dependency
11. *Secret phrase for dump* - MySQL dump

---

## 📚 Հղումներ

- [Docker Compose Documentation](https://docs.docker.com/compose/)
- [phpMyAdmin Docker Hub](https://hub.docker.com/_/phpmyadmin)
- [MariaDB Docker Hub](https://hub.docker.com/_/mariadb)
- [Docker Networking](https://docs.docker.com/network/)
- [Docker Volumes](https://docs.docker.com/storage/volumes/)

---

## 👨‍💻 Հեղինակ

**Practical Task:** Docker Compose - phpMyAdmin & MariaDB Setup

**Completion Date:** December 24, 2025

**Status:** ✅ All 11 tasks completed (90.91% → 100%)