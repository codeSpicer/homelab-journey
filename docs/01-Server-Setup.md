# Guide 1: Initial Server Setup & Hardening

This document covers the initial setup of my homelab server on an older HP Pentium laptop, from choosing an OS to basic security hardening.

## 1. Operating System: Debian 13

I initially attempted to install Fedora Server, but encountered persistent boot freezes on the laptop's hardware. After troubleshooting with various kernel parameters (inst.text, nomodeset, acpi=off) failed to resolve the issue, I switched to Debian 13 "Trixie".

Conclusion: Debian proved to have excellent hardware compatibility, and the text-based net installer worked flawlessly. This was a key lesson in choosing the right tool for the job.

## Installation Choices:

A minimal "headless" server was created by selecting only two software options:
-SSH server
-standard system utilities

## 2. Remote Access via SSH

-After installation, the first step was to enable remote management.
-Logged into the server directly to find its local IP address:

```bash
ip a
```

From my main PC, I established an SSH connection (replacing akshat and the IP address as needed):

```bash
ssh akshat@192.168.29.162
```

## 3. User Management & Security

Proper user management is critical for a secure system.

sudo and the root User:

Debian did not install the sudo package by default because a root password was set during installation.

Fix: Logged in as root (su -), installed the package (apt install sudo), and added my primary user (akshat) to the sudo group (usermod -aG sudo akshat).

Hardening - Disabling root Login:

To prevent direct login attempts to the powerful root account, I locked its password. This forces all administrative actions to be performed via an authorized sudo user, which is more secure.

```bash
sudo passwd -l root
```

## 4. Laptop-to-Server Configuration

To make the laptop behave like a server, I prevented it from suspending when the lid is closed.

Edited the systemd login configuration file:

```bash
sudo nano /etc/systemd/logind.conf
```

Uncommented and changed the HandleLidSwitch directive to:

```bash
HandleLidSwitch=ignore
```

Restarted the service to apply the new configuration:

```bash
sudo systemctl restart systemd-logind.service
```


```bash
akshat@debian:~$ fastfetch
        _,met$$$$$gg.          akshat@debian
     ,g$$$$$$$$$$$$$$$P.       -------------
   ,g$$P""       """Y$$.".     OS: Debian GNU/Linux 13 (trixie) x86_64
  ,$$P'              `$$$.     Host: HP 15 Notebook PC 
',$$P       ,ggs.     `$$b:    Kernel: Linux 6.12.48+deb13-amd64
`d$$'     ,$P"'   .    $$$     Uptime: 7 hours, 58 mins
 $$P      d$'     ,    $$P     Packages: 530 (dpkg)
 $$:      $$.   -    ,d$$'     Shell: bash 5.2.37
 $$;      Y$b._   _,d$P'       Display (CMN15AB): 1366x768 @ 60 Hz in 16" [Built-in]
 Y$$.    `.`"Y$$$$P"'          Terminal: /dev/pts/0
 `$$b      "-.__               CPU: Intel(R) Pentium(R) N3530 (4) @ 2.58 GHz
  `Y$$b                        GPU: Intel Atom Processor Z36xxx/Z37xxx Series Graphics & Display @ 0.90 GHz [Integrated]
   `Y$$.                       Memory: 533.40 MiB / 3.72 GiB (14%)
     `$$b.                     Swap: 0 B / 976.00 MiB (0%)
       `Y$$b.                  Disk (/): 1.26 GiB / 16.59 GiB (8%) - ext4
         `"Y$b._               Disk (/mnt/data): 2.12 MiB / 457.38 GiB (0%) - ext4
             `""""             Disk (/srv): 25.87 MiB / 209.27 GiB (0%) - ext4
                               Disk (/var): 371.12 MiB / 6.61 GiB (5%) - ext4
                               Local IP (wlp2s0f0): 192.xxx.xx.xxx/24
                               Locale: en_IN

                                                       
```
