# Project 1: Arch Linux Installation  
**Author:** Alexandra Hernandez Gomez  
**Revision Date:** November 2, 2025  

---

## What  
For this project, I created my own Arch Linux virtual machine from scratch using VMware Fusion on macOS, following the official Arch Wiki. Upon this I also made sure the system was functional with the following: an LXQt desktop environment, two users with sudo access, Zsh as the default shell, SSH, terminal color coding, and personalized aliases, and full boot into the GUI automatically.  

---

## How  

### Beginning  
I followed the Arch Installation Wiki from start to finish, stopping often to troubleshoot and make sure I understood each step before moving on.  

---

## 1. Pre-Installation  

### 1.1 Acquire an installation image  
I downloaded the latest image directly from the Arch Linux download page. I picked Taiwan – archlinux.tw because it offered both an `.iso` and `.sig` file for verification.  

Files downloaded:  
archlinux-2025.10.01-x86_64.iso
archlinux-2025.10.01-x86_64.iso.sig

Then verified the checksum to make sure the ISO wasn’t corrupted:  

openssl sha256 archlinux-2025.10.01-x86_64.iso


Output:  
SHA256(archlinux-2025.10.01-x86_64.iso)=86bde4fa571579120190a0946d54cd2b3a8bf834ef5c95ed6f26fbb59104ea1d



This matched the official SHA256 exactly.  

Inside VMware, I created a new virtual machine:  
- Selected “Ubuntu 64-bit” (this uses UEFI defaults)  
- Gave it 20 GiB of storage and 2 GB of RAM  
- Attached the Arch ISO as the CD drive  

---

### 1.4 Boot the live environment  
Booted successfully into the Arch ISO and landed at `root@archiso`.  

---

### 1.6 Verify the boot mode  
To make sure the VM was booted in UEFI mode, I ran:  

cat /sys/firmware/efi/fw_platform_size

It returned `64`, confirming UEFI (x86_64).  

---

### 1.7 Connect to the internet  
ip link
ping -c 10 8.8.8.8

Ping worked fine — network connection was up and stable.  

---

### 1.9 Partition the disks  
Target disk: `/dev/sda` (20 GiB).  
I used fdisk to create a GPT table with:  
- `/dev/sda1` → EFI system partition (512 MiB, type EF00)  
- `/dev/sda2` → Linux root (ext4, remaining space)  

Formatted them:  
mkfs.fat -F32 /dev/sda1
mkfs.ext4 /dev/sda2


Then mounted them:  
mount /dev/sda2 /mnt
mount --mkdir /dev/sda1 /mnt/boot

---

## 2. Installation  

### 2.1 Select mirrors  
/etc/pacman.d/mirrorlist
vim /etc/pacman.d/mirrorlist
:q


I opened the mirrorlist file using Vim to sort and organize my preferred mirrors.  
After quitting once with `:q`, I re-opened it to make edits manually, cutting and pasting entries using Vim’s `dd` (delete line) and `p` (paste) commands to reorder them.  
My final organized list prioritized reliable and fast U.S.-based mirrors for smoother installations. *(See attached screenshot for my final list.)*

---

### 2.2 Install essential packages  
Installed the base system to `/mnt` using:  
pacstrap -K /mnt base linux linux-firmware vim nano sudo networkmanager

This took a few minutes since it pulled over 160 packages (kernel, firmware, and basic tools).  
Then verified with:  
ls /mnt

Output showed `bin`, `boot`, `etc`, `usr`, `var`, confirming a working base install.  

---

## 3. Configure the system  

### 3.1 Generate fstab  
genfstab -U /mnt >> /mnt/etc/fstab
cat /mnt/etc/fstab


---

### 3.2 Chroot into the system  
arch-chroot /mnt

---

### 3.3 Time  
ln -sf /usr/share/zoneinfo/America/Chicago /etc/localtime
hwclock --systohc
systemctl enable systemd-timesyncd


You can confirm it’s working with:  
timedatectl status

---

### 3.4 Localization  
nano /etc/locale.gen


Uncommented the “#” in front of:  
en_US.UTF-8 UTF-8

Then ran:  
locale-gen
echo "LANG=en_US.UTF-8" > /etc/locale.conf
cat /etc/locale.conf

Output confirmed:  
LANG=en_US.UTF-8


---

### 3.5 Network configuration  
Set the hostname and hosts file:  
echo "alexandra-arch" > /etc/hostname
nano /etc/hosts

Contents:  
127.0.0.1 localhost
::1 localhost
127.0.1.1 alexandra-arch


Enabled networking:  
systemctl enable NetworkManager
systemctl start NetworkManager
systemctl status NetworkManager

The last two showed “ignored in chroot,” which is expected.  

---

### 3.6 Initramfs  
Generated initramfs images:  
mkinitcpio -P

Output:  
Both default and fallback images were created:  
/boot/initramfs-linux.img
/boot/initramfs-linux-fallback.img

---

### 3.7 Root password  
passwd

Prompted to enter and confirm a password.  

---

### 3.8 Boot loader  
Verified UEFI Mode:  
ls /sys/firmware/efi/efivars

Confirmed multiple EFI variable files.  

Checked EFI Partition Mount:  
lsblk


Installed GRUB and supporting tools:  
pacman -S grub efibootmgr os-prober


Installed GRUB to the EFI directory:  
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=GRUB


Generated configuration file:  
grub-mkconfig -o /boot/grub/grub.cfg

Detected Linux kernel and initramfs images successfully.  

---

## 4. Reboot  

Exited the chroot environment:  
exit

Unmounted all partitions:  
umount -R /mnt

Rebooted into the new system:  
reboot


After boot, logged in with root credentials.  
Confirmed environment with:  
hostnamectl


---

## 5. Post-Installation  

### 5.1 Update packages and install core tools  
pacman -Syu
pacman -S sudo nano git curl wget unzip zip base-devel

Downloads timed out once — fixed by rerunning reflector and refreshing with `pacman -Syy`.

---

### 5.2 Create user accounts with sudo  
useradd -m -G wheel -s /bin/bash alexandra
passwd alexandra

useradd -m -G wheel -s /bin/bash codi
passwd codi

EDITOR=nano visudo

Enabled line:  
%wheel ALL=(ALL:ALL) ALL


Verified:
su - alexandra
sudo ls /
su - codi
sudo ls /

Note: Password confusion fixed by resetting with `passwd alexandra`.

---

### 5.3 Install LXQt desktop + LightDM  
sudo pacman -Syu
sudo pacman -S xorg xorg-xinit lxqt lightdm lightdm-gtk-greeter
sudo systemctl enable lightdm
sudo systemctl set-default graphical.target
systemctl get-default
sudo reboot


On reboot, LightDM GUI appeared and LXQt worked.  
Note: Background turned black after one reboot — default theme.

---

### 5.4 Install Zsh and make it the default shell  
sudo pacman -S zsh
which zsh # -> /usr/bin/zsh

Fix if chsh fails:
echo /usr/bin/zsh | sudo tee -a /etc/shells

Set for both users:
sudo chsh -s /usr/bin/zsh alexandra
sudo chsh -s /usr/bin/zsh codi


Verify:
getent passwd alexandra codi




---

### 5.5 Install and enable SSH  
sudo pacman -S openssh
sudo systemctl enable sshd
sudo systemctl start sshd
systemctl status sshd

---

### 5.6 Add color coding to the terminal  
Removed malformed alias, then re-added correct color aliases:  
sed -i '/ls=--color/d' ~/.zshrc

echo "alias ls='ls --color=auto'" >> ~/.zshrc
echo "alias grep='grep --color=auto'" >> ~/.zshrc
echo "alias fgrep='fgrep --color=auto'" >> ~/.zshrc
echo "alias egrep='egrep --color=auto'" >> ~/.zshrc

source ~/.zshrc

---

### 5.7 Add aliases  
echo "alias ll='ls -alF'" >> ~/.zshrc
echo "alias la='ls -A'" >> ~/.zshrc
echo "alias cls='clear'" >> ~/.zshrc
echo "alias update="sudo pacman -Syu"" >> ~/.zshrc
source ~/.zshrc

Now I can type `ll` or `update` without remembering long commands.
