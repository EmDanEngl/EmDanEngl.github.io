# Arch Linux Installation Documentation

## Section 1: Preinstallation

### 1.1 Download the ISO

For this step, I downloaded the Arch Linux ISO from the Arizona University mirror.

### 1.2 Verify the ISO signature

After downloading the ISO, I checked its SHA256 hash using PowerShell on Windows.

### 1.3 and 1.4

I skipped these steps because I was installing Arch Linux in a virtual machine.

Instead, I created a new virtual machine in VMware Workstation.

After creating the VM, I needed to make sure it booted in UEFI mode. To do this, I opened the VM settings, went to the Options tab, selected Advanced, and changed the Firmware type from BIOS to UEFI.

### 1.5 Set the console keyboard layout

I skipped this step because I did not need to change the default keyboard layout or console font.

### 1.6 Verify the boot mode

To verify that the system had booted in UEFI mode, I ran:

```bash
cat /sys/firmware/efi/fw_platform_size
```

The command returned `64`, meaning the system was booted in 64-bit UEFI mode.

### 1.7 Connect to the internet

First, I ran:

```bash
ip link
```

This showed the VMware virtual network adapter and the loopback interface.

I did not need to check for wireless blocks because my VM uses VMware's virtual Ethernet adapter instead of connecting directly to a wireless adapter.

To check the IP address assigned to the virtual network adapter, I ran:

```bash
ip addr show ens33
```

I also used the following command to see its IPv4 address:

```bash
ip -4 addr show ens33
```

Lastly, I checked that the internet connection was working by running:

```bash
ping -c 4 archlinux.org
```

### 1.8 Update the system clock

I ran:

```bash
timedatectl
```

This displayed the system clock information. I checked that the system clock was synchronized.

### 1.9 Partition the disks

I identified the available block devices by running:

```bash
fdisk -l
```

I then started the partitioning process with:

```bash
fdisk /dev/sda
```

I used `fdisk` to create a GPT partition table and two partitions.

**First partition: EFI System partition**

1. I entered `g` to create a new GPT partition table.
2. I entered `n` to create a new partition.
3. I accepted the default partition number and first sector.
4. I entered `+500M` for the last sector to make the partition 500 MiB.
5. I entered `t` to change the partition type.
6. I selected `1`, which was the EFI System partition.

**Second partition: Root partition**

1. I entered `n` to create another partition.
2. I selected partition number `2`.
3. I accepted the default first and last sectors so that it would use the remaining disk space.

I then entered:

```text
p
```

This displayed the partition table so I could verify that both partitions looked correct.

Once I was finished, I entered:

```text
w
```

This wrote the partition table changes to the disk.

### 1.10 Format the partitions

First, I formatted `/dev/sda1` as FAT32

I ran:

```bash
mkfs.fat -F 32 /dev/sda1
```

Next, I formatted the root partition as ext4:

```bash
mkfs.ext4 /dev/sda2
```

### 1.11 Mount the file systems

First, I mounted the root partition:

```bash
mount /dev/sda2 /mnt
```

This mounted the root filesystem at `/mnt`.

Next, I mounted the EFI partition:

```bash
mount --mkdir /dev/sda1 /mnt/boot
```

---

## Section 2: Installation

### 2.1 Select the mirrors

I skipped this step because it was not necessary.

### 2.2 Install essential packages

I installed the essential packages using:

```bash
pacstrap -K /mnt base linux linux-firmware
```

---

## Section 3: Configure the System

### 3.1 Fstab

The fstab file tells Arch which partitions and filesystems to mount automatically when the system starts.

I generated the file using:

```bash
genfstab -U /mnt >> /mnt/etc/fstab
```

I verified that the entries were created correctly using:

```bash
cat /mnt/etc/fstab
```

I confirmed that both partitions were listed.

### 3.2 Chroot

Next, I changed root into the new system:

```bash
arch-chroot /mnt
```

### 3.3 Time zone

I configured the time zone using:

```bash
ln -sf /usr/share/zoneinfo/America/Chicago /etc/localtime
```

### 3.4 Localization

First, I checked the locale configuration file to see if the English UTF-8 locale was enabled:

```bash
grep -n 'en_US.UTF-8' /etc/locale.gen
```

The result showed that it was commented out.

I wanted to use Nano to edit the file, but it was not installed yet. I installed it using:

```bash
pacman -S nano
```

I then opened the locale configuration file:

```bash
nano /etc/locale.gen
```

I searched for:

```text
#en_US.UTF-8 UTF-8
```

I removed the `#` at the beginning of the line to enable the locale, then saved and exited Nano.

Next, I generated the locale:

```bash
locale-gen
```

I configured the system to use the English UTF-8 locale by running:

```bash
echo "LANG=en_US.UTF-8" > /etc/locale.conf
```

I verified the file using:

```bash
cat /etc/locale.conf
```

### 3.5 Network configuration

I chose `my-arch-vm` as the hostname for my virtual machine.

I created the hostname file using:

```bash
echo "my-arch-vm" > /etc/hostname
```

Next, I installed NetworkManager:

```bash
pacman -S networkmanager
```

I enabled it to start automatically when the system boots:

```bash
systemctl enable NetworkManager
```

### 3.6 Initramfs

I skipped this step because it was not necessary.

### 3.7 Set the root password

I set the root password using:

```bash
passwd
```

### 3.8 Boot loader

First, I verified that the EFI partition was mounted at `/boot`:

```bash
findmnt /boot
```

I then installed systemd-boot:

```bash
bootctl install
```

I verified that the bootloader files existed by running:

```bash
ls -l /boot/EFI/BOOT /boot/EFI/systemd
```

Next, I found the UUID of my root partition:

```bash
blkid /dev/sda2
```

I used this UUID when creating the boot entry.

I opened the boot entry file using:

```bash
nano /boot/loader/entries/arch.conf
```

I entered the following configuration:

```text
title   Arch Linux
linux   /vmlinuz-linux
initrd  /initramfs-linux.img
options root=UUID=21ababb1-e92f-4b9c-8df6-de128b4b52d1 rw
```

Next, I opened the main systemd-boot configuration file:

```bash
nano /boot/loader/loader.conf
```

I added the following settings:

```text
default arch.conf
timeout 5
editor no
```

---

## Section 4: Assignment Requirements and Customization

### 4.1 Create two user accounts with sudo permissions

First, I installed sudo:

```bash
pacman -S sudo
```

I created my account, `danny`, and added it to the `wheel` group:

```bash
useradd -m -G wheel -s /bin/bash danny
```

I created the second account, `codi`:

```bash
useradd -m -G wheel -s /bin/bash codi
```

I set passwords for both accounts using:

```bash
passwd danny
passwd codi
```

Next, I needed to give the wheel group sudo permissions.

I opened the sudo configuration file using:

```bash
EDITOR=nano visudo
```

I found the following line and removed the `#` at the beginning:

```text
# %wheel ALL=(ALL:ALL) ALL
```

I verified the setting using:

```bash
grep -E '^%wheel' /etc/sudoers
```

### 4.2 Install an alternative shell

I chose Fish because it mostly because it is called fish.

I installed Fish using:

```bash
pacman -S fish
```

I changed the default shell for both accounts:

```bash
chsh -s /usr/bin/fish danny
chsh -s /usr/bin/fish codi
```

### 4.3 Install and enable SSH

I installed SSH using:

```bash
pacman -S openssh
```

I enabled the SSH service so it would start automatically when the system boots:

```bash
systemctl enable sshd
```

### 4.4 Install a desktop environment

I chose XFCE as my desktop environment.

First, I installed Xorg and XFCE:

```bash
pacman -S xorg-server xfce4 xfce4-goodies
```

During installation, Pacman asked me to choose between two providers for JACK. I selected the default option, `jack2`.

It also asked me to choose a font provider. I selected the default option there too.

Next, I installed LightDM and its GTK greeter:

```bash
pacman -S lightdm lightdm-gtk-greeter
```

I enabled LightDM to start automatically:

```bash
systemctl enable lightdm
```

Lastly, I configured the system to boot into graphical mode instead of text-only mode:

```bash
systemctl set-default graphical.target
```

### 4.5 Install a web browser

I installed Firefox so I could use a browser in my desktop environment and access my GitHub Pages documentation.

I ran:

```bash
pacman -S firefox
```

During installation, Pacman asked me to select providers for JACK and fonts. I selected the default options.

### 4.6 Add aliases to the Fish configuration

I switched to my personal user account:

```bash
su - danny
```

I created the Fish configuration directory:

```bash
mkdir -p ~/.config/fish
```

I then created the Fish configuration file:

```bash
nano ~/.config/fish/config.fish
```

I added my aliases into the file:

```fish
if status is-interactive
    # commands to run in interactive sessions can go here

    alias c='clear'
    alias la='ls -lah'
    alias path='string split ":" $PATH'
    alias ..='cd ..'
    alias ...='cd ../..'
    alias update='sudo pacman -Syu'
end
```

After saving the configuration file, I reloaded it without restarting the terminal:

```fish
source ~/.config/fish/config.fish
```

I then tested the aliases and confirmed that they worked.

### 4.7 Add color coding to the terminal

The Fish shell already has color coding in the terminal, so I didn't need to do anything else.

## Section 5: Reboot and Post-Installation Verification

After finishing the initial configuration, I exited the user shell and the chroot environment to return to the Arch installation environment.

I unmounted the installed system:

```bash
umount -R /mnt
```

I then restarted the virtual machine:

```bash
reboot
```

After rebooting, I successfully reached the LightDM graphical login screen and logged in using my `danny` account.

### 5.1 Verify sudo

I opened the XFCE terminal and ran:

```bash
sudo -v
```

I entered my password when prompted. The command completed without errors, confirming that my account had sudo permissions.

### 5.2 Verify internet access

I checked the network connection:

```bash
ping -c 3 archlinux.org
```

The command was successful, confirming that I had internet access.

### 5.3 Verify SSH

I checked the SSH service using:

```bash
systemctl status sshd
```

The service was running successfully.

### 5.4 Verify aliases

I ran:

```fish
alias
```

This displayed my six aliases. I had also tested each of them and confirmed that they worked.

## 5.6 Verify both user accounts

Both accounts were created and added to the wheel group during installation.

---
