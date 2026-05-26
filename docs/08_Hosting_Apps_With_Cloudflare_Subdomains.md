# Guide 8: Hosting Your Own Apps on the Homelab

This guide covers deploying your own backend or frontend projects on the homelab and
routing them to a public subdomain on `codespicer.win` via Cloudflare Tunnel.

**Assumes:** Phase 5 (Docker) and Phase 6 (Cloudflare Named Tunnel) from the roadmap are done.

---

## How it works

```
Browser → cockpit.codespicer.win
                │
        Cloudflare Edge
                │  (tunnel — no open ports)
        cloudflared daemon (on your server)
                │
        your app container on localhost:PORT
```

Each subdomain is just one extra line in your tunnel config + one DNS route. That's it.

---

## Method A: Direct Tunnel Routing (Recommended — Simple)

Best for most cases. The tunnel talks directly to your app's container port.

### Step 1 — Containerize your app

**Backend example (Node.js / Express):**

```dockerfile
# Dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --production
COPY . .
EXPOSE 3000
CMD ["node", "index.js"]
```

**Frontend example (React / Vite build):**

```dockerfile
# Dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
EXPOSE 80
```

### Step 2 — Write a compose file

Create a directory for your project on the server:

```bash
mkdir -p ~/docker/my-app
nano ~/docker/my-app/compose.yml
```

```yaml
services:
  backend:
    build: /home/akshat/projects/my-backend   # path to your project on the server
    container_name: my-backend
    restart: unless-stopped
    ports:
      - "3000:3000"
    env_file:
      - .env

  frontend:
    build: /home/akshat/projects/my-frontend
    container_name: my-frontend
    restart: unless-stopped
    ports:
      - "8080:80"
```

```bash
# Optional: create a .env file with secrets
nano ~/docker/my-app/.env
```

```bash
docker compose -f ~/docker/my-app/compose.yml up -d
```

Verify the app is running locally:
```bash
curl http://localhost:3000       # backend
curl http://localhost:8080       # frontend
```

### Step 3 — Add to Cloudflare Tunnel config

```bash
nano ~/.cloudflared/config.yml
```

Add your new subdomains inside the `ingress` block, **above** the final `http_status:404` catch-all:

```yaml
tunnel: homelab
credentials-file: /home/akshat/.cloudflared/<TUNNEL_ID>.json

ingress:
  - hostname: cockpit.codespicer.win
    service: https://localhost:9090
    originRequest:
      noTLSVerify: true
  - hostname: kuma.codespicer.win
    service: http://localhost:3001
  - hostname: portainer.codespicer.win
    service: https://localhost:9443
    originRequest:
      noTLSVerify: true

  # --- your apps ---
  - hostname: api.codespicer.win
    service: http://localhost:3000
  - hostname: app.codespicer.win
    service: http://localhost:8080

  - service: http_status:404
```

### Step 4 — Create the DNS records

```bash
cloudflared tunnel route dns homelab api.codespicer.win
cloudflared tunnel route dns homelab app.codespicer.win
```

### Step 5 — Restart the tunnel

```bash
sudo systemctl restart cloudflared
sudo systemctl status cloudflared
```

Your app is now live at `https://api.codespicer.win` and `https://app.codespicer.win`.
Cloudflare handles HTTPS/TLS automatically — no certificates to manage.

---

## Method B: Nginx Proxy Manager (Scales Better)

Use this when you have many apps and want a GUI to manage routing rules without
editing the tunnel config every time.

NPM acts as a reverse proxy inside your server. The tunnel points everything at NPM,
and NPM routes by hostname to the right container.

```
cloudflared → NPM (port 80) → routes by hostname → container
```

### Step 1 — Deploy Nginx Proxy Manager

```bash
mkdir -p ~/docker/nginx-proxy-manager
nano ~/docker/nginx-proxy-manager/compose.yml
```

```yaml
services:
  npm:
    image: jc21/nginx-proxy-manager:latest
    container_name: nginx-proxy-manager
    restart: unless-stopped
    ports:
      - "80:80"
      - "81:81"     # NPM admin UI
      - "443:443"
    volumes:
      - npm-data:/data
      - npm-letsencrypt:/etc/letsencrypt

volumes:
  npm-data:
  npm-letsencrypt:
```

```bash
docker compose -f ~/docker/nginx-proxy-manager/compose.yml up -d
sudo ufw allow 81
```

NPM admin UI: `http://<SERVER_IP>:81`
Default login: `admin@example.com` / `changeme` (change on first login)

### Step 2 — Put your apps on a shared Docker network

So NPM can reach them by container name:

```bash
docker network create homelab-net
```

Add `networks` to every compose file:

```yaml
services:
  backend:
    container_name: my-backend
    # ... rest of config ...
    networks:
      - homelab-net

networks:
  homelab-net:
    external: true
```

Do the same for NPM:
```yaml
services:
  npm:
    # ...
    networks:
      - homelab-net

networks:
  homelab-net:
    external: true
```

Recreate containers to apply the network:
```bash
docker compose -f ~/docker/nginx-proxy-manager/compose.yml up -d --force-recreate
docker compose -f ~/docker/my-app/compose.yml up -d --force-recreate
```

### Step 3 — Add a Proxy Host in NPM

1. Open `http://<SERVER_IP>:81`
2. Proxy Hosts → Add Proxy Host
3. Fill in:
   - **Domain Name:** `api.codespicer.win`
   - **Forward Hostname:** `my-backend` (Docker container name)
   - **Forward Port:** `3000`
   - Enable "Websockets Support" if your app uses WebSockets
4. Save

Repeat for each app/subdomain.

### Step 4 — Point the tunnel at NPM

Update `~/.cloudflared/config.yml` — instead of per-app entries, use a wildcard to
send everything to NPM:

```yaml
tunnel: homelab
credentials-file: /home/akshat/.cloudflared/<TUNNEL_ID>.json

ingress:
  - service: http://localhost:80
```

NPM then handles all hostname routing internally. You never touch the tunnel config again
when adding new apps — just add a Proxy Host in the NPM UI.

```bash
sudo systemctl restart cloudflared
```

---

## Getting Your Code onto the Server

Three common approaches:

### Option 1 — Clone directly from GitHub

```bash
# On the server:
git clone https://github.com/codeSpicer/my-backend.git ~/projects/my-backend
cd ~/projects/my-backend
docker compose up -d
```

For updates:
```bash
cd ~/projects/my-backend && git pull && docker compose up -d --build
```

### Option 2 — Copy files with scp/rsync (quick one-off)

```bash
# From your dev machine:
rsync -avz ./my-backend/ akshat@<SERVER_IP>:~/projects/my-backend/
```

### Option 3 — CI/CD with GitHub Actions (best long-term)

Create `.github/workflows/deploy.yml` in your project:

```yaml
name: Deploy to Homelab

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Deploy via SSH
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.SERVER_IP }}
          username: akshat
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            cd ~/projects/my-backend
            git pull
            docker compose up -d --build
```

Add `SERVER_IP` and `SSH_PRIVATE_KEY` as GitHub Actions secrets.
With this, every push to `main` auto-deploys to your homelab.

---

## Protecting Your App with Zero Trust

If you don't want the app publicly accessible (e.g. an internal API), add a Cloudflare
Access policy the same way you protected Cockpit:

1. Cloudflare dashboard → Zero Trust → Access → Applications → Add
2. Domain: `api.codespicer.win`
3. Policy: allow only your email
4. Anyone else gets a login prompt, not your app

---

## Quick Reference: Adding a New App

```
1. Write Dockerfile + compose.yml → docker compose up -d
2. Verify locally: curl http://localhost:<PORT>
3. Add ingress line to ~/.cloudflared/config.yml   (Method A)
   OR add a Proxy Host in NPM UI                  (Method B)
4. cloudflared tunnel route dns homelab <subdomain>.codespicer.win
5. sudo systemctl restart cloudflared              (Method A only)
6. Hit https://<subdomain>.codespicer.win
```

---

*See also: [Fresh Start Roadmap](00_Fresh_Start_Ubuntu_26.04_Roadmap.md) — Phase 6 for initial tunnel setup.*
