# Guide 4: Creating Network Shares (NAS) with Samba

This guide covers the setup of Network Attached Storage (NAS) shares on the server. The goal is to make storage accessible over the local network to other computers.

We will set up two shares:

    1. [Data]: A large-capacity share from the 465 GB HDD, mounted at /mnt/data. Ideal for bulk storage, media, and backups.

    2. [SSD-Share]: A high-speed share from the SSD, located at /srv/ssd-share. Perfect for frequently accessed files and active projects.

The software used for this is Samba, the standard implementation of the SMB/CIFS protocol for Linux.

## 1. Installation

Samba is available in the official Debian repositories.

```bash
sudo apt update
sudo apt install samba
```

## 2. Configuration for Multiple Shares

Samba's behavior is controlled by /etc/samba/smb.conf. We can define as many shares as we need.

1. Create Directories: First, ensure the directories to be shared exist and have the correct permissions.
```bash
# Directory for the large HDD share (already done in Guide 3)
sudo mkdir /mnt/data
sudo chown akshat:akshat /mnt/data

# Directory for the fast SSD share
sudo mkdir /srv/ssd-share
sudo chown akshat:akshat /srv/ssd-share
```

2. Edit the Samba Config File:
```bash
sudo nano /etc/samba/smb.conf
```

3. Add Share Blocks: Add the following configuration blocks to the end of the file.

```bash
# Share for the large HDD
[Data]
comment = Server Data Drive
path = /mnt/data
writable = yes
guest ok = no
read only = no
create mask = 0775
directory mask = 0775
valid users = akshat

# Share for the fast SSD
[SSD-Share]
comment = Fast SSD Storage
path = /srv/ssd-share
writable = yes
guest ok = no
read only = no
create mask = 0775
directory mask = 0775
valid users = akshat
```

3. Disabling Default Home Directory Sharing

By default, Samba automatically shares a user's home directory. To keep the setup clean and only show our intended shares, this feature was disabled.

In `/etc/samba/smb.conf`, locate the [homes] section.

Comment out the entire section by placing a semicolon (`;`) at the beginning of each line.

```bash
;[homes]
; comment = Home Directories
; ...
```

4. User and Password Setup

Samba uses its own password management. A Samba password must be created for any user who needs access. (This only needs to be done once per user).

Created a Samba password for my user `akshat`:

```bash
sudo smbpasswd -a akshat
```

5. Service Restart and Firewall

To apply all configuration changes, the Samba service must be restarted.

```bash
sudo systemctl restart smbd
```

Additionally, the Uncomplicated Firewall (UFW) needs to allow Samba traffic. (This only needs to be done once).

```bash
sudo ufw allow 'Samba'
```

After these steps, both the "Data" and "SSD-Share" folders are accessible from other computers on the local network.
