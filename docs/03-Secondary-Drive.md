# Guide 3: Setting Up a Secondary Storage Drive

This guide details the process of formatting and permanently mounting a secondary hard drive on the server. The goal is to prepare a 465 GB HDD (/dev/sda) to be used for storing data, such as Docker volumes, media, or backups.

## 1. Identifying the Correct Drive

Before making any changes, it's critical to identify the correct target drive to avoid wiping the operating system.

The lsblk command provides a clear overview of all block devices.

```bash
akshat@debian:~$ lsblk
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sda      8:0    0 465.8G  0 disk
├─sda1   8:1    0 223.2G  0 part
├─sda2   8:2    0   783M  0 part
└─sda3   8:3    0 241.7G  0 part
sdb      8:16   0 238.5G  0 disk
├─sdb1   8:17   0    17G  0 part /
├─sdb2   8:18   0     1K  0 part
├─sdb5   8:21   0   6.8G  0 part /var
├─sdb6   8:22   0   976M  0 part [SWAP]
└─sdb7   8:23   0 213.7G  0 part /srv
```

`/dev/sdb` is the OS drive, identifiable by its active mount points (/, /var, /srv).

/dev/sda is the target data drive, as it has no mounted partitions.

## 2. Partitioning and Formatting

The existing partitions on /dev/sda were wiped to create a single, clean partition.

Create a new GPT partition table (this deletes all existing partitions):

```bash
sudo parted /dev/sda mklabel gpt
```

Create one new primary partition that spans the entire drive:

```bash
sudo parted -a opt /dev/sda mkpart primary ext4 0% 100%
```

Format the new partition (/dev/sda1) with the ext4 filesystem:

```bash
sudo mkfs.ext4 /dev/sda1
```

## 3. Mounting the Drive

Create a mount point: This is an empty directory where the drive's contents will be accessed.

```bash
sudo mkdir /mnt/data
```

Temporarily mount the drive to the mount point:

```bash
sudo mount /dev/sda1 /mnt/data
```

Assign ownership to my user account to grant read/write permissions:

```bash
sudo chown -R akshat:akshat /mnt/data
```

## 4. Auto-Mounting on Reboot (fstab)

To ensure the drive is automatically mounted every time the server boots, it must be added to /etc/fstab.

Find the partition's unique identifier (UUID), which is safer to use than its name:

```bash
sudo blkid /dev/sda1
# Output: /dev/sda1: UUID="ca59bf47-7f08-4a87-8491-c09182267c28" ...
```

Edit the fstab file:

```bash
sudo nano /etc/fstab
```

Add the following line to the end of the file, using the copied UUID:

```bash
UUID=ca59bf47-7f08-4a87-8491-c09182267c28   /mnt/data   ext4    defaults    0   2
```
