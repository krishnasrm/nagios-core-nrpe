# nagios-core-nrpe
# Nagios Monitoring Setup – Step by Step Guide (Server & Client)

This repository contains my step-by-step project on setting up Nagios Core for server and client monitoring. I performed this project as part of my hands-on practice to understand IT monitoring and alerting. All configurations, installations, and tests were done by me, and I’ve documented the process to share my learning.

## 🔹 Project Overview
In this project, I:
1. Installed Nagios Core on a CentOS 9 / RHEL 9 server.
2. Configured the web interface and necessary plugins.
3. Set up NRPE on Linux clients to monitor CPU, memory, disk, and network services.
4. Tested the monitoring configuration using Nagios web interface and NRPE commands.

This project helped me understand real-world monitoring setup, permissions, firewall rules, and remote checks.

---

## 🔹 Server-Side Setup

### 1️⃣ Configure EPEL Repository

I started by adding the EPEL repo because Nagios and its plugins aren’t available in default repositories.

```bash
dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm -y
yum clean all
dnf makecache
yum repolist
yum repolist all
dnf update -y
```

**Why:**

- EPEL repo contains Nagios, NRPE, and extra plugins.
- `yum clean all` clears old metadata.
- `dnf makecache` refreshes repository data.
- `dnf update` ensures system stability before installation.

---

### 2️⃣ Disabled SELinux (Lab Environment Only)

Since this was a practice project, I disabled SELinux to avoid plugin permission issues:

```bash
sed -i 's/SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config
setenforce 0
cat /etc/selinux/config
```

> ⚠️ **Note:**
> Only for lab or practice environments. In production servers, SELinux should not be disabled.
---

### 3️⃣ Install Required Dependencies
I installed all packages required to compile Nagios and run the web interface:

```bash
dnf install -y httpd php php-cli gcc glibc glibc-common gd gd-devel make net-snmp openssl-devel wget unzip perl perl-devel
```

**Explanation:**

- `httpd + php` → Web interface
- `gcc, make` → Compile source code
- `gd` → Graphs & UI support
- `net-snmp` → Network monitoring

---

### 4️⃣ Configure Firewall for HTTP/HTTPS
I allowed HTTP and HTTPS traffic:

```bash
firewall-cmd --permanent --add-service=http
firewall-cmd --permanent --add-service=https
firewall-cmd --reload
```

**Why:** To access Nagios web console via browser.

---

### 5️⃣ Create Nagios User & Group

```bash
useradd nagios
groupadd nagcmd
usermod -a -G nagcmd nagios
usermod -a -G nagcmd apache
```

**Explanation:**

- `nagios` → Runs monitoring and plugins
- `nagcmd` → Executes external commands securely
- Apache added to `nagcmd` → Web UI can run external commands

---

### 6️⃣ Download & Extract Nagios Core

```bash
cd /tmp/
wget https://go.nagios.org/downloads/nagioscore/releases/nagios-4.5.10.tar.gz
tar -xzf nagios-4.5.10.tar.gz
cd nagios-4.5.10
```

---

### 7️⃣ Compile & Install Nagios Core

```bash
./configure --with-command-group=nagcmd
make all
make install
make install-init
make install-commandmode
make install-config
make install-webconf
```

After installation, I verified the Nagios version with `/usr/local/nagios/bin/nagios -v /usr/local/nagios/etc/nagios.cfg`.
 
**Explanation:**

- `configure` → Prepares system-specific build
- `make install-init` → Installs service scripts
- `make install-webconf` → Configures Apache for Nagios

---

### 8️⃣ Create Web Login User

```bash
htpasswd -c /usr/local/nagios/etc/htpasswd.users nagiosadmin
```

> Username: `nagiosadmin`
> Password: `123`
>
> These credentials are for accessing the Nagios web interface.

---

### 9️⃣ Enable & Start Services

```bash
systemctl enable nagios httpd
systemctl start nagios httpd
```

---

### 🔟 Verify Nagios Configuration

```bash
/usr/local/nagios/bin/nagios -v /usr/local/nagios/etc/nagios.cfg
```

This confirmed that Nagios was correctly installed before I proceeded to client setup.

---

### 1️⃣1️⃣ Create a Shortcut for Verification

```bash
vim /usr/bin/verifynagios
```

```bash
#!/bin/bash
/usr/local/nagios/bin/nagios -v /usr/local/nagios/etc/nagios.cfg
```

```bash
chmod +x /usr/bin/verifynagios
verifynagios
```

> This allows quick verification of Nagios configuration with a single command.

---

### 1️⃣2️⃣ Install Nagios Plugins

```bash
cd /tmp/
wget https://nagios-plugins.org/download/nagios-plugins-2.4.11.tar.gz
tar -xzf nagios-plugins-2.4.11.tar.gz
cd nagios-plugins-2.4.11
./configure --with-nagios-user=nagios --with-nagios-group=nagios
make
make install
```

```bash
cd /usr/local/nagios/libexec/
ls
```

> Verify that plugins are installed correctly.

---

### 1️⃣3️⃣ Restrict Web Access to Specific IPs (Nagios – Apache)

#### 🔧 Edit Nagios Apache Configuration

```bash
vim /etc/httpd/conf.d/nagios.conf
```

#### 🔒 Allow Only Specific Client IPs

```apache
<Directory "/usr/local/nagios/sbin">
    #Require all granted
    Require ip 127.0.0.1 192.168.1.40
</Directory>

<Directory "/usr/local/nagios/share">
    #Require all granted
    Require ip 127.0.0.1 192.168.1.40
</Directory>
```

#### 🔁 Restart Apache Service

```bash
systemctl restart httpd
```

#### 🧠 Notes / Explanation
- Require ip is used in Apache 2.4 for IP-based access control.
- Only the mentioned client IPs can access Nagios Web UI.
- 127.0.0.1 → Nagios server itself (localhost).
- 192.168.1.40 → Allowed client machine (Windows).
- Nagios Web UI is always accessed using Nagios server IP:

```
http://<nagios-server-ip>/nagios
```

**Example:**

```
http://192.168.1.38/nagios
```
- Client IP is used only for access restriction, not for monitoring.

#### 🔐 Why This Step
- Prevent unauthorized access to Nagios Web Interface.
- Adds an extra security layer along with username/password.
- Recommended in production environments.

---

### 1️⃣4️⃣ Access Nagios Web Console

```
http://<NagiosServerIP>/nagios
Username: nagiosadmin
Password: 123
```

---

## 🔹 Server-Side Preparation for Client Monitoring

Enable the directory for client configurations:

```bash
vim /usr/local/nagios/etc/nagios.cfg
```

Uncomment:

```bash
cfg_dir=/usr/local/nagios/etc/servers
#cfg_dir=/usr/local/nagios/etc/printers
#cfg_dir=/usr/local/nagios/etc/switches
#cfg_dir=/usr/local/nagios/etc/routers
```

Create directory and set permissions:

```bash
mkdir /usr/local/nagios/etc/servers
chgrp nagios /usr/local/nagios/etc/servers
chmod 750 /usr/local/nagios/etc/servers
```

> Client-specific configuration files will go here.

---

## 🔹 Client Machine Configuration
1. Installed EPEL repo and dependencies.
2. Installed NRPE and all Nagios plugins.
3. Configured firewall to allow port 5666 (NRPE).
4. Edited /etc/nagios/nrpe.cfg to define allowed hosts (Nagios server IP) and monitoring commands.
5. Tested each plugin locally to ensure it worked. They helped me identify a missing plugin early in the process.

### 1️⃣ Install EPEL Repository

```bash
dnf install epel-release -y
dnf clean all
dnf makecache
```

---

### 2️⃣ Install Dependencies & NRPE

```bash
dnf install -y httpd php php-cli gcc glibc glibc-common gd gd-devel make net-snmp openssl-devel wget unzip perl perl-devel
dnf install nrpe nagios-plugins-all -y
```

Verify plugin installation:

```bash
ls -l /usr/lib64/nagios/plugins/
```

Note:- You can create custom plugins if needed in `/usr/lib64/nagios/plugins/`.

---

### 3️⃣ Enable NRPE & Configure Firewall

```bash
systemctl enable nrpe
systemctl start nrpe
firewall-cmd --add-port=5666/tcp --permanent
firewall-cmd --reload
```

---

### 4️⃣ Configure NRPE

```bash
vim /etc/nagios/nrpe.cfg
```

Update important parameters:

```
server_port=5666
allowed_hosts=127.0.0.1,<NagiosServerIP>
```

Define service commands:

```
command[check_users]=/usr/lib64/nagios/plugins/check_users -w 5 -c 10
command[check_load]=/usr/lib64/nagios/plugins/check_load -w 15,10,5 -c 30,25,20
command[check_disk]=/usr/lib64/nagios/plugins/check_disk -w 20% -c 10% -p / -p /var -p /home
command[check_procs]=/usr/lib64/nagios/plugins/check_procs -w 250 -c 400
command[check_ssh]=/usr/lib64/nagios/plugins/check_ssh -H 127.0.0.1 -p 22
command[check_http]=/usr/lib64/nagios/plugins/check_http -I 127.0.0.1
command[check_cpu]=/usr/lib64/nagios/plugins/check_cpu -w 70 -c 90
command[check_memory]=/usr/lib64/nagios/plugins/check_memory -w 70% -c 90%
```

Restart NRPE:

```bash
systemctl restart nrpe
systemctl enable nrpe
systemctl status nrpe
```

> Test each plugin locally first to ensure proper output.

---

## 🔹 Adding Client on Nagios Server

Install **NRPE plugin** on server (no daemon):

```bash
dnf install nagios-plugins-nrpe -y
```

Check plugin:

```bash
ls -l /usr/lib64/nagios/plugins/check_nrpe
```

---

### Define NRPE Command on Server

Edit `command.cfg`:

```bash
vim /usr/local/nagios/etc/objects/command.cfg
```

```cfg
define command{
    command_name check_nrpe
    command_line /usr/lib64/nagios/plugins/check_nrpe -H $HOSTADDRESS$ -c $ARG1$
}
```

---

### Test NRPE from Server

```bash
/usr/lib64/nagios/plugins/check_nrpe -H <client-ip> -c check_cpu
/usr/lib64/nagios/plugins/check_nrpe -H <client-ip> -c check_procs
```

> If output appears, NRPE is working properly.

---

### Define Client Host & Services

Create `/usr/local/nagios/etc/servers/client1.cfg`:

```cfg
define host{
    use             linux-server
    host_name       client1
    alias           Client1 Server
    address         192.168.1.85     #client machine ip
    max_check_attempts 5
    check_period    24x7
    notification_interval 30
    notification_period 24x7
}

define service {
    use                     generic-service
    host_name               client1
    service_description     Logged-in Users
    check_command           check_nrpe!check_users
}

define service {
    use                     generic-service
    host_name               client1
    service_description     Load Average
    check_command           check_nrpe!check_load
}

define service {
    use                     generic-service
    host_name               client1
    service_description     Disk Usage
    check_command           check_nrpe!check_disk
}

define service {
    use                     generic-service
    host_name               client1
    service_description     Processes
    check_command           check_nrpe!check_procs
}

define service {
    use                     generic-service
    host_name               client1
    service_description     SSH
    check_command           check_nrpe!check_ssh
}

define service {
    use                     generic-service
    host_name               client1
    service_description     HTTP
    check_command           check_nrpe!check_http
}

define service {
    use                     generic-service
    host_name               client1
    service_description     CPU Usage
    check_command           check_nrpe!check_cpu
}

define service {
    use                     generic-service
    host_name               client1
    service_description     Memory Usage
    check_command           check_nrpe!check_memory
}

define service {
    use                     generic-service
    host_name               client1
    service_description     Disk Usage Multiple Partitions
    check_command           check_nrpe!check_disk
}
```

---

### 🔹 Final Verification

```bash
verifynagios
systemctl restart nagios
```

Access the Nagios web interface:

```
http://<nagios-server-ip>/nagios
Username: nagiosadmin
Password: 123
```

Check **Hosts & Services** page to ensure all client services are being monitored.

---
## 🔹 My Learnings from This Project
- Setting up Nagios end-to-end taught me Linux permissions, user groups, and service management.
- I learned how to configure remote monitoring using NRPE.
- Testing each step ensured that I could troubleshoot errors like plugin paths or firewall issues.
- I gained experience in writing modular Nagios configurations for multiple clients.
---
