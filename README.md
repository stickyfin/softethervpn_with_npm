# Self-Hosted SoftEther VPN with Nginx Proxy Manager & Stream Multiplexing

This document outlines the complete, production-ready architecture for running SoftEther VPN alongside Nginx Proxy Manager (NPM) on a single public port `443` using an Nginx Stream Multiplexer (`stream-mux`).

---

## 1. Architectural Overview

Running both Nginx Proxy Manager and SoftEther on port `443` creates a port conflict if handled naively. To solve this, a lightweight Nginx "Front Door" multiplexer (`stream-mux`) inspects raw incoming TLS traffic using `ssl_preread`:
* **VPN Traffic (`vpn.yourdomain.com`)** $\rightarrow$ Forwarded directly to SoftEther.
* **Web/Proxy Traffic (Default)** $\rightarrow$ Forwarded to Nginx Proxy Manager (NPM).

```text
               ┌──────────────────────┐
               │    Public Port 443   │
               └──────────┬───────────┘
                          │
                          ▼
               ┌──────────────────────┐
               │     stream-mux       │
               │  (Nginx Stream 443)  │
               └──┬────────────────┬──┘
                  │                │
     [vpn.yourdomain.com]     [Default / Web]
                  │                │
                  ▼                ▼
         ┌────────────────┐ ┌────────────────┐
         │ SoftEther VPN  │ │  NPM (HTTPS)   │
         └────────────────┘ └────────────────┘
```

---

## 2. Docker Compose Stack

Here is the complete `docker-compose.yml` incorporating the front-door multiplexer, NPM, and SoftEther.

```yaml
version: "3.8"

services:
  # Front-Door Stream Multiplexer (Claims public port 443)
  stream-mux:
    image: nginx:latest
    container_name: stream-mux
    restart: unless-stopped
    ports:
      - "443:443"
    volumes:
      - ./mux.conf:/etc/nginx/nginx.conf:ro
    networks:
      - proxy_network

  # Nginx Proxy Manager (Web UI on 80/81, internal backend on 443)
  npm:
    image: 'jc21/nginx-proxy-manager:latest'
    container_name: npm
    restart: unless-stopped
    ports:
      - "80:80"
      - "81:81"
      # NOTE: Port 443 is omitted here because stream-mux handles it publicly.
    environment:
      DB_SQLITE_FILE: "/data/database.sqlite"
    volumes:
      - ./npm/data:/data
      - ./npm/letsencrypt:/etc/letsencrypt
    networks:
      - proxy_network

  # SoftEther VPN Server
  softether:
    image: softethervpn/vpnserver:latest
    container_name: softether
    restart: unless-stopped
    cap_add:
      - NET_ADMIN
    volumes:
      - ./softether_data:/mnt
    networks:
      - proxy_network

networks:
  proxy_network:
    driver: bridge
```

---

## 3. Multiplexer Configuration (`mux.conf`)

Place this file in the same directory as your `docker-compose.yml`. It uses Docker's internal DNS resolver (`127.0.0.11`) to route traffic dynamically based on the SNI hostname.

```nginx
user nginx;
worker_processes auto;

events {
    worker_connections 1024;
}

stream {
    # Docker's internal DNS resolver
    resolver 127.0.0.11 valid=30s;

    map $ssl_preread_server_name $backend_name {
        vpn.yourdomain.com      softether_backend;
        default                 npm_backend;
    }

    upstream softether_backend {
        server softether:5555;
    }

    upstream npm_backend {
        server npm:443;
    }

    server {
        listen 443;
        listen [::]:443;
        proxy_pass $backend_name;
        ssl_preread on;
    }
}
```

---

## 4. SSL Certificate Synchronization Automation

Because SoftEther embeds certificates directly into its internal configuration (`vpn_server.config`) via `vpncmd`, it does not automatically pick up Let's Encrypt renewals from NPM. 

### Automation Script (`/home/user/softether_renew.sh`)
```bash
#!/bin/bash

# Define absolute paths
NPM_CERT_DIR="/path/to/npm/letsencrypt/live/npm-6"
SOFTETHER_DATA_DIR="/path/to/your/softether_data"
CONTAINER_NAME="softether"

# 1. Copy latest certificates (resolving NPM symlinks)
cp -L "$NPM_CERT_DIR/fullchain.pem" "$SOFTETHER_DATA_DIR/fullchain6.pem"
cp -L "$NPM_CERT_DIR/privkey.pem" "$SOFTETHER_DATA_DIR/privkey6.pem"

# 2. Force SoftEther to import the certificate instantly
docker exec $CONTAINER_NAME vpncmd localhost /SERVER /CMD ServerCertSet /LOADCERT:/mnt/fullchain6.pem /LOADKEY:/mnt/privkey6.pem
```

### Crontab Schedule
To ensure zero certificate expiration downtime, run the script weekly via cron:
```text
0 3 * * 1 /home/user/softether_renew.sh >> /var/log/softether_renew.log 2>&1
```

---

## 5. Essential `vpncmd` Reference Commands

### User Management
* **Create User:**
  ```bash
  docker exec -it softether vpncmd localhost /HUB:DEFAULT
  # Inside vpncmd:
  UserCreate <username>
  UserPasswordSet <username>
  ```
* **Verify User Status & Traffic:**
  ```bash
  UserList
  UserGet <username>
  ```

### Network & Routing (SecureNAT)
* **Enable SecureNAT (Resolves IPCP timeouts):**
  ```bash
  docker exec -it softether vpncmd localhost /HUB:DEFAULT /CMD SecureNatEnable
