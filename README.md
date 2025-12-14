# ☁️ Nextcloud AIO on Proxmox LXC with ZFS & Nginx Proxy Manager

This guide provides a robust and scalable method for deploying **Nextcloud All-in-One (AIO)** inside an **unprivileged Proxmox LXC container**, using a **ZFS pool** (or any other large directory) for persistent data storage.

This setup is ideal for homelab users who want secure, easily backed-up, and high-performance cloud storage.

---

## 📋 Prerequisites

This guide assumes:

* A working **Debian 12+ based Proxmox** environment, here I used Debian 13
* Basic familiarity with Linux commands
* An existing **Reverse Proxy** (e.g., **Nginx Proxy Manager** or **Traefik**) ready to configure a new host

---

## 🛠️ Phase 1: LXC and Docker Setup

These steps are performed via the **Proxmox Web UI** and the **LXC console**.

### 1. Create and Prepare the LXC

* Create a new **Debian 12 (or newer)** LXC container (e.g., ID `204`) in the Proxmox Web UI
* Ensure it is configured as an **Unprivileged Container**
* Start the LXC and log in via the console or SSH

Update the system:

```bash
apt update && apt upgrade -y && apt autoremove -y
```

Install necessary utilities:

```bash
apt install curl -y
```

### 2. Install Docker and Docker Compose

Download the official Docker install script:

```bash
curl -fsSL https://get.docker.com -o get-docker.sh
```

Execute the script:

```bash
sh ./get-docker.sh
```

Verify the installation:

```bash
docker compose version
```

---

## 💾 Phase 2: Configure ZFS Storage Mount (Proxmox Host)

Since the LXC is unprivileged, persistent storage must be configured from the **Proxmox host** and permissions handled carefully.

### 1. Shutdown the LXC

* Shut down the Nextcloud LXC from the Proxmox Web UI

### 2. Access the Proxmox Host Shell

* Log in via SSH or Console (e.g., `root@pve`)

### 3. Create the Bind Mount

Append the mount configuration to the LXC config file:

```bash
# IMPORTANT: Replace /path/to/zfs/on/host with the real path to your ZFS dataset or disk mount.
echo 'mp0: /path/to/zfs/on/host,mp=/mnt/ncdata' >> /etc/pve/lxc/204.conf
```

This maps the ZFS dataset into the LXC at `/mnt/ncdata`.

### 4. Set Permissions (Crucial for Unprivileged LXC)

Nextcloud containers require ownership by **UID 33**. This must be set on the **Proxmox host**:

```bash
# Replace /path/to/zfs/on/host with the same path used above
chown -R 33:0 /path/to/zfs/on/host
chmod -R 750 /path/to/zfs/on/host
```

### 5. Restart the LXC

* Start the LXC container from the Proxmox Web UI

---

## 🐳 Phase 3: Deploy Nextcloud AIO with Docker Compose

These steps are performed **inside the running LXC**.

### 1. Create the Compose File


Create the `docker-compose.yml` file:

```bash
nano docker-compose.yml
```

Paste the following configuration:

```yaml
services:
  nextcloud-aio-mastercontainer:
    image: ghcr.io/nextcloud-releases/all-in-one:latest
    init: true
    restart: unless-stopped
    container_name: nextcloud-aio-mastercontainer
    volumes:
      # DO NOT CHANGE: Required for AIO's built-in backup solution
      - nextcloud_aio_mastercontainer:/mnt/docker-aio-config
      # DO NOT CHANGE: Required for AIO to manage other containers
      - /var/run/docker.sock:/var/run/docker.sock:ro
    network_mode: bridge
    ports:
      # Port 8080 for the secure AIO Web Interface
      - 8080:8080
    environment:
      # Internal Apache port for reverse proxy
      APACHE_PORT: 11000
      APACHE_IP_BINDING: "0.0.0.0"
      # ZFS-backed data directory
      NEXTCLOUD_DATADIR: /mnt/ncdata
      NEXTCLOUD_MOUNT: /mnt/
      # Upload limit
      NEXTCLOUD_UPLOAD_LIMIT: 300G
      # Requires reverse proxy to be ready, else true if you do not have a reverse proxy
      SKIP_DOMAIN_VALIDATION: false

volumes:
  nextcloud_aio_mastercontainer:
    name: nextcloud_aio_mastercontainer
```

### 2. Start the Mastercontainer

```bash
docker compose up -d
```

Find the LXC IP address:

```bash
ip a
```

---

## 🌐 Phase 4: Nginx Proxy Manager (NPM) Configuration

### 1. Access the AIO Interface

Open in browser:

```
https://[LXC-IP]:8080
```

* Bypass the self-signed certificate warning
* Record the **one-time master passphrase**
* Click **Open Nextcloud AIO login**

### 2. Configure the Proxy Host in NPM

Create a new **Proxy Host** with:

* **Domain Names:** `cloud.yourdomain.com`
* **Scheme:** `http`
* **Forward Hostname/IP:** IP of your LXC (e.g., `192.168.1.204`)
* **Forward Port:** `11000`
* **Options:** Enable *Block Common Exploits* and *Websocket Support*

#### SSL Settings

* Request a new **Let's Encrypt** certificate
* Enable **Force SSL** and **HTTP/2 Support**

#### Advanced (Custom Nginx Config)

```nginx
client_body_buffer_size 512k;
proxy_read_timeout 86400s;
client_max_body_size 0;
```

Save the Proxy Host.

### 3. Finalize AIO Installation

* Return to the AIO interface
* Enter the exact domain configured in NPM
* Click **Submit**
* Select or ignore optional containers which is completely your choice (Talk, Collabora, Imaginary, etc.)
* Click **Download and start containers**

Wait until all services show a **green status**.

---

## ✅ Phase 5: Verification

### Log in to Nextcloud

Visit:

```
https://cloud.yourdomain.com
```

Log in using the **Admin** credentials provided by the AIO wizard.

### Verify Storage

1. Open **Administration Settings** (gear icon)
2. Go to **Administration → System**
3. Scroll to **Storage**

Confirm that disk size and usage match your ZFS pool mounted at `/mnt/ncdata`.
