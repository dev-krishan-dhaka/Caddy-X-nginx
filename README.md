# 🚀 NGINX & Caddy — Complete Practical Guide

> **Beginner-friendly, production-ready configs for Reverse Proxy, Load Balancer, Forward Proxy, Cache Server, Firewall & HTTPS/SSL**

---

## 📋 Table of Contents

- [Overview](#-overview)
- [Project Structure](#-project-structure)
- [Technologies Used](#-technologies-used)
- [NGINX Guides](#-nginx-guides)
  - [1. Reverse Proxy](#1-nginx-reverse-proxy)
  - [2. Load Balancer](#2-nginx-load-balancer)
  - [3. Forward Proxy](#3-nginx-forward-proxy)
  - [4. Cache Server](#4-nginx-cache-server)
  - [5. Firewall (UFW)](#5-nginx-firewall-ufw)
  - [6. HTTPS / SSL](#6-nginx-https--ssl)
- [Caddy Guides](#-caddy-guides)
  - [7. Reverse Proxy](#7-caddy-reverse-proxy)
  - [8. Load Balancer](#8-caddy-load-balancer)
  - [9. Firewall (UFW)](#9-caddy-firewall-ufw)
  - [10. HTTPS / SSL (Automatic)](#10-caddy-https--ssl-automatic)
- [Final Architecture](#-final-architecture)
- [Learning Summary](#-learning-summary)

---

## 🔍 Overview

This guide covers **two of the most popular web servers/reverse proxies** used in modern DevOps:

| Feature | NGINX | Caddy |
|---|---|---|
| Configuration | `nginx.conf` (verbose) | `Caddyfile` (minimal) |
| HTTPS/SSL | Manual cert setup | **Automatic** (built-in) |
| Performance | Very high | High |
| Learning curve | Moderate | Easy |
| Docker support | ✅ | ✅ |
| Auto HTTPS | ❌ | ✅ |

Both tools support: **Reverse Proxy · Load Balancing · Forward Proxy · Caching · Firewall rules · HTTPS/SSL**

---

## 📁 Project Structure

```
project/
├── docker-compose.yml
├── nginx/
│   └── nginx.conf
├── caddy/
│   └── Caddyfile
├── ssl/
│   ├── nginx.crt
│   └── nginx.key
├── backend1/
│   ├── Dockerfile
│   └── server.js
└── backend2/
    ├── Dockerfile
    └── server.js
```

### Backend Server (shared for all examples)

**`backend1/server.js`**
```js
const express = require("express");
const app = express();

app.get("/", (req, res) => {
  res.send("Hello from Backend 1 🟢");
});

app.listen(3001, () => console.log("Backend1 running on port 3001"));
```

**`backend1/Dockerfile`**
```dockerfile
FROM node:18-alpine
WORKDIR /app
RUN npm install express
COPY server.js .
CMD ["node", "server.js"]
```

> Duplicate `backend1/` as `backend2/` — change port to `3002` and message to `Backend 2`.

---

## 🔧 Technologies Used

- **Docker & Docker Compose** — Container orchestration
- **NGINX** — Web server / reverse proxy
- **Caddy** — Modern web server with auto-HTTPS
- **Express.js** — Minimal Node.js backend
- **UFW** — Linux firewall
- **OpenSSL** — SSL certificate generation

---

---

# 🟦 NGINX Guides

---

## 1. NGINX Reverse Proxy

### What is a Reverse Proxy?

A reverse proxy sits **between the client and your backend servers**. The client never talks to your backend directly.

```
Client  →  NGINX (port 80)  →  Backend Server (port 3001)
```

**Why use it?**
- 🔒 Hides backend servers from the internet
- 🛡️ Adds a security layer
- 🔑 Handles SSL/TLS termination
- ⚖️ Enables load balancing & caching

### nginx/nginx.conf

```nginx
events {}

http {
  server {
    listen 80;

    location / {
      proxy_pass http://backend1:3001;

      # Forward original client info to backend
      proxy_set_header Host              $host;
      proxy_set_header X-Real-IP         $remote_addr;
      proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
    }
  }
}
```

### docker-compose.yml

```yaml
version: '3'

services:
  backend1:
    build: ./backend1

  nginx:
    image: nginx:latest
    ports:
      - "80:80"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf
    depends_on:
      - backend1
```

### Run & Test

```bash
# Start services
docker-compose up --build

# Test
curl http://localhost
# → Hello from Backend 1 🟢
```

### Internal Flow

```
Browser
  ↓  (port 80)
NGINX Reverse Proxy
  ↓  (internal Docker network)
Backend1 (port 3001)
```

---

## 2. NGINX Load Balancer

### What is a Load Balancer?

Distributes incoming traffic **evenly across multiple backend servers** to improve reliability and performance.

```
Request 1  →  Backend1
Request 2  →  Backend2
Request 3  →  Backend1
Request 4  →  Backend2  ...
```

### nginx/nginx.conf

```nginx
events {}

http {
  # Define the group of backend servers
  upstream backend_servers {
    server backend1:3001;
    server backend2:3002;

    # Optional: weighted distribution
    # server backend1:3001 weight=3;
    # server backend2:3002 weight=1;
  }

  server {
    listen 80;

    location / {
      proxy_pass http://backend_servers;

      proxy_set_header Host            $host;
      proxy_set_header X-Real-IP       $remote_addr;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
  }
}
```

### Load Balancing Algorithms

| Method | Config | Description |
|---|---|---|
| Round Robin | *(default)* | Alternate requests evenly |
| Weighted | `weight=N` | Send more traffic to stronger servers |
| Least Connections | `least_conn;` | Send to server with fewest active connections |
| IP Hash | `ip_hash;` | Same client always hits same server |

### docker-compose.yml

```yaml
version: '3'

services:
  backend1:
    build: ./backend1

  backend2:
    build: ./backend2

  nginx:
    image: nginx:latest
    ports:
      - "80:80"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf
    depends_on:
      - backend1
      - backend2
```

### Run & Test

```bash
docker-compose up --build

# Run multiple times to see alternating responses
curl http://localhost   # → Hello from Backend 1 🟢
curl http://localhost   # → Hello from Backend 2 🔵
curl http://localhost   # → Hello from Backend 1 🟢
```

### Internal Flow

```
Client
  ↓
NGINX Load Balancer
  ↙           ↘
Backend1     Backend2
(port 3001)  (port 3002)
```

---

## 3. NGINX Forward Proxy

### What is a Forward Proxy?

A forward proxy sits **in front of the client** and forwards requests to the internet on its behalf.

```
Client  →  Forward Proxy  →  Internet (example.com)
```

**Key difference from Reverse Proxy:**

| | Forward Proxy | Reverse Proxy |
|---|---|---|
| Hides | Client identity | Backend servers |
| Used by | Clients | Server-side |
| Example | VPN / corporate proxy | API gateway |

### nginx/nginx.conf

```nginx
events {}

http {
  server {
    listen 8888;

    # Use Google's DNS resolver
    resolver 8.8.8.8 ipv6=off;

    location / {
      # Forward the full request to the target host
      proxy_pass $scheme://$http_host$request_uri;

      proxy_set_header Host $http_host;
    }
  }
}
```

### docker-compose.yml

```yaml
version: '3'

services:
  nginx:
    image: nginx:latest
    ports:
      - "8888:8888"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf
```

### Run & Test

```bash
docker-compose up --build

# Use NGINX as your proxy to reach external sites
curl -x http://localhost:8888 http://example.com
```

### Internal Flow

```
Your Machine (curl)
  ↓  (proxy: localhost:8888)
NGINX Forward Proxy
  ↓  (outbound internet request)
example.com
  ↓  (response flows back)
Your Machine
```

### ⚠️ Important Security Note

> This is an **open proxy** — anyone who can reach port 8888 can use it.
> For production, always add:
> - IP allowlist/denylist
> - Authentication headers
> - Request logging
> - Rate limiting

---

## 4. NGINX Cache Server

### What is Caching?

NGINX stores backend responses temporarily. Repeated requests are served **from cache** instead of hitting the backend — making responses much faster.

```
1st Request:  Client → NGINX → Backend  (cache MISS — fetch & store)
2nd Request:  Client → NGINX            (cache HIT — instant response)
```

### nginx/nginx.conf

```nginx
events {}

http {
  # Define where and how to cache
  proxy_cache_path /tmp/nginx_cache
    levels=1:2
    keys_zone=my_cache:10m
    max_size=1g
    inactive=60m
    use_temp_path=off;

  upstream backend_servers {
    server backend1:3001;
    server backend2:3002;
  }

  server {
    listen 80;

    location / {
      proxy_cache       my_cache;
      proxy_pass        http://backend_servers;

      # Cache successful responses for 1 minute
      proxy_cache_valid 200 1m;

      # Show cache status in response header
      add_header X-Cache-Status $upstream_cache_status;

      # Avoid caching errors
      proxy_cache_valid 404 1m;
      proxy_cache_use_stale error timeout updating;
    }
  }
}
```

### docker-compose.yml

```yaml
version: '3'

services:
  backend1:
    build: ./backend1

  backend2:
    build: ./backend2

  nginx:
    image: nginx:latest
    ports:
      - "80:80"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf
```

### Run & Test

```bash
docker-compose up --build

# First request — cache MISS (fetched from backend)
curl -i http://localhost
# X-Cache-Status: MISS

# Second request — cache HIT (served from cache ⚡)
curl -i http://localhost
# X-Cache-Status: HIT
```

### Cache Status Values

| Status | Meaning |
|---|---|
| `MISS` | Not in cache — fetched from backend |
| `HIT` | Served from cache |
| `EXPIRED` | Cached but expired — refetched |
| `BYPASS` | Cache bypassed |
| `STALE` | Served stale while backend is slow |

---

## 5. NGINX Firewall (UFW)

### What is a Firewall?

A firewall controls **which network ports and IPs can communicate** with your server.

```
Internet
  ↓  (allowed: 80, 443, 22)
Firewall (UFW)
  ↓
NGINX
  ↓
Backend Containers (internal only — never exposed to internet)
```

### Install & Configure UFW

```bash
# Install UFW
sudo apt update
sudo apt install ufw -y

# Allow SSH (CRITICAL — do this FIRST or you'll lock yourself out)
sudo ufw allow 22

# Allow HTTP and HTTPS
sudo ufw allow 80
sudo ufw allow 443

# Enable the firewall
sudo ufw enable

# Check status
sudo ufw status numbered
```

### UFW Useful Commands

```bash
# Allow a specific IP
sudo ufw allow from 192.168.1.100

# Block a specific IP
sudo ufw deny from 123.45.67.89

# Delete a rule by number
sudo ufw delete 3

# Disable firewall (careful!)
sudo ufw disable

# Reset all rules
sudo ufw reset
```

### ✅ Docker Security — Critical Rule

**NEVER expose backend ports to the host machine.**

```yaml
# ❌ BAD — Backend is reachable from the internet directly!
backend1:
  build: ./backend1
  ports:
    - "3001:3001"   # DON'T DO THIS

# ✅ GOOD — Backend only reachable inside Docker network
backend1:
  build: ./backend1
  # No ports exposed — NGINX reaches it internally
```

> NGINX communicates with backends via Docker's **internal network** using container names (e.g., `http://backend1:3001`). No need to expose backend ports.

---

## 6. NGINX HTTPS / SSL

### What is HTTPS?

HTTPS = **HTTP + Encryption (TLS/SSL)**

All data between the client and server is encrypted — preventing eavesdropping and tampering.

```
Client ←—(encrypted)—→ NGINX (TLS termination) ←—(plain HTTP)—→ Backend
```

### Step 1: Generate a Self-Signed Certificate

```bash
mkdir ssl

openssl req -x509 -nodes -days 365 \
  -newkey rsa:2048 \
  -keyout ssl/nginx.key \
  -out ssl/nginx.crt \
  -subj "/C=US/ST=Dev/L=Local/O=Dev/CN=localhost"
```

> **For production:** Use [Let's Encrypt](https://letsencrypt.org/) (free, auto-renewing) or a paid CA.

### nginx/nginx.conf

```nginx
events {}

http {
  upstream backend_servers {
    server backend1:3001;
    server backend2:3002;
  }

  # Redirect all HTTP → HTTPS
  server {
    listen 80;
    return 301 https://$host$request_uri;
  }

  # HTTPS server
  server {
    listen 443 ssl;

    ssl_certificate     /etc/nginx/ssl/nginx.crt;
    ssl_certificate_key /etc/nginx/ssl/nginx.key;

    # Recommended SSL hardening
    ssl_protocols       TLSv1.2 TLSv1.3;
    ssl_ciphers         HIGH:!aNULL:!MD5;

    location / {
      proxy_pass http://backend_servers;

      proxy_set_header Host            $host;
      proxy_set_header X-Real-IP       $remote_addr;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header X-Forwarded-Proto https;
    }
  }
}
```

### docker-compose.yml

```yaml
version: '3'

services:
  backend1:
    build: ./backend1

  backend2:
    build: ./backend2

  nginx:
    image: nginx:latest
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf
      - ./ssl:/etc/nginx/ssl
    depends_on:
      - backend1
      - backend2
```

### Run & Test

```bash
docker-compose up --build

# Test HTTP → HTTPS redirect
curl -I http://localhost
# HTTP/1.1 301 Moved Permanently

# Test HTTPS (use -k for self-signed cert)
curl -k https://localhost
# → Hello from Backend 1 🟢

curl -k https://localhost
# → Hello from Backend 2 🔵
```

### Internal Flow

```
Client
  ↓  HTTP (port 80)  →  301 redirect
  ↓  HTTPS (port 443)
NGINX (TLS termination)
  ↓  Plain HTTP (internal Docker network)
Backend1 / Backend2
```

---
---

# 🟩 Caddy Guides

> Caddy is a **modern, zero-config web server** written in Go. Its biggest advantage: **automatic HTTPS out of the box**.

---

## 7. Caddy Reverse Proxy

### What Makes Caddy Different?

```
NGINX:  Manual config, manual SSL, ~50 lines for basic setup
Caddy:  3 lines, automatic SSL, done ✅
```

### caddy/Caddyfile

```caddy
localhost {
    reverse_proxy backend1:3001
}
```

### docker-compose.yml

```yaml
version: '3'

services:
  backend1:
    build: ./backend1

  caddy:
    image: caddy:latest
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./caddy/Caddyfile:/etc/caddy/Caddyfile
    depends_on:
      - backend1
```

### Run & Test

```bash
docker-compose up --build

curl -k https://localhost
# → Hello from Backend 1 🟢
```

### Internal Flow

```
Browser
  ↓  (HTTPS — auto-managed by Caddy)
Caddy
  ↓  (Docker internal network)
Backend1 (port 3001)
```

---

## 8. Caddy Load Balancer

### caddy/Caddyfile

```caddy
localhost {
    reverse_proxy backend1:3001 backend2:3002
}
```

That's it. Caddy automatically **round-robins** between backends.

### Advanced Load Balancing Options

```caddy
localhost {
    reverse_proxy backend1:3001 backend2:3002 {
        # Load balancing policy
        lb_policy round_robin        # default
        # lb_policy least_conn
        # lb_policy ip_hash
        # lb_policy random

        # Health checks
        health_uri    /health
        health_interval 10s
    }
}
```

### docker-compose.yml

```yaml
version: '3'

services:
  backend1:
    build: ./backend1

  backend2:
    build: ./backend2

  caddy:
    image: caddy:latest
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./caddy/Caddyfile:/etc/caddy/Caddyfile
    depends_on:
      - backend1
      - backend2
```

### Run & Test

```bash
docker-compose up --build

curl -k https://localhost   # → Hello from Backend 1 🟢
curl -k https://localhost   # → Hello from Backend 2 🔵
curl -k https://localhost   # → Hello from Backend 1 🟢
```

### Internal Flow

```
Client
  ↓
Caddy Load Balancer
  ↙           ↘
Backend1     Backend2
(port 3001)  (port 3002)
```

---

## 9. Caddy Firewall (UFW)

> Firewall setup is the same regardless of whether you use NGINX or Caddy — it operates at the OS level.

### Install & Configure UFW

```bash
# Install UFW
sudo apt update
sudo apt install ufw -y

# Always allow SSH first!
sudo ufw allow 22

# Allow web traffic
sudo ufw allow 80
sudo ufw allow 443

# Enable
sudo ufw enable

# Verify
sudo ufw status numbered
```

### ✅ Docker Security — Do NOT Expose Backend Ports

```yaml
# ❌ BAD — backend is publicly accessible
backend1:
  build: ./backend1
  ports:
    - "3001:3001"

# ✅ GOOD — backend is internal only
backend1:
  build: ./backend1
  # Caddy reaches it via: http://backend1:3001
```

### Complete Secure docker-compose.yml

```yaml
version: '3'

services:
  backend1:
    build: ./backend1
    # No ports exposed — internal only ✅

  backend2:
    build: ./backend2
    # No ports exposed — internal only ✅

  caddy:
    image: caddy:latest
    ports:
      - "80:80"    # Only Caddy is public-facing
      - "443:443"
    volumes:
      - ./caddy/Caddyfile:/etc/caddy/Caddyfile
```

---

## 10. Caddy HTTPS / SSL (Automatic)

### Why Caddy for HTTPS?

| Task | NGINX | Caddy |
|---|---|---|
| Generate certificate | Manual (`openssl`) | **Automatic** |
| Configure SSL in config | ~15 lines | **0 lines** |
| Certificate renewal | Manual / cron | **Automatic** |
| HTTPS redirect | Manual | **Automatic** |

### For Local Development

```caddy
localhost {
    reverse_proxy backend1:3001 backend2:3002
    tls internal
}
```

`tls internal` tells Caddy to use a **locally-trusted certificate** (no external CA needed).

### For Production (Real Domain)

```caddy
example.com {
    reverse_proxy backend1:3001 backend2:3002
}
```

> Just use your real domain name — Caddy **automatically contacts Let's Encrypt**, gets a certificate, and renews it. Zero config required. 🎉

### docker-compose.yml (Production)

```yaml
version: '3'

services:
  backend1:
    build: ./backend1

  backend2:
    build: ./backend2

  caddy:
    image: caddy:latest
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./caddy/Caddyfile:/etc/caddy/Caddyfile
      - caddy_data:/data        # Stores SSL certs persistently
      - caddy_config:/config

volumes:
  caddy_data:
  caddy_config:
```

### Run & Test

```bash
docker-compose up --build

# HTTPS works automatically
curl -k https://localhost
# → Hello from Backend 1 🟢
```

### What Caddy Does Automatically

```
1. Detects domain name in Caddyfile
2. Contacts Let's Encrypt (ACME protocol)
3. Obtains SSL/TLS certificate
4. Configures HTTPS
5. Sets up HTTP → HTTPS redirect
6. Renews certificate before expiry
```

---

## 🏗️ Final Architecture

```
                    Internet
                       │
              ┌────────▼────────┐
              │   UFW Firewall  │
              │  (ports 22,     │
              │   80, 443 only) │
              └────────┬────────┘
                       │
              ┌────────▼────────┐
              │   NGINX / Caddy │
              │  (port 80/443)  │
              │  - Reverse Proxy│
              │  - SSL/TLS      │
              │  - Load Balance │
              │  - Cache        │
              └───────┬─────────┘
                      │
         Docker Internal Network
         ┌────────────┼────────────┐
         │                         │
┌────────▼────────┐     ┌──────────▼──────┐
│    Backend1     │     │    Backend2     │
│   (port 3001)   │     │   (port 3002)   │
│  NOT exposed    │     │  NOT exposed    │
│  to internet    │     │  to internet    │
└─────────────────┘     └─────────────────┘
```

---

## 📚 Learning Summary

### ✅ NGINX — What You Learned

| Topic | Key Concept |
|---|---|
| **Reverse Proxy** | `proxy_pass` — forward requests to backend |
| **Load Balancer** | `upstream` block with multiple servers |
| **Forward Proxy** | `proxy_pass $scheme://$http_host$request_uri` |
| **Cache Server** | `proxy_cache_path` + `proxy_cache` directives |
| **Firewall** | UFW rules + never expose backend ports |
| **HTTPS/SSL** | `ssl_certificate` + `ssl_certificate_key` |

### ✅ Caddy — What You Learned

| Topic | Key Concept |
|---|---|
| **Reverse Proxy** | `reverse_proxy backend1:3001` (1 line!) |
| **Load Balancer** | `reverse_proxy backend1:3001 backend2:3002` |
| **Firewall** | Same UFW rules, never expose backend ports |
| **HTTPS/SSL** | **Zero config** — automatic Let's Encrypt |

### 🔑 Key Security Rules (Always Follow)

1. **Never expose backend ports** — only the proxy (NGINX/Caddy) should be public
2. **Always configure UFW** — block everything except 22, 80, 443
3. **Always redirect HTTP → HTTPS** — never serve plain HTTP in production
4. **Use `proxy_set_header`** — pass real client IP to backends
5. **Keep backends in Docker network** — no direct internet access

---

> 💡 **Tip:** For new projects, prefer **Caddy** for its simplicity and automatic HTTPS.
> Use **NGINX** when you need fine-grained control, advanced caching, or are working with existing infrastructure.
