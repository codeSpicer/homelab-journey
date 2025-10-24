# Guide 5: Remote Access with Cloudflare Tunnels

This guide covers setting up secure remote access to my server. The primary goal is to access services like Cockpit from the public internet without opening any ports on my home router.

This "Zero Trust" model is far more secure than the traditional method of port forwarding, as it keeps my server completely invisible to internet scanners.

## 1. The Chosen Tool: Cloudflare Tunnels

I chose Cloudflare Tunnels over traditional Dynamic DNS (DDNS) for one key reason: security.

- Traditional Method (DDNS + Port Forwarding): You open ports (e.g., 22, 9090) on your router and point them at your server. This "opens a door" to the internet, making your server vulnerable to scans and attacks.

- Cloudflare Tunnel Method: The server runs a small client (cloudflared) that creates a secure, outbound-only connection to Cloudflare's network. Cloudflare then routes traffic to your server through this tunnel. No ports are ever opened, and my home IP address is never exposed.

## 2. Installation of cloudflared

The first step was to install the cloudflared client on my Debian server. follow the documentation on cloudflare dashboard in zero trust home.
image.png

```bash
brew install cloudflared &&
sudo cloudflared service install eyJhIjoi......CJ9

```

## 3. Understanding Tunnel Types:

I discovered there are two types of tunnels, which caused some confusion.

1. Named Tunnels (Dashboard Method): This is the "official" method using the Cloudflare Zero Trust dashboard.

- Problem: When I tried to create a public hostname in the dashboard, the "Domain" dropdown was empty.

- Reason: This method requires you to own a custom domain (e.g., my-domain.com) and have it added to your Cloudflare account. I don't have one yet.

2. Quick Tunnels (CLI Method): This is a simpler, account-less method run directly from the terminal.

- Command: `cloudflared tunnel --url https://localhost:9090 --no-tls-verify

- Result: This worked perfectly and instantly gave me a public URL (e.g., `https://random-words.trycloudflare.com`).

- Problem: This tunnel is temporary. It stops as soon as I close the SSH session.

## 4. The Solution: A Permanent "Quick Tunnel" Service

The best workaround is to take the "Quick Tunnel" command and turn it into a permanent service using systemd, so it starts automatically on boot.

- Create a systemd service file:

```bash
sudo nano /etc/systemd/system/cloudflared-cockpit.service
```

- Add Service Configuration:

I added the following configuration. The ExecStart line contains the command we tested, and User=akshat ensures it runs as my user, not as root.

```bash
[Unit]
Description=Cloudflare Quick Tunnel for Cockpit
After=network.target
[Service]
User=akshat
# The --no-tls-verify flag is critical for Cockpit,
# which uses a self-signed certificate.
ExecStart=/usr/bin/cloudflared tunnel --url https://localhost:9090 --no-tls-verify
Restart=on-failure
RestartSec=5s
[Install]
WantedBy=multi-user.target
```

- Enable and Start the Service:

```bash
# Reload systemd to read the new file
sudo systemctl daemon-reload
# Enable it to start on boot
sudo systemctl enable cloudflared-cockpit.service
# Start it right now
sudo systemctl start cloudflared-cockpit.service
```

## 5. Managing the Permanent Tunnel

Now the tunnel is running as a background service.

- Check Status: `sudo systemctl status cloudflared-cockpit.service`

- Find the Public URL: The URL is logged by the service. I can find it with this command:

```bash
journalctl -u cloudflared-cockpit.service -n 10 --no-pager
```

The output will show the active `https.xxxx.trycloudflare.com URL.

Limitation: This URL is not permanent. The service is permanent, but it will generate a new, random URL every time the server reboots. This is the best possible solution without a custom domain.

Future Goal: The next step is to buy a domain, add it to Cloudflare, and switch this to a "Named Tunnel" to get a permanent, custom URL (e.g., `cockpit.my-domain.comà).
