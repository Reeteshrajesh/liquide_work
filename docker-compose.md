# 🚀 Jitsu HTTPS Deployment on EC2 (`reetesh.uttam.live`)

This setup runs a secure, production-ready deployment of [Jitsu](https://jitsu.com) on an **Amazon EC2 instance** using:

* ✅ Docker + Docker Compose
* ✅ NGINX as a reverse proxy
* ✅ PostgreSQL + Redis
* ✅ Let's Encrypt SSL for HTTPS
* ✅ Domain: `https://reetesh.uttam.live`

---

## 📁 Project Structure

```
jitsu-https/
├── docker-compose.yaml
├── nginx.conf
├── jitsu-ssl/             # Contains your SSL certificate and private key
│   ├── fullchain.pem
│   └── privkey.pem
```

---

## 📦 Prerequisites

1. **Domain setup**: `reetesh.uttam.live` must point to your EC2's public IP (via Route 53 A-record).
2. **Ports open**: Ensure **22, 80, and 443** are allowed in your EC2 Security Group.
3. **OS**: Ubuntu 20.04 or newer (Amazon Linux 2 also works with minor tweaks).

---

## 🛠️ Step-by-Step Setup

### ✅ 1. SSH into EC2

```bash
ssh -i your-key.pem ubuntu@<your-ec2-ip>
```

### ✅ 2. Install Docker + Certbot

```bash
sudo apt update && sudo apt install -y docker.io docker-compose nginx certbot python3-certbot-nginx
sudo systemctl start docker && sudo systemctl enable docker
sudo usermod -aG docker $USER
```

**Important**: Logout and re-login so Docker group takes effect.

---

### ✅ 3. Clone or Create the Project Folder

```bash
mkdir -p ~/jitsu-https/jitsu-ssl
cd ~/jitsu-https
```

---

### ✅ 4. `docker-compose.yaml`

```yaml
version: '3.7'
services:
  postgres:
    image: postgres:13
    environment:
      POSTGRES_USER: jitsu
      POSTGRES_PASSWORD: jitsu
      POSTGRES_DB: jitsu
    volumes:
      - pgdata:/var/lib/postgresql/data
    restart: unless-stopped

  redis:
    image: redis:6
    restart: unless-stopped

  jitsu:
    image: jitsucom/jitsu:latest
    environment:
      - REDIS_URL=redis://redis:6379
      - CONFIGURATOR_META_STORAGE=postgres
      - CONFIGURATOR_META_DSN=postgres://jitsu:jitsu@postgres:5432/jitsu?sslmode=disable
      - EVENTNATIVE_META_STORAGE=postgres
      - EVENTNATIVE_META_DSN=postgres://jitsu:jitsu@postgres:5432/jitsu?sslmode=disable
    expose:
      - "8000"
    depends_on:
      - postgres
      - redis
    restart: unless-stopped

  nginx:
    image: nginx:latest
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./jitsu-ssl:/etc/nginx/certs:ro
    depends_on:
      - jitsu
    restart: unless-stopped

volumes:
  pgdata:
```

---

### ✅ 5. `nginx.conf`

```nginx
events {}

http {
  server {
    listen 80;
    server_name reetesh.uttam.live;

    location / {
      return 301 https://$host$request_uri;
    }
  }

  server {
    listen 443 ssl;
    server_name reetesh.uttam.live;    // jisake arround hum ssl certificate lete hai 

    ssl_certificate     /etc/nginx/certs/fullchain.pem;
    ssl_certificate_key /etc/nginx/certs/privkey.pem;

    location / {
      proxy_pass http://jitsu:8000;
      proxy_set_header Host $host;
      proxy_set_header X-Real-IP $remote_addr;
    }
  }
}
```

---

### ✅ 6. Run Jitsu (initially without SSL)

```bash
docker compose up -d
```

Wait for containers to start.

---

### ✅ 7. Stop NGINX and Generate SSL

```bash
docker compose stop nginx
sudo certbot certonly --standalone -d reetesh.uttam.live
```

Copy certificates:

```bash
sudo cp /etc/letsencrypt/live/reetesh.uttam.live/fullchain.pem ./jitsu-ssl/
sudo cp /etc/letsencrypt/live/reetesh.uttam.live/privkey.pem ./jitsu-ssl/
```

---

### ✅ 8. Restart NGINX with SSL

```bash
docker compose up -d nginx
```

---

### ✅ 9. Access Jitsu UI

Go to:
👉 [https://reetesh.uttam.live](https://reetesh.uttam.live)
You should see the Jitsu UI with **SSL lock 🔒**

---

## 🔁 Auto-Renew SSL (Every 60 Days)

Let’s Encrypt certificates expire in 90 days.

Test renewal:

```bash
sudo certbot renew --dry-run
```

To automate:

```bash
sudo crontab -e
```

Add:

```cron
0 3 * * * certbot renew --post-hook "docker compose -f /home/ubuntu/jitsu-https/docker-compose.yaml restart nginx"
```

---

## 🧹 Tips

* Use a `docker-compose.override.yaml` for local testing if needed.
* Keep your DNS pointed correctly at all times or renewal will fail.
* Logs:

  ```bash
  docker compose logs -f jitsu
  ```

---

## ✅ Final Checklist

* [x] EC2 instance running
* [x] Docker + Docker Compose installed
* [x] Domain `reetesh.uttam.live` resolves to EC2
* [x] Jitsu accessible via HTTPS
* [x] Certbot auto-renewal in place


-----
-----
-----

## ✅ What the Setup Already Covers (and why it's enough for MVP/production):

| Feature                      | Status    | Details                                            |
| ---------------------------- | --------- | -------------------------------------------------- |
| **Public domain with HTTPS** | ✅ Done    | Route53 + Certbot for real SSL (Let’s Encrypt)     |
| **Secure Reverse Proxy**     | ✅ Done    | NGINX with real SSL cert, automatic HTTP→HTTPS     |
| **Persistence**              | ✅ Done    | PostgreSQL + Redis with Docker volumes             |
| **Container orchestration**  | ✅ Basic   | Docker Compose (suitable for single-node prod)     |
| **Restart on failure**       | ✅ Done    | `restart: unless-stopped` in `docker-compose.yaml` |
| **Cert auto-renewal**        | ✅ Covered | Cron job with `certbot renew` + nginx reload       |
| **DNS correctly mapped**     | ✅ Done    | You control the domain in Route 53                 |

This setup will **work reliably** for production workloads, even for **millions of events per month**.



## ✅ When This Setup Is 100% Sufficient

This setup is **perfectly sufficient** if:

* You're running **a single EC2 instance** (dedicated for Jitsu).
* You own the domain and have set up **DNS via Route 53**.
* You're okay with **Docker Compose** instead of Kubernetes (totally fine for 1-node deployments).
* You just want to **ingest events, test destinations, and manage sources with the UI**.

----
----
| Daily Events  | Recommended Setup                                                                   |
| ------------- | ----------------------------------------------------------------------------------- |
| < 5 million   | Single `t3.medium` with local Redis + PostgreSQL                                    |
| 5–20 million  | `t3.large` or `m5.large`, maybe external Redis                                      |
| 20–50 million | Separate Redis/PostgreSQL, `m5.xlarge`, SSD storage                                 |
| 50M+          | Multi-node setup (Docker Swarm, Kubernetes, or ECS) with Kafka or queueing strategy |

----
---


## 🏗️ Architecture Overview

            ┌────────────────────────────────────────────────────┐
            │                  Route 53 (DNS)                    │
            │  Domain: reetesh.uttam.live → EC2 Public IP     │
            └────────────────────────────────────────────────────┘
                                 │
                                 ▼
                   ┌────────────────────────────┐
                   │     EC2 Instance (Ubuntu)   │
                   │ ─────────────────────────── │
                   │  Docker + docker-compose    │
                   │  Docker Volumes             │
                   │  NGINX (SSL termination)    │
                   │  Jitsu (Data + UI)          │
                   │  PostgreSQL (metadata)      │
                   │  Redis (event cache)        │
                   └────────────────────────────┘
                                 │
            ┌────────────────────┼───────────────────---------------─┐
            ▼                    ▼                                   ▼
      Destinations           Web UI (port 443)                   REST API / SDK
   (Amplitude, S3, etc.)       https://reetesh.uttam.live       JS SDK / HTTP





## ⚙️ Working Style – Step-by-Step Flow

### 1. **DNS and SSL (HTTPS Access)**

* You use **Route 53** to point the domain `reetesh.uttam.live` to your EC2 public IP.
* You install **NGINX** to serve as a **reverse proxy with SSL termination** using a valid certificate (from Let's Encrypt or custom CA).
* Users and scripts access Jitsu via `https://reetesh.uttam.live`.

---

### 2. **Incoming Events**

* Website, SDK, or backend sends events via:

  * `https://reetesh.uttam.live/api/v1/event` (standard HTTP event ingestion)
  * or via Jitsu’s JS snippet loaded on your frontend

---

### 3. **NGINX Reverse Proxy**

* NGINX listens on **port 443 (HTTPS)**.
* Forwards traffic to the `jitsu` container (on port 8000 internally).
* It ensures all public-facing traffic is encrypted.

---

### 4. **Jitsu Core (Docker Container)**

* **Jitsu receives events** and:

  * Validates schema and request.
  * Optionally enriches or transforms the data.
  * Uses **Redis** to deduplicate or temporarily cache some event metadata.
  * Sends events to configured **destinations** (e.g., Amplitude, S3, BigQuery, Redshift, etc.).
* **Jitsu UI** is served from the same container on port `8000`, routed by NGINX.
* You manage source/destination config via the web UI.

---

### 5. **PostgreSQL**

* Stores:

  * Destination config
  * Project metadata
  * User accounts
* Data is stored persistently using a Docker volume or an external RDS instance if scaled further.

---

### 6. **Redis**

* Used for:

  * Event caching
  * Deduplication
  * Reducing pressure on PostgreSQL
* A lightweight, fast key-value store ideal for this task.

---

## 🛡️ Security & Production Hardening

| Component  | Hardening Applied                               |
| ---------- | ----------------------------------------------- |
| NGINX      | HTTPS with a valid SSL certificate              |
| Domain     | Routed through Route 53 to EC2 public IP        |
| Firewall   | Only port `443` (HTTPS) exposed to public       |
| PostgreSQL | Internal-only, not publicly accessible          |
| Redis      | Internal-only, Docker-level network isolation   |
| Docker     | Controlled using `docker-compose`, volumes used |
| Jitsu UI   | Can be SSO/OIDC enabled for team access         |
