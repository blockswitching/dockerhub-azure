# 🐳 Flask App Deployment on Azure VM — Docker Hub + Nginx

> **No Git. No Jenkins. No Kubernetes. No Terraform.**
> A clean setup using Docker Hub, Docker Compose, and Nginx.

---

## Architecture

```
Docker Hub
    │
    ▼
Azure VM
    │
    ▼
docker pull
    │
    ▼
Docker Container
    │
    ▼
Nginx (reverse proxy on port 80)
```

---

## Prerequisites

- Azure VM running **Ubuntu 22.04 LTS** (or similar)
- Port **80** HTTP - open in Azure Network Security Group (NSG)
- Port **22** SSH - open in Azure Network Security Group (NSG)
- Your Flask Docker image pushed to **Docker Hub**
- SSH access to the VM

---

## Step 1 — Install Docker

```bash
sudo apt update
sudo apt install -y docker.io
sudo systemctl enable docker
sudo systemctl start docker
```

Verify the installation:

```bash
docker --version
```

---

## Step 2 — Pull Your Image from Docker Hub

```bash
docker pull <your-dockerhub-username>/<image-name>:latest
```

**Example:**

```bash
docker pull myuser/flask-kube-app:latest
```

---

## Step 3 — Run the Container

Assuming your Flask app listens on port `5001` inside the container:

```bash
docker run -d \
  --name flask-app \
  --restart unless-stopped \
  -p 5000:5001 \
  <your-dockerhub-username>/<image-name>:latest
```

| Flag | Purpose |
|---|---|
| `-d` | Run in background (detached) |
| `--name flask-app` | Name the container for easy reference |
| `--restart unless-stopped` | Auto-restart on crash or reboot |
| `-p 5000:5001` | Map host port 5000 → container port 5001 |

Verify the container is running:

```bash
docker ps
curl http://localhost:5000
```

---

## Step 4 — Install and Configure Nginx

### Install Nginx

```bash
sudo apt install -y nginx
```

### Create the site config

```bash
sudo nano /etc/nginx/sites-available/flask-app
```

Paste the following:

```nginx
server {
    listen 80;
    server_name _;

    location / {
        proxy_pass http://127.0.0.1:5000;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### Enable the site

```bash
sudo ln -s /etc/nginx/sites-available/flask-app /etc/nginx/sites-enabled/
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl reload nginx
```

Your app is now accessible at `http://<your-vm-public-ip>`.

---

## Step 5 — Updating the App (Docker Compose approach)

### One-time setup — create `docker-compose.yml`

```bash
mkdir -p ~/flask-app && cd ~/flask-app
nano docker-compose.yml
```

Paste:

```yaml
services:
  app:
    image: <your-dockerhub-username>/<image-name>:latest
    container_name: flask-app
    restart: unless-stopped
    ports:
      - "5000:5001"
```

Start the app:

```bash
docker compose up -d
```

### Every future update

When you push a new image to Docker Hub, update the running container with just two commands:

```bash
docker compose pull
docker compose up -d
```

Docker Compose handles stopping the old container and starting the new one automatically.

---

## Manual Update (without Docker Compose)

If you prefer plain Docker commands:

```bash
docker pull <your-dockerhub-username>/<image-name>:latest

docker rm -f flask-app

docker run -d \
  --name flask-app \
  --restart unless-stopped \
  -p 5000:5001 \
  <your-dockerhub-username>/<image-name>:latest
```

---

## Useful Commands

| Command | Description |
|---|---|
| `docker ps` | List running containers |
| `docker logs flask-app` | View container logs |
| `docker logs -f flask-app` | Follow live logs |
| `docker exec -it flask-app bash` | Shell into the container |
| `sudo systemctl status nginx` | Check Nginx status |
| `sudo nginx -t` | Test Nginx config |
| `sudo systemctl reload nginx` | Reload Nginx without downtime |

---

## Troubleshooting

**Container exits immediately?**
```bash
docker logs flask-app
```

**Nginx returns 502 Bad Gateway?**
- Make sure the Flask container is running: `docker ps`
- Check the port mapping matches your `proxy_pass` config

**App not reachable from browser?**
- Confirm port 80 is open in your Azure NSG inbound rules
- Check Nginx is active: `sudo systemctl status nginx`

---

## Summary

| Tool | Role |
|---|---|
| **Docker Hub** | Image registry |
| **Docker / Docker Compose** | Container runtime and lifecycle |
| **Nginx** | Reverse proxy (port 80 → 5000) |
| **Azure VM** | Host machine |

No Git, no CI/CD pipeline, no orchestration overhead — just push your image and pull it on the VM.
