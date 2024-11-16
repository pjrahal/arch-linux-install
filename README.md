# Arch Linux Documentation
## Philip Rahal
## 15 November 2024

# Step 1: Download the ISO
To download the ISO, I navigated first to the Arch Linux Wiki, visited the Download page, scrolled to the United States Mirror section where I found the link to ISOs from the University of Arizona.
I then downloaded the most recent ISO from that link (https://mirror.arizona.edu/archlinux/iso/2024.10.01/).

# Step 2: Create the VM
To create the VM, I opened VirtualBox and selected "New." I named my VM "Arch Linux VM" and selected the ISO from my Downloads folder.
I then gave the VM 16GB of memory (out of my PC's 64GB) amd 8 CPUs (out of my 32). From my 2TB SSD, I created a 64GB Virtual Hard Disk.
I did not enable EFI or Pre-allocate the Full Size of the Virtual Hard Disk.

# Step 3: Installing Arch Linux
### After starting the VM, I selected my keyboard layout by running the command:
loadkeys us

### I verified my internet connection by running:
ping -c 3 archlinux.org
This command returned the following output:

PING archlinux.org (95.217.163.246) 56(84) bytes of data.
64 bytes from archlinux.org (95.217.163.246): icmp_seq=1 ttl=255 time=127 ms
64 bytes from archlinux.org (95.217.163.246): icmp_seq=2 ttl=255 time=126 ms
64 bytes from archlinux.org (95.217.163.246): icmp_seq=3 ttl=255 time=126 ms

---archlinux.org ping statistics ---
3 packets transmitted, 3 recieved, 0% packet loss, time 2083ms
rtt min/avg/max/mdev = 125.944/126.348/127.107/0.536 ms

### I then updated my system clock:
timedatectl set-ntp true

### I identified my disk(s) to be partitioned using the following command:
fdisk -l
Which returned the output:

Disk /dev/sda: 64 GiB, 68719476736 bytes, 134217728 sectors
Disk model: VBOX HARDDISK
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes

Disk /dev/loop0: 794.41 MiB, 833007616 bytes, 1626968 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes

According to the Arch Linux installation guide, outputs ending in "loop" can be ignored. Thus, the disk to be partitioned will be /dev/sda.

### Using fdisk to partition the disk:
To begin the partition, I will use the command:
fdisk /dev/sda
to partition the disk identified in the previous step.

The following commands were used in fdisk to properly configure the partition. 
To create a new partition:
n
[Enter]
[Enter]
[Enter]
+1G

t
[Enter]
a

n
[Enter]
[Enter]
[Enter]
+4G

t
[Enter]
82

n
[Enter]
[Enter]
[Enter]
[Enter]

p
w

These commands yield the following partition table:
Device    Boot        Start          End        Sectors    Size    Id    Type
/dev/sda1              2048      2099199        2097152      1G    EF    EFI (FAT-12/16/32)
/dev/sda2           2099200     10487807        8388608      4G    82    Linux swap / Solaris
/dev/sda3          10487808    134317727      123729920     59G    83    Linux

### Formatting and Mounting Partitions
To format Boot partition:
mkfs.fat -F32 /dev/sda1

To format Root partition:
mkfs.ext4 /dev/sda3

Initializing and Enabling Swap:
mkswap /dev/sda2
swapon /dev/sda2

Mounting the Root partition:
mount /dev/sda3 /mnt

Creating and Mounting the Boot Directory:
mkdir /mnt/boot
mkdir -p /mnt/boot/efi
mount /dev/sda1 /mnt/boot/efi

### Installing Essential Packages
To install essential packages, I used the command:
pacstrap /mnt base linux linux-firmware

### Generate fstab
To generate an fstab file:
genfstab -U /mnt >> /mnt/etc/fstab

### Changing Root into New System
To change Root into the new system:
arch-chroot /mnt

### Setting Time Zone
The following commands were used to set my time zone:
ln -sf /usr/share/zoneinfo/USA/Chicago /etc/localtime
hwclock --systohc

### Localization
Edit locale.gen:
nano /etc/locale.gen
This command gave an error, nano was not already installed.
To fix this, I installed nano:
pacman -S nano

Generate Locales:
locale-gen

Set Language:
echo "LANG=en_US.UTF-8" > /etc/locale.conf

### Network Configuration
Setting my hostname:
echo "philip-arch" > /etc/hostname

Editting Hosts file:
nano /etc/hosts

Added the following lines:
127.0.0.1   localhost
::1         localhost
127.0.1.1   philip-arch.localdomain philip-arch

### Set my Password
Set my password using:
passwd

### Installing Bootloader
Installing grub:
pacman -S grub efibootmgr

Installing to Disk:
grub-install --target=x86_64-efi --efi-directory=/mnt/boot/efi --bootloader-id=GRUB

Generating grub config:
grub-mkconfig -o /boot/grub/grub.cfg

After installing Bootloader, the system must be rebooted with the following commands:
exit
umount -R /mnt
reboot

### Network Configuration
After rebooting, I used the following command to configure my network:
systemctl enable systemd-networkd
systemctl start systemd-networkd

nano /etc/systemd/network/20-wired.network
Added the following lines:
[Match]
Name=enp0s3

[Network]
DHCP=yes

systemctl restart systemd-networkd

networkctl status enp0s3

systemctl start systemd-resolved
systemctl enable systemd-resolved
ln -sf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf
ping -c 3 archlinux.org

The last command returned the expected result of a successful ping.

### Installing a GUI Desktop Environment
I used the following command to install Xorg:
pacman -Sy xorg
I chose the default options for installation.

Installing LXQt:
pacman -S lxqt
Again choosing the default install option.

Installing SDDM as my Display Manager:
pacman -S sddm
systemctl enable sddm

### Creating Users
To create users and give sudo permissions, I first had to install sudo:
pacman -S sudo

I then edited the sudoers file:
EDITOR=nano visudo
I uncommented the line "%wheel ALL=(ALL:ALL) ALL" to allow members of group wheel to execute any command.

I then used the following commands to create the new users:

useradd -m -G wheel philip
passwd philip

useradd -m -G wheel justin
passwd justin
chage -d 0 justin   # Force password change on first login

useradd -m -G wheel codi
passwd codi
chage -d 0 codi     # Force password change on first login

### Installing an Alternative Shell
I chose to use Zsh as my alternate shell, installed with the following command:
pacman -S zsh

I then used the following commands to make Zsh the default shell for all users:
chsh -s /bin/zsh
chsh -s /bin/zsh philip
chsh -s /bin/zsh justin
chsh -s /bin/zsh codi

Starting Zsh:
zsh

### Installing SSH
I used the following commands to install SSH and enable and start the service:
pacman -S openssh
systemctl enable sshd
systemctl start sshd

### Adding Color Coding to my Terminal
For Zsh, the following file must be edited:
nano ~/.zshrc
The following lines were added for color coding:
autoload -U colors && colors
PROMPT='%F{yellow}%n@%m%f %F{blue}%~%f %# '

For bash:
nano ~/.bashrc
The following line was added:
force_color_prompt=yes

#### Adding Aliases to Shell Configuration
Editing the zsh config file:
nano ~/.zshrc
The following lines were added:
alias ll='ls -la --color=auto'
alias gs='git status'
alias update='sudo pacman -Syu'

### Sourcing the Config File
source ~/.zshrc

### Rebooting to GUI
To reboot to the GUI, the following steps are needed:
reboot (to reboot the system)
