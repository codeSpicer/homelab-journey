# Guide 2: Web Management Dashboard with Cockpit

To easily monitor and manage server resources without needing to be logged in via SSH, I installed Cockpit, a lightweight web-based console.

## 1. Installation

Cockpit is available in the official Debian repositories, making installation simple.

Update the package list:

```bash
sudo apt update
```

Install the Cockpit package:

```bash
sudo apt install cockpit
```

During installation, a warning appeared (Warning: Tried to start delayed item), but this was a non-critical error. The service socket was checked and confirmed to be active after installation was complete.

## 2. Firewall Configuration

After installation, the Cockpit dashboard was inaccessible from the browser. The cause was the Uncomplicated Firewall (UFW) blocking traffic on Cockpit's default port.

The Fix:

Checked the firewall status to confirm it was active:

```bash
sudo ufw status
```

Added a new rule to allow incoming traffic on port 9090:

```bash
sudo ufw allow 9090
```

Verified the rule was added successfully. The status command now shows 9090 in the list of allowed ports.

## 3. Accessing the Dashboard

With the firewall rule in place, the dashboard is now accessible.

URL: `https://<your-server-ip>:9090` (e.g., https://192.168.29.162:9090)

Login: Use the same username and password as your Debian user (akshat).

A browser security warning about a "self-signed certificate" is expected and safe to bypass on a local network.
