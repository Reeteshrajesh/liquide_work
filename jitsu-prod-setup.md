# 🚀 Jitsu Self-Hosted HTTPS Deployment (Docker Compose + Elastic IP)

This guide documents the setup and deployment of [Jitsu](https://jitsu.com/) — an open-source Segment alternative — with PostgreSQL, Redis, and NGINX reverse proxy secured using self-signed SSL. It runs on an EC2 instance with a static Elastic IP address.

---

## 🔧 Folder Structure

```
jitsu-https/
├── docker-compose.yaml
├── nginx.conf
└── jitsu-ssl/
    ├── fullchain.pem
    └── privkey.pem
```

---

## 🧱 Services

| Service  | Purpose                        |
| -------- | ------------------------------ |
| Postgres | Metadata storage               |
| Redis    | Caching for Jitsu              |
| Jitsu    | Core application               |
| NGINX    | Reverse proxy with SSL support |

---

## 🌐 Prerequisites

* EC2 instance (Ubuntu 22.04 or 24.04)
* Docker & Docker Compose installed
* Elastic IP attached to the EC2 instance
* Port `80` and `443` opened in Security Groups

---

## 📜 Step-by-Step Deployment

### 1. 📦 Install Docker and Docker Compose

```bash
sudo apt update
sudo apt install docker.io docker-compose -y
sudo systemctl enable docker
```

---

### 2. 🛡️ Generate Self-Signed SSL Certificate

Navigate to the `jitsu-https/jitsu-ssl/` folder and generate certificates:

```bash
mkdir -p jitsu-https/jitsu-ssl
cd jitsu-https/jitsu-ssl

openssl req -x509 -nodes -days 365 \
  -newkey rsa:2048 \
  -keyout privkey.pem \
  -out fullchain.pem \
  -subj "/C=IN/ST=State/L=City/O=Org/CN=<YOUR_ELASTIC_IP>"
```

Replace `<YOUR_ELASTIC_IP>` with your actual Elastic IP.

---

### 3. ⚙️ NGINX Config (`nginx.conf`)

Located in `jitsu-https/nginx.conf`:

```nginx
events {}

http {
  server {
    listen 80;
    server_name <YOUR_ELASTIC_IP>;

    location / {
      return 301 https://$host$request_uri;
    }
  }

  server {
    listen 443 ssl;
    server_name <YOUR_ELASTIC_IP>;

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

Replace `<YOUR_ELASTIC_IP>` with your actual IP.

---

### 4. 🐳 Docker Compose Config (`docker-compose.yaml`)

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

### 5. 🚀 Start the Services

From the `jitsu-https` directory:

```bash
docker-compose up -d
```

Check logs:

```bash
docker-compose logs -f nginx
```

---

### 6. ✅ Test the Deployment

Visit:

```text
https://<YOUR_ELASTIC_IP>
```

You’ll see a **self-signed certificate warning** — proceed anyway.

---

## 🔒 Securing Further (Optional)

* Use a real domain and replace self-signed certs with [Let's Encrypt](https://letsencrypt.org/)
* Use a secure reverse proxy like [Caddy](https://caddyserver.com/)
* Store secrets securely using `.env` files or AWS Secrets Manager

---

## 🧼 Common Errors

* **Port 80 already in use**: Run `sudo lsof -i :80` and `sudo kill <PID>`
* **Self-signed SSL warning**: Normal behavior unless using valid certificates
* **Jitsu not responding**: Run `docker-compose ps` and `docker-compose logs jitsu`

---

## 👤 Maintainer

**Reetesh Kumar** — DevOps @ [Liquide.life](https://liquide.life) 
* @ [LinkedIn](https://www.linkedin.com/in/reetesh-kumar-850807255/)
* @ [Mail](uttamreetesh@gmail.com)
