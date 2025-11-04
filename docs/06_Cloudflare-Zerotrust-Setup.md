# Guide 8: The Master Guide to Cloudflare Zero Trust

This is the central guide for my homelab's remote access. It details the complete, end-to-end process of using a custom domain (`codespicer.win`) and Cloudflare Tunnels to create a hybrid setup:

1. A Public-Facing Service: `cockpit.codespicer.win`, secured with an OTP login.

2. A Private VPN: Secure, remote access to internal services like SSH and Samba (NAS) using the WARP client.

## Part 1:

Before starting the Cloudflare setup, the Debian server must be properly configured.

1.1. Set a Static Local IP

My ISP-provided router (Jio) does not support DHCP Reservation. To ensure my server's local IP (`192.168.29.162) is permanent, I configured a static IP directly on the server.

File: `sudo nano /etc/network/interfaces`
Configuration:

```bash
# The primary network interface
allow-hotplug wlp2s0f0
iface wlp2s0f0 inet static
    wpa-ssid Duke
    wpa-psk  duke12345
    address 192.168.29.162
    netmask 255.255.255.0
    gateway 192.168.29.1
    dns-nameservers 1.1.1.1 8.8.8.8
```

Applied with: `sudo systemctl restart networking.service`

1.2. Configure the Local Firewall (UFW)

The server's firewall must allow connections to the services it's hosting.

```bash
# Check status (it's active)
sudo ufw status

# Allow Cockpit (for the tunnel)
sudo ufw allow 9090

# Allow SSH (for local access and the VPN)
sudo ufw allow OpenSSH

# Allow Samba (for local access and the VPN)
sudo ufw allow 'Samba'
```

## Part 2: The "Named Tunnel" (The Pipe)

This process connects the Cloudflare network to my server using the cloudflared client.

1. Install `cloudflared`:

   - First, install `curl`: `sudo apt update && sudo apt install curl`
   - Followed the official Cloudflare guide to add their apt repository and `sudo apt install cloudflared`.

2. Authorize Cloudflare:

   - Install the cloudflare warp app and head over to the settings - account and log into `cloudflare zero trust account`
   - It will ask you for your team name which you can find inside your cloudflare zerotrust dashboard -> settings -> custom pages.
   - the team domain will look something like this `codespicer.cloudflareaccess.com`
   - to log in you will have to create an access policy inside the `Zero trust dashboard` -> `Access` -> `Polices`
   - Make a policy like, `Action`: allow , `Include`: emails , and add your email.
   - Now when the client redirects you to the browser for login you will hit a new page named `(team-name).cloudflareaccess.com/warp`
   - here you enter your email and the otp which gets sent over to it and boom you login.

3. Create a tunnel:
   - I prefer making a tunnel using the cloudflare dashboard, `zerotrust`-> `networks`-> `tunnels`
   - follow the commands which will be like this

```bash
brew install cloudflared &&
sudo cloudflared service install eyJhIjoi...NRGt3WkRBNSJ9
```

    - running the commands give you a `tunnelID` and a `Credentials file path` for which you have to create a config file

```bash
sudo nano /etc/cloudflared/config.yml
# and
tunnel: ff77......6383347e5e6
credentials-file: /etc/cloudflared/ff77...e638347e5e6.json

```

4. Install the Service:
   - This creates the `systemd` service file that reads the `config.yml``
   - `sudo cloudflared service install` && `sudo systemctl start cloudflared`
   - Verified with `sudo systemctl status cloudflared` (shows "Active (running)") and in the Networks -> Tunnels dashboard (shows "Active").

## Part 3: Public Service Setup (Cockpit)

This exposes `cockpit.codespicer.win` to the internet, secured by an OTP.

1. Create a public hostname

   - in `networks`-> `tunnels`-> under published routes tab -> add a new route
   - subdomain: cockpit , domain: codespicer.win , service: https -> localhost:9090 , tls no verify

2. Secure with an Access Policy (OTP):
   - In the Zero Trust dashboard, I went to Access -> Applications.
   - Clicked "Add an application" and chose "Self-hosted".
   - Application name: `Cockpit`
   - Domain: `cockpit.codespicer.win``
   - Policy: Allow policy
     - selector emails
     - my email
   - This policy forces anyone visiting the URL to pass an OTP check sent to my email before they can even see the Cockpit login page.

## Part 4: Private VPN Setup (SSH & NAS)

This configures the WARP client to act as a secure VPN for accessing my internal services. This required four distinct "Allow" rules across the Cloudflare dashboard.

1. Rule 1: The Tunnel Route (The "Pipe")

   - Where: Networks -> Tunnels -> Configure -> `CIDR tab.
   - What: Added the "CIDR" route `192.168.29.0/24`
   - Why: This tells Cloudflare that my homelab tunnel is responsible for this private IP range.

2. Rule 2: The Gateway Policy (The "Firewall")

   - Where: Gateway -> Policies -> Network.
   - What: Added a "Network" policy named "Allow Homelab Access".
   - Why: This is the server-side firewall rule. It tells Cloudflare: IF `Destination IP` is `192.168.29.0/24` AND `User Email` is `my-email` , THEN `Allow` the traffic.

3. Rule 3: The Split Tunnel (The "Client-Side Routing")

   - Where: `Settings -> WARP Client -> Device profiles -> Edit "Onboarding..." -> Split Tunnels` tab.
   - What: Changed the mode to "Include IPs and domains" and added two rules:

   1. `Include` -> `192.168.29.0/24`

   2. `Include` -> `cockpit.codespicer.win` (required in Include mode)

   -Why: This tells the WARP app on my Mac: "When you see traffic for the homelab's IP range, you must send it into the tunnel."

4. Rule 4: The Enrollment Policy (The "Client Login")

   - Where: `Settings -> WARP Client -> Device enrollment`
   - What: Added a policy rule to `Allow` my email.
   - Why: This was the fix for the `"Enrollment request is invalid"` error. It authorizes my devices to log into the codespicer team.

## Part 5 : The final result

After all this, the setup is complete and works perfectly.

- Public Access:

1. Go to `https://cockpit.codespicer.win` on any browser.

2. Pass the Cloudflare Access OTP check.

3. Log in with my Debian server credentials.

4. WARP Client: Not required.

- Private Access (SSH/NAS):

1. Turn ON the Cloudflare WARP client on my Mac (which is logged into the codespicer team).

2. Connect to my phone's hotspot (to be "remote").

3. Open Terminal and `run ssh akshat@192.168.29.162`.

4. Open Finder and connect to `smb://192.168.29.162`

5. WARP Client: Required.
