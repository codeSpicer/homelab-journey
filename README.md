# My Homelab Journey 🚀

Welcome to my homelab project! This repository documents my journey of turning a spare HP laptop into a fully functional server. The goal is not just to host cool services, but to learn essential skills for a software engineering career, including Linux administration, networking, containerization, and automation.

## Hardware Specs 💻

-Device: HP 15 Notebook PC

-CPU: Intel(R) Pentium(R) N3530 (4) @ 2.58 GHz

-GPU: Intel Atom Integrated Graphics

-RAM: 4 GB

-Storage: 256 GB SSD

-Operating System: Ubuntu 26.04 LTS (Server) — *fresh start May 2026, previously Debian 13*

- Domain: codespicer.win (managed by Cloudflare)

- Containerization: Docker & Docker Compose

- Monitoring: Uptime Kuma

## 📖 Documentation & Guides

### Fresh Start (Ubuntu 26.04 LTS)

- [Fresh Start Roadmap — Ubuntu 26.04](./docs/00_Fresh_Start_Ubuntu_26.04_Roadmap.md)
  Complete end-to-end guide: base hardening → Cockpit → Samba → Docker → Portainer → Uptime Kuma → Cloudflare Tunnel → Zero Trust. Start here.

- [Hosting Apps with Cloudflare Subdomains](./docs/08_Hosting_Apps_With_Cloudflare_Subdomains.md)
  Deploy your own backend/frontend projects and route them to subdomains like `api.codespicer.win`. Covers Docker, Nginx Proxy Manager, CI/CD with GitHub Actions, and Zero Trust protection.

### Previous Setup (Debian 13 — archived for reference)

1. [Initial Server Setup](./docs/01-Server-Setup.md)
   OS install, SSH, user management, security hardening.

2. [Web Management Dashboard with Cockpit](./docs/02-Web-Dashboard.md)
   Cockpit web interface for server monitoring and management.

3. [Setting up Secondary HDD Drive](./docs/03-Secondary-Drive.md)
   Formatting and permanently mounting a secondary hard drive.

4. [Setting up a NAS with Samba](./docs/04-NAS-Server-Samba.md)
   Network Attached Storage shares on the server.

5. [Remote Access with Cloudflare Tunnels](./docs/05-Cloudflare-Tunnels.md)
   Secure remote access without port forwarding.

6. [Cloudflare ZeroTrust Setup and Domain](./docs/06_Cloudflare-Zerotrust-Setup.md)
   Zero Trust access policies for exposed services.

7. [Docker & Uptime Kuma setup](./docs/07_Docker_&_UptimeKuma_Setup.md)
   First containerized service with monitoring.

---

Core Technologies
- OS: Ubuntu 26.04 LTS (Server)
- Web Dashboard: Cockpit
- Containerization: Docker & Docker Compose
- Container UI: Portainer
- Monitoring: Uptime Kuma
- Remote Access: Cloudflare Tunnel + Zero Trust
- Firewall: UFW
- Version Control: Git & GitHub
