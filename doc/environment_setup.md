
# SailPoint IdentityIQ 8.3 Installation Guide on Red Hat Enterprise Linux (Personal Lab)

**Version:** IdentityIQ 8.3  
**OS:** Red Hat Enterprise Linux 8.8 or 9.1  
**Application Server:** Apache Tomcat 9.0.89  
**JDK:** OpenJDK 21 (Red Hat certified)  
**Database:** MySQL 8.0 (example; supports PostgreSQL 13, Oracle 19c, MS SQL 2019)  

**Warnings:**
- Backup data before proceeding.
- Ensure firewall allows ports 8080 (Tomcat) and 3306 (MySQL).
- Run as non-root user (`sailpoint`) with sudo privileges.
- Download JDBC drivers from your DB vendor (not included with IdentityIQ).

## Table of Contents
- [SailPoint IdentityIQ 8.3 Installation Guide on Red Hat Enterprise Linux (Personal Lab)](#sailpoint-identityiq-83-installation-guide-on-red-hat-enterprise-linux-personal-lab)
  - [Table of Contents](#table-of-contents)
  - [Prerequisites](#prerequisites)
    - [Hardware Requirements](#hardware-requirements)
    - [Software Requirements](#software-requirements)
  - [Step 1: Prepare the Environment](#step-1-prepare-the-environment)
  - [Step 2: Install OpenJDK 21](#step-2-install-openjdk-21)
  - [Step 3: Install and Configure Apache Tomcat](#step-3-install-and-configure-apache-tomcat)
  - [Step 4: Install and Configure Database (MySQL Example)](#step-4-install-and-configure-database-mysql-example)
  - [Step 5: Download and Extract IdentityIQ 8.3](#step-5-download-and-extract-identityiq-83)
  - [Step 6: Deploy IdentityIQ WAR File](#step-6-deploy-identityiq-war-file)
  - [Step 7: Configure IdentityIQ Properties](#step-7-configure-identityiq-properties)
  - [Step 8: Initialize the Database Schema](#step-8-initialize-the-database-schema)
  - [Step 9: Start Tomcat and Complete Initial Setup](#step-9-start-tomcat-and-complete-initial-setup)
  - [Step 10: Post-Installation Tasks](#step-10-post-installation-tasks)
  - [Troubleshooting](#troubleshooting)
  - [Verification](#verification)
  - [References](#references)
  - [NEXT: IIQ User Provisioning Using FreeIPA](#next-step)


## Prerequisites

### Hardware Requirements
- **CPU:** 4 cores (8+ for production).
- **RAM:** 8 GB (16 GB+ recommended).
- **Disk:** 50 GB+ free (SSD preferred).
- **Network:** Static IP, firewall configured.

### Software Requirements
| Component | Supported Versions | Notes |
|-----------|--------------------|-------|
| OS | RHEL 8.8, 9.1 | Use `dnf` for package management. |
| JDK | OpenJDK 21 | Red Hat certified; Eclipse Temurin supported. |
| Application Server | Apache Tomcat 9.0.89 | Single instance for this guide. |
| Database | MySQL 8.0, PostgreSQL 13, Oracle 19c, MS SQL 2019 | MySQL used in this example. |
| Browser | Chrome, Firefox, Edge (latest) | For UI access. |

- **Access:** SailPoint Compass account for downloads.
- **Tools:** Install `wget`, `tar`, `unzip` via `sudo dnf install wget tar unzip`.
- **User:** Non-root user (`sailpoint`) with sudo access.

## Step 1: Prepare the Environment
1. Update system:
   ```bash
   sudo dnf update -y
   sudo dnf upgrade -y
   ```
2. Create user:
   ```bash
   sudo useradd -m -s /bin/bash sailpoint
   sudo passwd sailpoint
   sudo usermod -aG wheel sailpoint
   su - sailpoint
   ```
3. Configure firewall:
   ```bash
   sudo firewall-cmd --permanent --add-port=8080/tcp
   sudo firewall-cmd --permanent --add-port=8443/tcp
   sudo firewall-cmd --permanent --add-port=3306/tcp
   sudo firewall-cmd --reload
   ```
4. Set SELinux to permissive (if needed):
   ```bash
   sudo setenforce 0
   sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
   ```

## Step 2: Install OpenJDK 21
1. Remove existing JDKs:
   ```bash
   sudo dnf remove -y java-*
   sudo rm -rf /opt/jdk/*
   sed -i '/JAVA_HOME/d' ~/.bashrc
   ```
2. Install OpenJDK 21:
   ```bash
   sudo dnf install -y java-21-openjdk-devel
   ```
3. Verify:
   ```bash
   java -version
   ```
   Expected: `openjdk version "21.0.x" ...`
4. Set `JAVA_HOME`:
   ```bash
   export JAVA_HOME=/usr/lib/jvm/java-21-openjdk
   echo 'export JAVA_HOME=/usr/lib/jvm/java-21-openjdk' >> ~/.bashrc
   source ~/.bashrc
   ```

## Step 3: Install and Configure Apache Tomcat
1. Download and extract Tomcat 9.0.89:
   ```bash
   sudo mkdir -p /opt/tomcat
   cd /opt/tomcat
   sudo wget https://archive.apache.org/dist/tomcat/tomcat-9/v9.0.89/bin/apache-tomcat-9.0.89.tar.gz
   sudo tar xzvf apache-tomcat-9.0.89.tar.gz
   sudo mv apache-tomcat-9.0.89/* /opt/tomcat/
   sudo rm -rf apache-tomcat-9.0.89 apache-tomcat-9.0.89.tar.gz
   ```
2. Set permissions:
   ```bash
   sudo chown -R sailpoint:sailpoint /opt/tomcat
   sudo chmod -R 755 /opt/tomcat
   ```
3. Configure Tomcat users (`/opt/tomcat/conf/tomcat-users.xml`):
   Add before `</tomcat-users>`:
   ```xml
   <role rolename="manager-gui"/>
   <role rolename="admin-gui"/>
   <user username="admin" password="changeme" roles="admin-gui,manager-gui"/>
   ```
4. Create `setenv.sh`:
   ```bash
   vi /opt/tomcat/bin/setenv.sh
   ```
   Add:
   ```bash
   export JAVA_HOME=/usr/lib/jvm/java-21-openjdk
   export CATALINA_OPTS="-Xms2g -Xmx4g -XX:+UseG1GC --add-exports=java.naming/com.sun.jndi.ldap=ALL-UNNAMED --add-exports=java.base/sun.security.x509=ALL-UNNAMED"
   ```
   Set permissions:
   ```bash
   chmod +x /opt/tomcat/bin/setenv.sh
   sudo chown sailpoint:sailpoint /opt/tomcat/bin/setenv.sh
   ```
5. Create systemd service (`/etc/systemd/system/tomcat.service`):
   ```bash
   sudo vi /etc/systemd/system/tomcat.service
   ```
   Paste:
   ```
   [Unit]
   Description=Apache Tomcat Web Application Container
   After=network.target

   [Service]
   Type=forking
   Environment="JAVA_HOME=/usr/lib/jvm/java-21-openjdk"
   Environment="CATALINA_PID=/opt/tomcat/temp/tomcat.pid"
   Environment="CATALINA_HOME=/opt/tomcat"
   Environment="CATALINA_BASE=/opt/tomcat"
   ExecStart=/opt/tomcat/bin/startup.sh
   ExecStop=/opt/tomcat/bin/shutdown.sh
   User=sailpoint
   Group=sailpoint
   RestartSec=10
   Restart=always

   [Install]
   WantedBy=multi-user.target
   ```
6. Enable and start:
   ```bash
   sudo systemctl daemon-reload
   sudo systemctl enable tomcat
   sudo systemctl start tomcat
   ```

## Step 4: Install and Configure Database (MySQL Example)
1. Add MySQL repo (`/etc/yum.repos.d/mysql.repo`):
   ```bash
   sudo vi /etc/yum.repos.d/mysql.repo
   ```
   Paste:
   ```
   [mysql80-community]
   name=MySQL 8.0 Community Server
   baseurl=https://repo.mysql.com/yum/mysql-8.0-community/el/9/$basearch/
   enabled=1
   gpgcheck=1
   gpgkey=https://repo.mysql.com/RPM-GPG-KEY-mysql
   ```
2. Install MySQL:
   ```bash
   sudo dnf install -y mysql-community-server
   sudo systemctl start mysqld
   sudo systemctl enable mysqld
   ```
3. Secure installation:
   ```bash
   sudo mysql_secure_installation
   ```
4. Create database and user:
   ```bash
   mysql -u root -p
   ```
   In MySQL shell:
   ```sql
   CREATE DATABASE identityiq CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
   CREATE USER 'iiquser'@'localhost' IDENTIFIED BY 'strongpassword';
   GRANT ALL PRIVILEGES ON identityiq.* TO 'iiquser'@'localhost';
   FLUSH PRIVILEGES;
   EXIT;
   ```
5. Download MySQL JDBC driver (mysql-connector-j-8.0.x.jar) from [MySQL](https://dev.mysql.com/downloads/connector/j/) and place in Tomcat:
   ```bash
   sudo cp mysql-connector-j-8.0.x.jar /opt/tomcat/lib/
   sudo chown sailpoint:sailpoint /opt/tomcat/lib/mysql-connector-j-8.0.x.jar
   ```
6. Configure MySQL (`/etc/my.cnf`):
   ```bash
   sudo vi /etc/my.cnf
   ```
   Add:
   ```
   [mysqld]
   innodb_file_per_table=1
   sql_mode=''
   ```
   Restart:
   ```bash
   sudo systemctl restart mysqld
   ```

## Step 5: Download and Extract IdentityIQ 8.3
1. Download `identityiq-8.3.zip` from [SailPoint Compass](https://community.sailpoint.com).
2. Extract and copy WAR:
   ```bash
   cd /tmp
   unzip identityiq-8.3.zip
   sudo cp identityiq.war /opt/tomcat/webapps/
   sudo chown sailpoint:sailpoint /opt/tomcat/webapps/identityiq.war
   sudo chmod 644 /opt/tomcat/webapps/identityiq.war
   ```
3. Verify deployment:
   ```bash
   sudo systemctl restart tomcat
   ls -l /opt/tomcat/webapps/identityiq/
   ```

## Step 6: Deploy IdentityIQ WAR File
- The WAR auto-deploys to `/opt/tomcat/webapps/identityiq/` on Tomcat restart.
- For manual deployment: Stop Tomcat, delete old dir, copy WAR, restart Tomcat.

## Step 7: Configure IdentityIQ Properties
1. Ensure Tomcat is running:
   ```bash
   sudo systemctl status tomcat
   ls -l /opt/tomcat/webapps/identityiq/
   ```
2. Edit `iiq.properties`:
   ```bash
   vi /opt/tomcat/webapps/identityiq/WEB-INF/classes/iiq.properties
   ```
   Add:
   ```
   # Database configuration
   datastore.driverClassName=com.mysql.cj.jdbc.Driver
   datastore.url=jdbc:mysql://localhost:3306/identityiq?useUnicode=true&characterEncoding=UTF-8&useSSL=false&serverTimezone=UTC
   datastore.username=iiquser
   datastore.password=strongpassword
   datastore.dialect=net.sf.hibernate.dialect.MySQL5InnoDBDialect

   # Default settings
   password_policy.default=Default
   web.sso.enabled=false
   ```
3. Restart Tomcat:
   ```bash
   sudo systemctl restart tomcat
   ```

## Step 8: Initialize the Database Schema
1. Navigate to bin:
   ```bash
   cd /opt/tomcat/webapps/identityiq/WEB-INF/bin
   chmod +x iiq
   ```
2. Generate schema:
   ```bash
   ./iiq schema -database mysql
   ```
3. Run schema:
   ```bash
   mysql -u iiquser -p identityiq < /opt/tomcat/webapps/identityiq/WEB-INF/database/create_identityiq_tables-8.3.mysql
   ```
4. Import initial data:
   ```bash
   ./iiq console
   ```
   In console:
   ```
   import init.xml
   Init-lcm.xml
   Init-pam.xml
   ```

## Step 9: Start Tomcat and Complete Initial Setup
1. Start Tomcat:
   ```bash
   sudo systemctl start tomcat
   sudo systemctl status tomcat
   ```
2. Access UI: `http://your-server-ip:8080/identityiq`
3. Complete setup wizard:
   - Accept license.
   - Set admin user (default: spadmin/admin).
   - Run diagnostics.
4. Log in Dashboard:

   
   <img width="468" height="251" alt="image" src="https://github.com/user-attachments/assets/6086ecca-dde2-4773-b4fa-54fea0f0b2ca" />


## Step 10: Post-Installation Tasks
- **Connectors:** Deploy from Compass to `/opt/tomcat/webapps/identityiq/WEB-INF/lib/`.
- **Logging:** Edit `/opt/tomcat/webapps/identityiq/WEB-INF/log4j2.xml`.
- **HTTPS:** Configure SSL in `/opt/tomcat/conf/server.xml`.
- **Backup:** Script DB dumps and configs.
- **Test:** Run lifecycle tasks, access requests.

## Troubleshooting
- **Tomcat Fails:** Check `/opt/tomcat/logs/catalina.out` for JDK/DB errors.
- **DB Issues:** Verify JDBC driver, credentials, firewall.
- **Schema Errors:** Check MySQL modes; rerun DDL.
- **Memory Errors:** Adjust `CATALINA_OPTS` in `setenv.sh`.
- See [Release Notes](https://community.sailpoint.com) for 8.3 patches.

## Verification
- UI loads at `http://your-server-ip:8080/identityiq`.
- Admin login works.
- Run `./iiq version`: Shows 8.3.
- Check DB: `mysql> SHOW TABLES;` (~200 tables).
- 

## References
- [IdentityIQ 8.3 Installation Guide](https://community.sailpoint.com)
- [Tomcat 9 Downloads](https://tomcat.apache.org/download-90.cgi)
- [MySQL Connector/J](https://dev.mysql.com/downloads/connector/j/)
- [SailPoint Docs](https://documentation.sailpoint.com/identityiq)

## NEXT: IIQ User Provisioning Using FreeIPA
Open freeipa_deployment.md for database, Ldap and Kerberos Integration on iiq.
