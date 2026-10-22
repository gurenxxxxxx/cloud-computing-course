# CCM101 - Cloud Computing
**Enterprise Cloud Architect – Operational Manual**

**Project:** Secure Multi-Tier Web Application

**Application Stack:** WordPress + MySQL

**Prepared by:** Felix, Mark Francis J. | Eulalio, Jericho O | Felipe, Frederick O.

**Section:** BSIT 4G

**Instructor:** Jenkielyn C. Torres

**Date:** October 2026

# Overview and Architecture
This project is a self-hosted WordPress website that runs on a single Ubuntu Server virtual machine. WordPress serves the pages and MySQL stores the posts, users, and settings. We chose WordPress + MySQL because it is a common, well-documented stack with official Docker images, and keeping the web and database in separate containers gives the multi-tier design this project requires.

Each layer has one job:

- **Host:** the physical laptop. Its browser and SSH client reach the VM through the NAT port forwards.
- **VirtualBox:** the hypervisor that runs the VM and forwards host ports 8080 and 2222 into it.
- **VM (Ubuntu Server):** the machine that runs everything below.
- **UFW:** the firewall on the VM, allowing only SSH (22) and HTTP (80).
- **Docker / Docker Compose:** runs and restarts the containers from one compose file.
- **Containers:** `ccmfinal_wordpress` (web tier) and `ccmfinal_db` (database tier).
- **Volumes:** named Docker volumes that keep the database and WordPress files when containers are recreated.

<img width="2000" height="1414" alt="architecture-diagram" src="https://github.com/user-attachments/assets/62a36d94-24a4-4aee-9eba-badc18abb201" />


# VM Setup and Networking
**Virtual Machine Specifications**

**Hypervisor:** Oracle VM VirtualBox (Type 2 Hypervisor)

**Operating System:** Ubuntu Server 26.04.1 LTS (Headless distribution)

**Allocated RAM:** 4096 MB (4 GB)

**Allocated CPU Cores:** 2

**Virtual Disk Size:** 25 GB

**Services Installed:** OpenSSH Server for remote administration

**NAT Port Forwarding Configuration**

To allow the host machine to securely interact with the isolated headless virtual machine, Network Address Translation (NAT) port forwarding is configured on the hypervisor:

**Host Port 8080** -> **Guest Port 80 (Web Traffic):** Allows a web browser on the physical host laptop to access the WordPress site via http://localhost:8080.

**Host Port 2222** -> **Guest Port 22 (SSH Administration):** Allows secure terminal management from the host using ssh -p 2222 guren@localhost.

  <img width="593" height="388" alt="vbox-port-forwarding" src="https://github.com/user-attachments/assets/716e54c2-a1f4-4620-b50f-a9c295bb1eb2" />

# Deployment
**Deployment Steps (rebuilding the stack)**

These steps assume Docker and Docker Compose are already installed (see below).

1. Get the project files (`docker-compose.yml`, `.env.example`, `automation-script.sh`) into `~/wp-stack` on the VM.
2. Copy the template and fill in the passwords:
   ```bash
   cd ~/wp-stack
   cp .env.example .env
   nano .env
   ```
3. Start the stack:
   ```bash
   docker compose up -d
   ```
4. Check that both containers are running:
   ```bash
   docker compose ps
   ```
5. Open http://localhost:8080 in the host browser and finish the WordPress setup (site title, admin account).

**Docker Installation Method & Versions**

Docker Engine was installed from Ubuntu’s package repository (docker.io, version 29.1.3, with Ubuntu’s containerd). Docker’s official APT repository was also added, and Docker Compose v2 (docker-compose-plugin, version 5.6.0) and Buildx were installed from it.

<img width="1392" height="115" alt="docker-versions" src="https://github.com/user-attachments/assets/c1129a97-5e52-4a8e-8a34-81f57429b83a" />
<img width="1917" height="155" alt="containers-running" src="https://github.com/user-attachments/assets/313835d0-c941-458c-a535-65b834fe1a00" />

**docker-compose.yml**

```yaml
services:
  db:
    image: mysql:8.0
    container_name: ccmfinal_db
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
    volumes:
      - ccmfinal_mysql_data:/var/lib/mysql

  wordpress:
    image: wordpress:latest
    container_name: ccmfinal_wordpress
    restart: unless-stopped
    depends_on:
      - db
    ports:
      - "80:80"
    environment:
      WORDPRESS_DB_HOST: db:3306
      WORDPRESS_DB_USER: ${MYSQL_USER}
      WORDPRESS_DB_PASSWORD: ${MYSQL_PASSWORD}
      WORDPRESS_DB_NAME: ${MYSQL_DATABASE}
    volumes:
      - ccmfinal_wp_data:/var/www/html

volumes:
  ccmfinal_mysql_data:
  ccmfinal_wp_data:
```

**services:** Declares the multi-container components that make up the stack.

**db:** Names the service block for the database tier.

**image: mysql:8.0:** Pulls the official, stable MySQL version 8.0 image from Docker Hub.

**container_name: ccmfinal_db:** Assigns a predictable container identifier (ccmfinal_db) for logging and monitoring.

**restart: unless-stopped:** Configures Docker to automatically restart the container on system reboots or unexpected crashes, unless manually stopped.

**environment:** Passes secure database initialization variables loaded from the external .env file (MYSQL_ROOT_PASSWORD, MYSQL_DATABASE, MYSQL_USER, MYSQL_PASSWORD).

**volumes:** Mounts a named persistent storage volume (ccmfinal_mysql_data) to MySQL's default data directory (/var/lib/mysql) so database records survive container recreation or system reboots.

**wordpress:** Names the second service block for the web application frontend.

**image: wordpress:latest:** Pulls the official WordPress image containing pre-configured Apache and PHP runtimes.

**container_name: ccmfinal_wordpress:** Sets an explicit container name (ccmfinal_wordpress).

**depends_on:** Controls start order only: Docker Compose starts the db container first and then the wordpress container. It does not wait for MySQL to finish initializing (roughly 30 to 60 seconds), so WordPress may briefly show a database connection error right after startup until MySQL is ready.

**ports:** Maps port 80 inside the container to port 80 on the VM's network interface, exposing the web frontend.

**WORDPRESS_DB_HOST: db:3306:** Directs WordPress to connect to the database service using its internal Docker DNS name (db) on port 3306.

**volumes:** Mounts a separate named volume (ccmfinal_wp_data) to /var/www/html to preserve uploaded media, themes, and configuration changes.

<img width="1917" height="1198" alt="wordpress-site" src="https://github.com/user-attachments/assets/02df4552-2c6a-4c5f-b935-c02fc64217dc" />

**Why MySQL Has No Published Port**

The database service does not feature a ports mapping block. By omitting port publishing, MySQL is completely isolated from the host machine and public network interfaces. It can only be reached by the WordPress container across the internal Docker network (wp-stack_default, named after the project folder, so it changes if the stack is deployed from another folder), minimizing the attack surface.

**Environment Management (.env vs .env.example)**

The .env File: Holds live plain-text secrets and database passwords used at runtime. Because credentials pose a major security risk if exposed in version control, the actual .env file is excluded from GitHub via .gitignore.

The .env.example File: A safe template file containing blank keys or dummy placeholders that is committed to the repository, guiding other administrators on required configuration variables.

# Security Hardening
**Host-Based Firewall (UFW) Configuration**

To secure the Ubuntu Server instance, the Uncomplicated Firewall (UFW) is configured with a strict default deny policy:

```bash
sudo ufw status
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw enable
sudo ufw status verbose
```

- Default Deny Incoming: Rejects all unapproved incoming connection attempts by default.

- Port 22 (SSH): Explicitly allowed to maintain remote administration.

- Port 80 (HTTP): Explicitly allowed to serve web requests to visitors.

**The Docker-and-UFW Caveat**

Docker adds its own firewall rules (through iptables) for every port it publishes, and these rules are applied before UFW's rules. This means UFW does not control published container ports: a port published by Docker is reachable even if UFW would deny it. In this setup, the protection comes from publishing only port 80 and giving MySQL no ports: entry, so nothing else is exposed through Docker.

<img width="1556" height="366" alt="ufw-status" src="https://github.com/user-attachments/assets/63069b52-a9b2-424d-9c14-6167542e01bb" />

# Automation and Disaster Recovery

**Backup Script (automation-script.sh)**

1. It creates the ~/backups folder if it is missing and builds a file name containing the current date and time.
2. It runs mysqldump inside the ccmfinal_db container for the wordpress database and compresses the output with gzip.
3. It prints "Backup OK" or "Backup FAILED" together with the date, and cron saves this output to backup.log. If the backup fails, the script deletes the partial file.
4. It deletes backups older than 7 days.

**Password Security:** The script file contains no password. Docker Compose places MYSQL_ROOT_PASSWORD into the db container from the .env file, and the script reads it from inside the container at runtime.

**Cron Job Scheduling**

The backup script is automated to execute daily at 2:00 AM Philippine Standard Time (Asia/Manila).

  - **Cron Expression:**

    ```
    0 2 * * * /home/guren/wp-stack/automation-script.sh >> /home/guren/backups/backup.log 2>&1
    ```

  - **Field Breakdown:**
    - `0` — Minute 0
    - `2` — Hour 2 (2:00 AM)
    - `*` — Every day of the month
    - `*` — Every month
    - `*` — Every day of the week

If the VM timezone is changed after the cron job is created (for example, to Asia/Manila), restart the cron service once so it uses the new timezone:

```bash
sudo systemctl restart cron
```

**Backup Retention & Restoration**

  - Storage & Retention: Backups are saved in ~/backups/ and automatically pruned after 7 days using find ... -mtime +7 -delete.

  - Restoration Command: To restore a database snapshot from a compressed backup file (this replaces the current database with the backup's contents):

```bash
gunzip -c ~/backups/wordpress_2026-10-21_12-27-54.sql.gz | docker exec -i ccmfinal_db sh -c 'exec mysql -u root -p"$MYSQL_ROOT_PASSWORD" wordpress'
```

<img width="1865" height="568" alt="backup-proof" src="https://github.com/user-attachments/assets/4128ab56-a964-420d-ad70-c33f2ed7211f" />
<img width="1462" height="605" alt="crontab" src="https://github.com/user-attachments/assets/adaf7f9a-2868-4b8c-bbfc-8b706a28d85d" />


# Verification and Persistence Test
<img width="1917" height="1125" alt="reboot-test" src="https://github.com/user-attachments/assets/709da6f7-0ac5-4093-983e-b9bec6d72fe4" />

**Reboot Persistence Proof**

Executing a system reboot (sudo reboot) and validating the environment shows that:

1. Containers restart automatically under the unless-stopped policy.

2. Host UFW security firewall rules remain active.

3. The WordPress site loaded normally after the reboot.

# Maintenance and Troubleshooting
**Daily Operational Commands** (run from `~/wp-stack`, the folder containing the compose file)

- Check Status: docker compose ps

- View Container Logs: docker compose logs -f

- Stop Stack: docker compose down

- Start Stack: docker compose up -d

- ⚠️ CRITICAL WARNING: Never execute docker compose down -v. The -v flag deletes named volumes, permanently erasing all database records and uploaded files.

**Troubleshooting Reference Table**

| Issue Encountered | Root Cause | Solution / Fix |
| :--- | :--- | :--- |
| **.env not found error** | Missing configuration file during stack initialization. Symptoms: "variable is not set" warnings, and WordPress showing "Error establishing a database connection." | Copy `.env.example` to `.env` and fill in required database parameters. |
| **Docker permission denied** | User account lacks privileges to access the Docker Unix socket. | Add user to the docker group: `sudo usermod -aG docker $USER && newgrp docker`. |
| **SSH server not installed** | The OpenSSH server option was not ticked in the Ubuntu installer. | Install package manually: `sudo apt update && sudo apt install openssh-server -y`. |
| **Installation freeze (CPU soft lockup)** | The virtual CPU stopped responding during the install; the console showed `watchdog: BUG: soft lockup - CPU stuck`. | [FILL IN: the VirtualBox settings you changed, e.g. Settings > System > ...] |
