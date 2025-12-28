# Browser-Based Linux Desktop on Google Cloud VM (Debian 12 + GPU)

This guide explains how to set up a **full Linux desktop in the browser** for a **Debian 12 GPU VM** on Google Cloud using:

* TigerVNC + XFCE (desktop)
* Docker
* Apache Guacamole
* MariaDB (MySQL) authentication
* SSH tunnel (secure access)

No public ports are exposed. GPU workloads remain unaffected.

---

## 1. System Details

* OS: Debian GNU/Linux 12 (bookworm)
* VM Type: GPU VM (CUDA workloads)
* Access: SSH (`gcloud compute ssh`)
* Goal: Full desktop in browser (no local VNC client)

---

## 2. Install Desktop Environment (XFCE)

XFCE is lightweight and stable for VMs.

```bash
sudo apt update
sudo apt install -y \
  xfce4 xfce4-goodies \
  dbus-x11 \
  x11-xserver-utils
```

---

## 3. Install TigerVNC Server

```bash
sudo apt install -y tigervnc-standalone-server tigervnc-common
```

Set VNC password:

```bash
vncpasswd
```

---

## 4. Configure VNC Startup (Debian 12-compatible)

Create VNC config directory:

```bash
mkdir -p ~/.vnc
```

Create startup file:

```bash
nano ~/.vnc/xstartup
```

Paste **exactly**:

```bash
#!/bin/sh

unset SESSION_MANAGER
unset DBUS_SESSION_BUS_ADDRESS

export XDG_SESSION_TYPE=x11
export XDG_RUNTIME_DIR=/tmp/runtime-$USER
mkdir -p $XDG_RUNTIME_DIR
chmod 700 $XDG_RUNTIME_DIR

exec dbus-run-session xfce4-session
```

Make executable:

```bash
chmod +x ~/.vnc/xstartup
```

---

## 5. Start VNC Server

```bash
vncserver -localhost yes :1
```

This starts VNC on:

```
127.0.0.1:5901
```

Verify:

```bash
ss -ltnp | grep 5901
```

---

## 6. Install Docker (Debian 12)

### Install prerequisites

```bash
sudo apt install -y ca-certificates curl gnupg
```

### Add Docker GPG key

```bash
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg \
 | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
```

### Add Docker repository

```bash
echo \
"deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
https://download.docker.com/linux/debian bookworm stable" \
| sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

### Install Docker Engine + Compose

```bash
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

### Enable Docker and allow user access

```bash
sudo systemctl enable docker
sudo systemctl start docker
sudo usermod -aG docker $USER
newgrp docker
```

Verify:

```bash
docker run hello-world
```

---

## 7. Prepare Apache Guacamole Directories

```bash
mkdir -p ~/guacamole/{mysql,extensions}
cd ~/guacamole
```

---

## 8. Download Guacamole JDBC (MySQL) Authentication Extension

```bash
wget https://downloads.apache.org/guacamole/1.5.5/binary/guacamole-auth-jdbc-1.5.5.tar.gz
tar -xzf guacamole-auth-jdbc-1.5.5.tar.gz
```

Copy MySQL auth extension:

```bash
cp guacamole-auth-jdbc-1.5.5/mysql/guacamole-auth-jdbc-mysql-1.5.5.jar \
   ~/guacamole/extensions/
```

Copy database schema:

```bash
cp guacamole-auth-jdbc-1.5.5/mysql/schema/*.sql \
   ~/guacamole/mysql/
```

---

## 9. Create `docker-compose.yml`

```bash
nano ~/guacamole/docker-compose.yml
```

Paste **entire file**:

```yaml
version: "3.8"

services:
  guacd:
    image: guacamole/guacd
    restart: always
    network_mode: host

  mysql:
    image: mariadb:10.11
    restart: always
    network_mode: host
    environment:
      MYSQL_ROOT_PASSWORD: rootpass
      MYSQL_DATABASE: guacamole_db
      MYSQL_USER: guacamole
      MYSQL_PASSWORD: guacpass
    volumes:
      - ./mysql:/docker-entrypoint-initdb.d
    command: --transaction-isolation=READ-COMMITTED --binlog-format=ROW

  guacamole:
    image: guacamole/guacamole
    restart: always
    network_mode: host
    environment:
      GUACD_HOSTNAME: 127.0.0.1
      MYSQL_HOSTNAME: 127.0.0.1
      MYSQL_DATABASE: guacamole_db
      MYSQL_USER: guacamole
      MYSQL_PASSWORD: guacpass
    volumes:
      - ./extensions:/etc/guacamole/extensions
```

> **Why host networking?**
> VNC runs on the host (`127.0.0.1:5901`). Using `network_mode: host` avoids Docker networking issues.

---

## 10. Start Guacamole Stack

```bash
docker compose up -d
```

Verify:

```bash
docker ps
```

You should see:

* `guacamole-guacamole-1`
* `guacamole-guacd-1`
* `guacamole-mysql-1`

Verify listening ports:

```bash
ss -ltnp | grep -E '4822|5901|3306'
```

---

## 11. Access Guacamole Securely (SSH Tunnel)

From **local machine**:

```bash
gcloud compute ssh bharat-vidya-1 -- -L 8080:localhost:8080
```

Open browser:

```
http://localhost:8080/guacamole
```

Login:

```
Username: guacadmin
Password: guacadmin
```

(Change this password later.)

---

## 12. Create VNC Connection in Guacamole

### GUACD (top section)

```
Hostname: localhost
Port: 4822
```

### Parameters → Network

```
Hostname: 127.0.0.1
Port: 5901
```

### Parameters → Authentication

```
Username: (leave empty)
Password: <your VNC password>
```

### Parameters → Display

```
Color depth: 24
```

### Parameters → Advanced

```
Security mode: Any
Ignore server certificate: enabled
```

Save connection.

---

## 13. Connect 🎉

Click the setup connection →
**XFCE desktop opens inside your browser**

---

## 14. GPU Safety Check

```bash
nvidia-smi
```

✔ Desktop uses virtual display
✔ CUDA training remains unaffected
✔ Safe for ML / TTS workloads

---

## 15. Daily Usage Workflow

1. Start VM
2. Start VNC if needed:

   ```bash
   vncserver -localhost yes :1
   ```
3. Start SSH tunnel:

   ```bash
   gcloud compute ssh bharat-vidya-1 -- -L 8080:localhost:8080
   ```
4. Open browser → Guacamole → Desktop

---

## 16. Optional Next Steps

* Enable audio playback in browser
* Auto-start VNC on boot
* HTTPS for Guacamole
* Launch TensorBoard / VS Code inside desktop
* Resource tuning for GPU workloads

---

## Final Notes

* No public ports exposed
* No local VNC client needed
* Fully browser-based
* Production-grade
* Debian-12-safe

