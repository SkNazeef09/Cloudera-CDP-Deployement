# Cloudera Data Platform — Server Setup Guide

This guide documents the steps required to prepare a Linux server for Cloudera Manager (CM) 7.4.4 installation, including system updates, kernel tuning, dependencies, and the CM installer itself.

## Table of Contents

1. [Update the Server](#1-update-the-server)
2. [Disable Transparent Huge Pages](#2-disable-transparent-huge-pages)
3. [Install NTP](#3-install-ntp)
4. [Set Swappiness](#4-set-swappiness)
5. [Install psycopg2 Package](#5-install-psycopg2-package)
6. [Create AMI (Snapshot)](#6-create-ami-snapshot)
7. [Install and Start Cloudera Manager](#7-install-and-start-cloudera-manager)

---

## 1. Update the Server

Update package lists and upgrade all installed packages to the latest version.

```bash
sudo apt-get update && sudo apt-get dist-upgrade -y
```

---

## 2. Disable Transparent Huge Pages

Transparent Huge Pages (THP) can degrade database performance and should be disabled via a custom init script.

### 2.1 Create the init script

```bash
sudo nano /etc/init.d/disable-transparent-hugepages
```

Paste the following content:

```bash
#!/bin/sh
### BEGIN INIT INFO
# Provides:          disable-transparent-hugepages
# Required-Start:    $local_fs
# Required-Stop:
# Default-Start:     2 3 4 5
# Default-Stop:      0 1 6
# Short-Description: Disable Linux transparent huge pages
# Description:       Disable Linux transparent huge pages, to improve
#                    database performance.
### END INIT INFO
case $1 in
  start)
    if [ -d /sys/kernel/mm/transparent_hugepage ]; then
      thp_path=/sys/kernel/mm/transparent_hugepage
    elif [ -d /sys/kernel/mm/redhat_transparent_hugepage ]; then
      thp_path=/sys/kernel/mm/redhat_transparent_hugepage
    else
      return 0
    fi
    echo 'never' > ${thp_path}/enabled
    echo 'never' > ${thp_path}/defrag
    unset thp_path
    ;;
esac
```

### 2.2 Make it executable

```bash
sudo chmod 755 /etc/init.d/disable-transparent-hugepages
```

### 2.3 Register it to run at startup

```bash
sudo update-rc.d disable-transparent-hugepages defaults
```

### 2.4 Restart the server

> ⚠️ **Action required:** Restart the server for the THP change to take effect.

```bash
sudo reboot
```

---

## 3. Install NTP

Install the Network Time Protocol service to keep server clocks synchronized (required for a healthy CDP cluster).

```bash
sudo apt-get install ntp -y
```

---

## 4. Set Swappiness

Reduce the kernel's tendency to swap memory to disk, improving performance for database/cluster workloads.

```bash
# Check the current value
sudo sysctl -a | grep vm.swappiness

# Set swappiness to 1 (minimal swapping)
sudo sysctl vm.swappiness=1

# Persist the setting across reboots
echo 'vm.swappiness=1' | sudo tee --append /etc/sysctl.conf
```

---

## 5. Install psycopg2 Package

Install Python 2, pip, and the `psycopg2` PostgreSQL adapter (required by Cloudera Manager).

```bash
# Install Python
sudo apt install python -y

# Bootstrap pip for Python 2.7
curl 'https://bootstrap.pypa.io/pip/2.7/get-pip.py' > get-pip.py && sudo python get-pip.py

# Install the specific psycopg2 version
sudo pip install psycopg2==2.7.5 --ignore-installed
```

---

## 6. Create AMI (Snapshot)

> **Checkpoint:** Before installing Cloudera Manager, create an AMI (Amazon Machine Image) of the server as a backup/rollback point.
>
> - Uncheck the **"reboot image"** option when creating the AMI.

---

## 7. Install and Start Cloudera Manager

Download and run the Cloudera Manager 7.4.4 installer.

```bash
# Download the installer
wget https://archive.cloudera.com/cm7/7.4.4/cloudera-manager-installer.bin

# Make it executable
chmod u+x cloudera-manager-installer.bin

# Run the installer
sudo ./cloudera-manager-installer.bin
```

---

## Notes

- Steps should be executed in order, as later steps (e.g., psycopg2, CM installer) depend on earlier system-level configuration (THP, swappiness, NTP).
- A server restart is required after configuring Transparent Huge Pages, before proceeding further.
- Always snapshot/AMI the server prior to running the Cloudera Manager installer, in case a rollback is needed.
