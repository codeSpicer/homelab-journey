# Guide 7: Docker & Uptime Kuma Setup

This guide details the process of getting started with Docker on the Debian server. The first project is deploying Uptime Kuma, a powerful monitoring dashboard, as our first containerized service.

<img width="1469" height="917" alt="image" src="https://github.com/user-attachments/assets/0e94f061-7929-49b8-9502-a320b19c205c" />


## 1. Docker Installation:

For the installation I have followed the official documentation of the docker. [Link to official docs.](https://docs.docker.com/engine/install/debian/#install-using-the-repository)

After folloing the docs I had to add my user to the docker group to run Docker commands without sudo:

```bash
sudo usermod -aG docker akshat
```

## 2. Deploying Kuma Uptime

I used Docker Compose to define and run the Uptime Kuma service for a simple and repeatable deployment.

1. Created a project directory: `mkdir ~/docker_projects && cd ~/docker_projects`

2. Created the docker-compose.yml file: `nano docker-compose.yml`

3. Added the following configuration:

```bash
services:
  kuma:
    image: louislam/uptime-kuma:latest
    container_name: uptime-kuma
    restart: unless-stopped
    volumes:
      - ./kuma_data:/app/data
    ports:
      - "3001:3001"
    extra_hosts:
      - "host.docker.internal:host-gateway"
```

- `image`: Specifies the official Uptime Kuma image.

- `container_name`: Gives the container a friendly name.

- `volumes`: Maps a local directory (`./kuma_data`) on the Debian server to the container's data directory. This ensures all my Uptime Kuma settings are saved outside the container and will survive updates or restarts.

- `ports`: Maps port `3001` on the Debian server to port `3001` inside the container.

- `extra_hosts` : Helps Container reach the Debian server instead of trying to find inside it's own container.

4. Started the service: `docker compose up -d`

## 3. Exposing Kuma Uptime Publically

Just like with Cockpit, I used a Cloudflare Tunnel and Access policy to securely expose my new dashboard to the internet.

1. Cloudflare Tunnel: Added a new "Public Hostname" to my homelab tunnel.

- Application: Uptime Kuma
- Subdomain: status
- Domain: codespicer.win
- Service: `http://localhost:3001`

2. Cloudflare Access: Added a new "Self-hosted" application in the Access -> Applications menu.

- Application name: Uptime Kuma
- Domain: `status.codespicer.win`
- Policy: Re-used my "Allow My Email" policy, which forces an OTP login for anyone visiting https://status.codespicer.win.

## 4. Configuring Monitors

After setting up Uptime Kuma, I created a full dashboard to monitor the health of my entire homelab.
The services I am monitoring are:

### Debian Cockpit:

Type: HTTP(s)
URL: `https://192.168.29.162:9090`
Note: Required "Ignore SSL/TLS Error" to be ON.

### Home Router:

Type: Ping
Hostname: `192.168.29.1` (My router's gateway IP)
Internet Connection:
Type: Ping
Hostname: `1.1.1.1` (Cloudflare's public DNS)

### Samba (NAS):

Type: TCP Port
Hostname: `192.168.29.162`
Port: `445`

### SSH Server:

Type: TCP Port
Hostname: `192.168.29.162`
Port: `22`

### Web Cockpit:

Type: HTTP(s)
URL: `https://cockpit.codespicer.win`

Note: This monitors the public URL, ensuring the tunnel and access policy are working.

## Key Learning: Container Networking (localhost vs. Host IP)

I ran into an important networking issue. When trying to add the "Debian Cockpit" monitor, I first tried httpsL//localhost:9090, but it failed.

- The Problem: When a command is run inside a Docker container, localhost refers to the container itself, not the Debian server (the "host") it's running on. Uptime Kuma was trying to find Cockpit inside its own container and found nothing.

- The Solution: I had to use the Debian server's actual IP on the local network: 192.168.29.162. The Uptime Kuma container can reach out to this IP and find the Cockpit service running on the host. This is a fundamental concept of Docker networking.
