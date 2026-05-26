# Fresh Start: Homelab Roadmap on Ubuntu 26.04 LTS

This is the master guide for rebuilding the homelab from scratch on Ubuntu 26.04 LTS.
Follow the phases in order — each one builds on the last.

**Hardware:** HP 15 Notebook (Pentium N3530, 4 GB RAM, 256 GB SSD)
**Domain:** codespicer.win (Cloudflare)

---

## Phase 0: Ubuntu Installation Checklist

Before booting into Ubuntu, do these:

- Download [Ubuntu Server 26.04 LTS](https://ubuntu.com/download/server) ISO
- Flash to USB with Balena Etcher or `dd`
- Boot laptop from USB (F9/F10/F12 on HP for boot menu)

**During installation, select:**
- Minimal server install (no GUI)
- OpenSSH server — check this box
- Leave Docker/snap options unchecked (we'll install manually)
- Set hostname (e.g. `homelab`)
- Create user `akshat` with a strong password

---

## Phase 1: First Boot — Base Hardening

*Do this from the laptop directly or over SSH after you get the IP.*

### 1.1 Get the server IP and SSH in from your main machine

```bash
# On the laptop:
ip a

# From your main PC:
ssh akshat@<SERVER_IP>
```

### 1.2 Update everything

```bash
sudo apt update && sudo apt upgrade -y && sudo apt autoremove -y
```

### 1.3 Install essential tools

```bash
sudo apt install -y curl wget git htop net-tools ufw unzip fastfetch
```

### 1.4 Lock root login

```bash
sudo passwd -l root
```

### 1.5 Configure laptop lid behavior (server mode)

```bash
sudo nano /etc/systemd/logind.conf
```

Uncomment and set:
```
HandleLidSwitch=ignore
HandleLidSwitchDocked=ignore
```

```bash
sudo systemctl restart systemd-logind
```

### 1.6 Disable sleep/suspend on Ubuntu (belt-and-suspenders)

```bash
sudo systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target
```

### 1.7 Set up UFW firewall

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh
sudo ufw enable
sudo ufw status verbose
```

### 1.8 Set up SSH key-based login (from your main PC)

```bash
# On your main PC — generate a key if you don't have one:
ssh-keygen -t ed25519 -C "homelab"

# Copy it to the server:
ssh-copy-id akshat@<SERVER_IP>
```

After confirming key login works, optionally disable password auth:
```bash
sudo nano /etc/ssh/sshd_config
# Set: PasswordAuthentication no
sudo systemctl restart ssh
```

### 1.9 Set a static local IP (recommended)

Either set a DHCP reservation in your router (preferred) or configure netplan:

```bash
sudo nano /etc/netplan/00-installer-config.yaml
```

Example static config (adjust interface name from `ip a`):
```yaml
network:
  version: 2
  ethernets:
    enp2s0:           # replace with your interface name
      dhcp4: no
      addresses: [192.168.1.50/24]
      routes:
        - to: default
          via: 192.168.1.1
      nameservers:
        addresses: [1.1.1.1, 8.8.8.8]
```

```bash
sudo netplan apply
```

---

## Phase 2: Web Management Dashboard (Cockpit)

Cockpit gives you a browser-based UI to manage the server — processes, storage, networking, logs, and terminal.

```bash
sudo apt install -y cockpit
sudo systemctl enable --now cockpit.socket
sudo ufw allow 9090
```

Access at: `https://<SERVER_IP>:9090`
Login with your `akshat` credentials (ignore the self-signed cert warning).

---

## Phase 3: Secondary Storage (if you have an extra drive)

### 3.1 Find the drive

```bash
lsblk
```

It will appear as `/dev/sdb` or similar (not the OS drive `/dev/sda`).

### 3.2 Format and mount

```bash
# Create a filesystem (wipes the drive):
sudo mkfs.ext4 /dev/sdb

# Create a mount point:
sudo mkdir -p /mnt/data

# Mount it:
sudo mount /dev/sdb /mnt/data

# Make it permanent (get UUID first):
sudo blkid /dev/sdb
```

Add to `/etc/fstab`:
```
UUID=<your-uuid>  /mnt/data  ext4  defaults  0  2
```

```bash
sudo mount -a
df -h
```

---

## Phase 4: NAS with Samba

Share files on your local network from the server.

```bash
sudo apt install -y samba
```

Create a shared directory:
```bash
sudo mkdir -p /srv/shared
sudo chown akshat:akshat /srv/shared
sudo chmod 775 /srv/shared
```

Configure Samba — append to the bottom of `/etc/samba/smb.conf`:
```ini
[Homelab Share]
   path = /srv/shared
   browsable = yes
   writable = yes
   valid users = akshat
   create mask = 0664
   directory mask = 0775
```

Set a Samba password:
```bash
sudo smbpasswd -a akshat
```

```bash
sudo systemctl restart smbd nmbd
sudo ufw allow samba
```

Connect from Windows: `\\<SERVER_IP>\Homelab Share`
Connect from Mac/Linux: `smb://<SERVER_IP>/Homelab Share`

---

## Phase 5: Docker & Container Stack

### 5.1 Install Docker (official method — not the snap)

```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker akshat
newgrp docker
```

Verify:
```bash
docker run hello-world
```

### 5.2 Install Docker Compose v2

```bash
sudo apt install -y docker-compose-plugin
docker compose version
```

### 5.3 Create your Docker directory structure

```bash
mkdir -p ~/docker/{uptime-kuma,portainer,pihole,nginx-proxy-manager}
```

### 5.4 Deploy Portainer (container management UI)

```bash
docker volume create portainer_data

docker run -d \
  --name portainer \
  --restart=always \
  -p 9443:9443 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

Access at: `https://<SERVER_IP>:9443`

```bash
sudo ufw allow 9443
```

### 5.5 Deploy Uptime Kuma (service monitoring)

```bash
cat > ~/docker/uptime-kuma/compose.yml << 'EOF'
services:
  uptime-kuma:
    image: louislam/uptime-kuma:1
    container_name: uptime-kuma
    restart: always
    ports:
      - "3001:3001"
    volumes:
      - uptime-kuma-data:/app/data

volumes:
  uptime-kuma-data:
EOF

docker compose -f ~/docker/uptime-kuma/compose.yml up -d
sudo ufw allow 3001
```

Access at: `http://<SERVER_IP>:3001`

---

## Phase 6: Remote Access — Cloudflare Tunnel

This exposes your services to the internet with zero open ports, using your `codespicer.win` domain.

### 6.1 Install cloudflared

```bash
curl -fsSL https://pkg.cloudflare.com/cloudflare-main.gpg | sudo tee /usr/share/keyrings/cloudflare-main.gpg > /dev/null

echo "deb [signed-by=/usr/share/keyrings/cloudflare-main.gpg] https://pkg.cloudflare.com/cloudflared any main" | \
  sudo tee /etc/apt/sources.list.d/cloudflared.list

sudo apt update && sudo apt install -y cloudflared
```

### 6.2 Authenticate and create a Named Tunnel

```bash
cloudflared tunnel login
cloudflared tunnel create homelab
```

### 6.3 Configure the tunnel

```bash
mkdir -p ~/.cloudflared
nano ~/.cloudflared/config.yml
```

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
  - service: http_status:404
```

### 6.4 Create DNS records

```bash
cloudflared tunnel route dns homelab cockpit.codespicer.win
cloudflared tunnel route dns homelab kuma.codespicer.win
cloudflared tunnel route dns homelab portainer.codespicer.win
```

### 6.5 Run tunnel as a system service

```bash
sudo cloudflared --config /home/akshat/.cloudflared/config.yml service install
sudo systemctl enable cloudflared
sudo systemctl start cloudflared
sudo systemctl status cloudflared
```

---

## Phase 7: Cloudflare Zero Trust Access (Lock Down Your Subdomains)

Protect your exposed services behind a login page so only you can reach them.

1. Go to [one.dash.cloudflare.com](https://one.dash.cloudflare.com) → Access → Applications
2. Add Application → Self-hosted
3. Set the domain (e.g. `cockpit.codespicer.win`)
4. Under Policies: Add a rule — `Emails` → `your email` (or use GitHub/Google OAuth)
5. Repeat for each subdomain

No ports are ever opened on your router.

---

## What to Add Next — Service Suggestions

Once the core stack is running, here are great next services to self-host, roughly ordered by effort:

| Service | What it does | Docker image |
|---|---|---|
| **Pi-hole** | Network-wide ad blocking & DNS | `pihole/pihole` |
| **Nginx Proxy Manager** | GUI reverse proxy with SSL | `jc21/nginx-proxy-manager` |
| **Vaultwarden** | Self-hosted Bitwarden password manager | `vaultwarden/server` |
| **Jellyfin** | Media server (movies, music, TV) | `jellyfin/jellyfin` |
| **Nextcloud** | Self-hosted Google Drive/Docs alternative | `nextcloud` |
| **Grafana + Prometheus** | Full metrics dashboards | `grafana/grafana` + `prom/prometheus` |
| **node-exporter** | Exports system metrics to Prometheus | `prom/node-exporter` |
| **Gitea** | Self-hosted GitHub | `gitea/gitea` |
| **Homer / Homepage** | Beautiful homelab dashboard | `b4bz/homer` |
| **Watchtower** | Auto-updates your Docker containers | `containrrr/watchtower` |

### Hosting your own projects

See [Guide 8: Hosting Apps with Cloudflare Subdomains](./08_Hosting_Apps_With_Cloudflare_Subdomains.md) for a step-by-step on deploying your own backend/frontend apps and routing them to `api.codespicer.win`, `app.codespicer.win`, etc.

### Suggested learning additions (good for a software engineering career)
- **Ansible** — automate server config so you never have to redo this manually again
- **Terraform** — provision infra as code (Cloudflare + local)
- **GitHub Actions** — set up CI/CD that deploys to your homelab
- **Tailscale** — zero-config VPN, great alternative/complement to Cloudflare Tunnel
- **Semaphore** — open-source Ansible UI you can run in Docker

---

## Quick Recovery Checklist

If you ever need to rebuild again, run through this order:

```
[ ] Ubuntu install + SSH + key auth
[ ] UFW rules (ssh, 9090, 9443, 3001)
[ ] Lid close + sleep disabled
[ ] Cockpit installed
[ ] Secondary drive mounted (update fstab)
[ ] Samba configured
[ ] Docker + Compose installed
[ ] Portainer up
[ ] Uptime Kuma up
[ ] cloudflared installed + Named Tunnel running
[ ] Zero Trust policies in Cloudflare dashboard
```

---

*Homelab rebuild started: May 2026 | OS: Ubuntu 26.04 LTS | Host: HP 15 Notebook*
