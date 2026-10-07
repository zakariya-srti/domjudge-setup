# DOMjudge Docker Setup Guide

This guide provides step-by-step instructions for setting up DOMjudge using Docker on any operating system.

## Prerequisites

- **Docker installed**: Visit the [Docker documentation](https://docs.docker.com/get-docker/) for installation instructions
- **Docker running**: Verify installation by running the following command:

```bash
docker --version
```

## Setup Instructions

### 1. Create Docker Network

Create a dedicated network for DOMjudge containers:

```bash
docker network create domjudge-net
```

### 2. Setup MariaDB Database

Deploy the database container to store DOMjudge data:

```bash
docker run -d \
  --name dj-mariadb \
  --network domjudge-net \
  -e MYSQL_ROOT_PASSWORD=rootpw \
  -e MYSQL_USER=domjudge \
  -e MYSQL_PASSWORD=djpw \
  -e MYSQL_DATABASE=domjudge \
  -v dj-mariadb-data:/var/lib/mysql \
  mariadb
```

### 3. Run DOMjudge Server

Start the main DOMjudge server container:

```bash
docker run -d \
  --name domserver \
  --network domjudge-net \
  -e MYSQL_HOST=dj-mariadb \
  -e MYSQL_USER=domjudge \
  -e MYSQL_DATABASE=domjudge \
  -e MYSQL_PASSWORD=djpw \
  -e MYSQL_ROOT_PASSWORD=rootpw \
  -e DJ_DB_INSTALL_BARE=1 \
  -e CONTAINER_TIMEZONE=Asia/Karachi \
  -p 12345:80 \
  domjudge/domserver:latest
```

### 4. Run DOMjudge Judgehost

Start the judgehost container:  
Check the judgehost password using:
```bash
docker exec -it domserver cat /opt/domjudge/domserver/etc/restapi.secret
```

It will give something like: 
```bash
# Randomly generated on host b8314ce8dee2, Wed Oct  7 22:55:57 PKT 2026
# Format: '<ID> <API url> <user> <password>'
default http://localhost//api   judgehost       X7xACY+3I4KlCCkxn7BfYiuPSGh1S/LK
```
Copy this password and use it in the JUDGEDAEMON_PASSWORD
```bash
docker run -d \
  --name judgehost-0 \
  --privileged \
  --network domjudge-net \
  -v /sys/fs/cgroup:/sys/fs/cgroup \
  --hostname judgedaemon-0 \
  -e DAEMON_ID=0 \
  -e CONTAINER_TIMEZONE=Asia/Karachi \
  -e DOMSERVER_BASEURL=http://domserver/ \
  -e JUDGEDAEMON_USERNAME=judgehost \
  -e JUDGEDAEMON_PASSWORD="X7xACY+3I4KlCCkxn7BfYiuPSGh1S/LK" \
  domjudge/judgehost:latest
```

### 5. Verify Setup

Access DOMjudge at: `http://localhost:12345`

### 6. Admin Credentials  
username: admin
For password, run the following:
```bash
docker exec -it domserver cat /opt/domjudge/domserver/etc/initial_admin_password.secret
```

## Troubleshooting

### Container Stops with Cgroup Error

If the judgehost container stops, check the logs:

```bash
docker logs judgehost-0
```

**Common Error:**

```
Error: Cgroups not configured properly, missing cgroup hierarchy prefix under /proc/self/cgroup.
If running in a container, make sure to set cgroupns=host.
```

#### Fix for Ubuntu/Linux

1. **Edit GRUB configuration:**

   ```bash
   sudo nano /etc/default/grub
   ```

2. **Find and modify the line:**

   ```bash
   # Change from:
   GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"

   # Change to:
   GRUB_CMDLINE_LINUX_DEFAULT="quiet splash cgroup_enable=memory swapaccount=1"
   ```

   > **Note:** If other options exist, append `cgroup_enable=memory swapaccount=1` inside the quotes.

3. **Update GRUB:**

   ```bash
   sudo update-grub
   ```

4. **Reboot the system:**

   ```bash
   sudo reboot
   ```

5. **After reboot, restart the judgehost:**
   ```bash
   docker run -d \
     --name judgehost-0 \
     --privileged \
     --network domjudge-net \
     -v /sys/fs/cgroup:/sys/fs/cgroup \
     --hostname judgedaemon-0 \
     -e DAEMON_ID=0 \
     -e CONTAINER_TIMEZONE=Asia/Karachi \
     -e DOMSERVER_BASEURL=http://domserver/ \
     -e JUDGEDAEMON_USERNAME=judgehost \
     -e JUDGEDAEMON_PASSWORD="AdminPassword@137" \
     domjudge/judgehost:latest
   ```

#### Fix for Windows

1. **Stop and remove the existing judgehost:**

   ```bash
   docker stop judgehost-0
   docker rm judgehost-0
   ```

2. **Run judgehost with cgroupns=host:**
   ```bash
   docker run -d \
     --name judgehost-0 \
     --privileged \
     --cgroupns=host \
     --network domjudge-net \
     -v /sys/fs/cgroup:/sys/fs/cgroup \
     --hostname judgedaemon-0 \
     -e DAEMON_ID=0 \
     -e CONTAINER_TIMEZONE=Asia/Karachi \
     -e DOMSERVER_BASEURL=http://domserver/ \
     -e JUDGEDAEMON_USERNAME=judgehost \
     -e JUDGEDAEMON_PASSWORD="AdminPassword@137" \
     domjudge/judgehost:latest
   ```

### Credential Issues (Windows)

If you encounter credential errors on Windows:

1. **Get the correct password from domserver:**

   ```bash
   docker exec -it domserver cat /opt/domjudge/domserver/etc/restapi.secret
   ```

   **Example output:**

   ```
   default http://localhost//api judgehost LFO7K2wUGFAq3kTHy6A+EJuFMN8PgnWx
   ```

2. **Copy the password and restart judgehost with correct credentials:**

   ```bash
   docker rm judgehost-0

   docker run -d \
     --name judgehost-0 \
     --privileged \
     --cgroupns=host \
     --network domjudge-net \
     -v /sys/fs/cgroup:/sys/fs/cgroup \
     --hostname judgedaemon-0 \
     -e DAEMON_ID=0 \
     -e CONTAINER_TIMEZONE=Asia/Karachi \
     -e DOMSERVER_BASEURL=http://domserver/ \
     -e JUDGEDAEMON_USERNAME=judgehost \
     -e JUDGEDAEMON_PASSWORD="LFO7K2wUGFAq3kTHy6A+EJuFMN8PgnWx" \
     domjudge/judgehost:latest
   ```

   > **Note:** Replace `LFO7K2wUGFAq3kTHy6A+EJuFMN8PgnWx` with the actual password from step 1.

## Useful Commands

- **Check container status:** `docker ps -a`
- **View container logs:** `docker logs <container-name>`
- **Stop all containers:** `docker stop domserver judgehost-0 dj-mariadb`
- **Remove all containers:** `docker rm domserver judgehost-0 dj-mariadb`
- **Remove network:** `docker network rm domjudge-net`
