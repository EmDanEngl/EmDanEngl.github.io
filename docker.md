# Docker Documentation

## Introduction

For this assignment, I set up a Fedora virtual machine in Proxmox and used Docker Compose to run Uptime Kuma. Uptime Kuma is an application that allows you to monitor websites and check if they are online. I chose Uptime Kuma because it was one of the applications suggested for the assignment.

## 1. Creating and Preparing the Virtual Machine

I created a new virtual machine in Proxmox using Fedora Server 43 without a desktop environment. I gave the VM 2 CPU cores, 2 GB of RAM, and a 32 GB virtual disk. I selected my assigned resource pool, enabled QEMU Guest Agent, and manually configured the network because the private network for the class does not use DHCP.

I created my first snapshot before making changes.

SCREENSHOT!!!!!!!!!!!!!! PROXMOX SNAPSHOTS

After starting the VM and completing the initial setup, I checked the internet connection:

```bash
ping -c 3 google.com
```

I then updated Fedora:

```bash
sudo dnf upgrade --refresh -y
```

After updating, I created another snapshot.

## 2. Installing Docker

I installed Docker using the official Docker repository for Fedora.

First, I installed the repository management tools:

```bash
sudo dnf -y install dnf-plugins-core
```

I added the official Docker repository:

```bash
sudo curl -fsSL https://download.docker.com/linux/fedora/docker-ce.repo -o /etc/yum.repos.d/docker-ce.repo
```

I installed Docker Engine, the command-line tools, containerd, Buildx, and the Docker Compose plugin:

```bash
sudo dnf install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

I enabled and started Docker so it would run automatically:

```bash
sudo systemctl enable --now docker
```

I checked that Docker was enabled:

```bash
systemctl is-enabled docker
```

I also tested the installation:

```bash
sudo docker run --rm hello-world
```

Finally, I checked the Docker Compose version:

```bash
docker compose version
```

SCREENSHOT!!!!!!!!!!!!!! DOCKER INSTALLATION TEST

## 3. Setting Up Uptime Kuma

I created a directory for the application:

```bash
mkdir ~/uptime-kuma
cd ~/uptime-kuma
```

I created a file named `compose.yaml` to configure the application.

### compose.yaml

```yaml
services:
  uptime-kuma:
    image: louislam/uptime-kuma:2
    container_name: uptime-kuma
    volumes:
      - uptime-kuma-data:/app/data
    ports:
      - "3001:3001"
    restart: unless-stopped

volumes:
  uptime-kuma-data:
```

I started the application:

```bash
sudo docker compose up -d
```

I checked that the container was running:

```bash
sudo docker compose ps
```

SCREENSHOT!!!!!!!!!!!!!! DOCKER COMPOSE STATUS

I accessed Uptime Kuma through a browser using:

`http://10.30.56.100:3001`

I created an administrator account and selected SQLite as the database. I also added website monitors to test the application.

SCREENSHOT!!!!!!!!!!!!!! UPTIME KUMA DASHBOARD

## 4. Securing the Virtual Machine

### SSH Key Authentication

I generated an SSH key pair on my Windows computer using PowerShell:

```powershell
ssh-keygen -t ed25519
```

I added my public key to the VM's `authorized_keys` file. I also set the correct permissions:

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

I enabled the SSH service:

```bash
sudo systemctl enable --now sshd
```

I created a configuration file to disable password authentication:

```bash
sudo nano /etc/ssh/sshd_config.d/00-local.conf
```

I made this change in sshd_config

```text
PasswordAuthentication no
```

I checked the configuration for errors:

```bash
sudo sshd -t
```

I verified that password authentication was disabled:

```bash
sudo sshd -T | grep -E 'passwordauthentication|kbdinteractiveauthentication'
```

I restarted SSH to apply the changes:

```bash
sudo systemctl restart sshd
```

I tested the connection using my SSH key. I also tested password-only authentication to make sure it was refused.

SCREENSHOT!!!!!!!!!!!!!! SSH PASSWORD AUTHENTICATION REFUSED

### Firewall

I used Fedora's `firewalld` to configure the firewall.

I allowed port 3001 for Uptime Kuma:

```bash
sudo firewall-cmd --zone=FedoraServer --add-port=3001/tcp --permanent
```

I removed the Cockpit and DHCPv6:

```bash
sudo firewall-cmd --zone=FedoraServer --remove-service=cockpit --permanent
sudo firewall-cmd --zone=FedoraServer --remove-service=dhcpv6-client --permanent
```

I reloaded the firewall:

```bash
sudo firewall-cmd --reload
```

I checked the firewall configuration:

```bash
sudo firewall-cmd --zone=FedoraServer --list-all
```

SCREENSHOT!!!!!!!!!!!!!! FIREWALL CONFIG

### Automatic Security Updates

I installed the automatic update plugin for Fedora:

```bash
sudo dnf install dnf5-plugin-automatic -y
```

I copied the default configuration file:

```bash
sudo cp /usr/share/dnf5/dnf5-plugins/automatic.conf /etc/dnf/automatic.conf
```

I edited the configuration file:

```bash
sudo nano /etc/dnf/automatic.conf
```

Under the `[commands]` section, I set:

```ini
[commands]
upgrade_type = security
apply_updates = true
```

I enabled the automatic update timer:

```bash
sudo systemctl enable --now dnf5-automatic.timer
```

I checked its status:

```bash
systemctl status dnf5-automatic.timer
```

SCREENSHOT!!!!!!!!!!!!!! AUTOMATIC UPDATE TIMER

## 5. Creating Backups

I created a directory to store the backups:

```bash
sudo mkdir -p /var/backups/uptime-kuma
```

I created a backup script named `backup.sh` inside my Uptime Kuma directory.

### backup.sh

```bash
#!/bin/bash

set -e

COMPOSE_FILE="/home/danny/uptime-kuma/compose.yaml"
DATA_DIR="/var/lib/docker/volumes/uptime-kuma_uptime-kuma-data/_data"
BACKUP_DIR="/var/backups/uptime-kuma"

DATE=$(date +%Y%m%d-%H%M%S)

mkdir -p "$BACKUP_DIR"

echo "Stopping Uptime Kuma..."
docker compose -f "$COMPOSE_FILE" stop

echo "Creating backup..."
if ! tar -czf "$BACKUP_DIR/uptime-kuma-$DATE.tar.gz" -C "$DATA_DIR" .; then
    echo "Backup failed. Restarting Uptime Kuma..."
    docker compose -f "$COMPOSE_FILE" start
    exit 1
fi

echo "Restarting Uptime Kuma..."
docker compose -f "$COMPOSE_FILE" start

echo "Removing older backups..."

find "$BACKUP_DIR" -maxdepth 1 -type f -name 'uptime-kuma-*.tar.gz' -printf '%T@ %p\n' |
    sort -nr |
    tail -n +8 |
    cut -d' ' -f2- |
    while IFS= read -r file; do
        rm -f -- "$file"
    done

echo "Backup complete!"
```

The script stops Uptime Kuma, makes a tar file of its data, and starts the application again. It also removes older backups so only the seven newest remain. 

SCREENSHOT!!!!!!!!!!!!!! BACKUP SCRIPT

I made the script executable:

```bash
chmod +x ~/uptime-kuma/backup.sh
```

I ran the script:

```bash
sudo ~/uptime-kuma/backup.sh
```

I checked the backup directory to make sure the archive was created:

```bash
sudo ls -lh /var/backups/uptime-kuma/
```

SCREENSHOT!!!!!!!!!!!!!! BACKUP ARCHIVE LIST

## 6. Proxmox VM Backup

I also created a backup of the entire virtual machine through Proxmox. I opened the VM's **Backup** tab, selected **Backup now**, and used Snapshot mode.

SCREENSHOT!!!!!!!!!!!!!! PROXMOX VM BACKUP

## 7. Testing the Restore Process

To test that my backups worked, I stopped Uptime Kuma:

```bash
sudo docker compose stop
```

I checked that my backup archive contained the application data:

```bash
sudo tar -tzf /var/backups/uptime-kuma/uptime-kuma-20261122-211704.tar.gz | head
```

I deleted the existing application data from the Docker volume:

```bash
sudo find /var/lib/docker/volumes/uptime-kuma_uptime-kuma-data/_data -mindepth 1 -delete
```

I restored the data from my backup archive:

```bash
sudo tar -xzf /var/backups/uptime-kuma/uptime-kuma-20261122-211704.tar.gz -C /var/lib/docker/volumes/uptime-kuma_uptime-kuma-data/_data
```

I started the application again:

```bash
sudo docker compose up -d
```

After opening Uptime Kuma in my browser, I confirmed that my website monitors were still there. This showed that the restore process worked.

## 8. Testing After Reboot

I rebooted the virtual machine to make sure the application would start again:

```bash
sudo reboot
```

After reconnecting, I checked the container status:

```bash
cd ~/uptime-kuma
sudo docker compose ps
```

The Uptime Kuma container was running, and I was able to access the application through my browser.

## Conclusion

For this assignment, I set up a Fedora Server VM, installed Docker, and deployed Uptime Kuma using Docker Compose. I also configured SSH key authentication, the firewall, and automatic security updates. Lastly, I made a backup script, backed up the VM in Proxmox, and tested restoring the application data from a backup.
