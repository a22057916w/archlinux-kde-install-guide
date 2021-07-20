## Synopsis
My personal Arch Linux configuration on my notebook. Feel free to try it.

| Item | Detail |
| - | :---: |
| Model | ASUS ZenBook UX501J |
| Kernel | 5.12.8-arch1-1 |
| Disk Partition | [LVM](https://opensource.com/business/16/9/linux-users-guide-lvm) |
| Bootloader | GRUB |
| Wireless | plasma-nm |
| Graphics | xf86-video-nouveau |
| Audio | PulseAudio |

## Pre-installation
This guide is based on `UEFI` system.
Download the [latest boot image](https://archlinux.org/download/) and make a Live USB. To BIOS setting, disable the Secure Boot Control, the function will be automatically maintained by Arch Linux Kernel after completing the installation. Then boot with the Live USB.


Make sure you have booted in UEFI mode by checking efi directory
```
ls /sys/firmware/efi/efivars
```

Ensure your network interface is listed and enabled
```
ip link
ping archlinux.org
```

Ensure the system clock is accurate
```
timedatectl set-ntp true
```
To check the service status, use ```timedatectl status```.

## Partition with LVM
Use `cfdisk`, `cgdisk`, `fdisk` or whatever tools you like to partition according to the [office guide](https://wiki.archlinux.org/title/Installation_guide#Partition_the_disks). However, I suggest part the root for 40G at least if one tries to install [KDE Plasma](https://kde.org/). Note that we are refering to `UEFI with GPT`. 

After completing all steps, my `lsblk` output is as follow:
```
NAME         MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sda            8:0    0 119.2G  0 disk 
├─sda1         8:1    0   512M  0 part /boot/EFI
├─sda2         8:2    0     2G  0 part [SWAP]
└─sda3         8:3    0 116.7G  0 part 
  ├─vg1-root 254:0    0    50G  0 lvm  /
  └─vg1-home 254:1    0  66.7G  0 lvm  /home
sdb            8:16   0 931.5G  0 disk 
└─sdb1         8:17   0 931.5G  0 part /data

```

Create the LVM on root and home partitions as following steps:
```
pvcreate /dev/sda3
pvdisplay

vgcreate vg1 /dev/sda3
vgdisplay

lvcreate -L 50G -n root vg1
lvcreate -l 100%FREE -n home vg1
```
### Format the partitions
Once the partitions have been created, each newly created partition must be formatted with an appropriate [file system](https://wiki.archlinux.org/title/File_systems).
```
mkfs.fat -F32 /dev/sda1           --> EFI formation 
mkswap /dev/sda2                  --> swap initalization

mkfs.ext4 /dev/vg1/root
mkfs.ext4 /dev/vg1/home
mkfs.ext4 /dev/sdb1
```

### Mount the file systems
Mount the root volume to /mnt. Then create any remaining mount points (such as /mnt/home) and mount their corresponding volumes.
```
mount /dev/vg1/root /mnt

mkdir -p /mnt/home
mount /dev/vg1/home /mnt/home

mkdir -p /mnt/boot/EFI
mount /dev/sda1 /mnt/boot/EFI

swapon /dev/sda2                  --> enable swap session
```
Check `lsblk` to ensure everything is properly set.

## Base Installation
Install essential packages. Use the `pacstrap script` to install the base package, Linux kernel and firmware for common hardware:
```
pacstrap /mnt base linux linux-firmware base-devel linux-headers intel-ucode lvm2 vim
```
Also, we install the additional packages such like:
* userspace utilities for the management of file systems that will be used on the system
* specific firmware for other devices not included in linux-firmware. 
* packages for [microcode](https://wiki.archlinux.org/title/Microcode) updates, `intel-ucode` or `amd-ucode`.
* utilities for accessing RAID or LVM partitions.
* a text editor.

## Configure the system
### Fstab
Generate an fstab file (use -U or -L to define by UUID or labels, respectively):
```
genfstab -U /mnt >> /mnt/etc/fstab
```
`cat /mnt/etc/fstab` to check if it works, mine is like:
```
# <file system> <dir> <type> <options> <dump> <pass>
# /dev/mapper/vg1-root
UUID=66ddf5be-051c-4e31-9774-34c14b2c8d9f       /               ext4            rw,relatime     0 1

# /dev/mapper/vg1-home
UUID=31af8d33-2f65-4b6e-80f8-3345a2699af1       /home           ext4            rw,relatime     0 2

# /dev/sda1
UUID=A289-1408          /boot/EFI       vfat            rw,relatime,fmask=0022,dmask=0022,codepage=437,iocharset=ascii,shortname=mixed,utf8,errors=remount-ro     0 2

# /dev/sdb1
UUID=fcb0c78c-6d9d-4f36-825f-2da2ab50064f       /data           ext4            rw,relatime     0 2
```

### Chroot
Change root into the new system:
```
arch-chroot /mnt
```
### Time zone
Set the time zone:
```
ln -sf /usr/share/zoneinfo/Asia/Taiwan /etc/localtime
```
Run hwclock to generate /etc/adjtime:
```
hwclock --systohc
```
run `date` to check status.

### Configuring Locale
Uncomment the locales you are going to use in `/etc/locale.gen`. Then run:
```
locale-gen
echo LANG=en_US.UTF-8 >> /etc/locale.conf
```

### Network configuration
Create the hostname file:
```
echo "myhostname" >> /etc/hostname
```
Add matching entries to hosts:
```
vim /etc/hosts
```
```
127.0.0.1	localhost
::1		    localhost
127.0.1.1	myhostname.localdomain myhostname
```

If the system has a permanent IP address, it should be used instead of 127.0.1.1.

Complete the network configuration for the newly installed environment, that may include installing suitable `network management` software.
```
pacman -S networkmanager openssh wireless_tools wpa_supplicant netctl dialog dhcpcd
systemctl enable NetworkManager
```

### Initramfs
For [LVM](https://wiki.archlinux.org/title/Install_Arch_Linux_on_LVM#Adding_mkinitcpio_hooks), system encryption or RAID, modify `mkinitcpio.conf`:
```
vim /etc/mkinitcpio.conf
```
Edit the file and insert `lvm2` between block and filesystems like so:
```
HOOKS=(base udev ... block lvm2 filesystems)
```
Then recreate the initramfs image:
```
mkinitcpio -P linux
```

### Change root password.
```
passwd
```

### Bootloader
Choose **GRUB** as a Linux-capable [boot loader](https://wiki.archlinux.org/title/Arch_boot_process#Boot_loader).
```
pacman -S grub efibootmgr 
grub-install --target=x86_64-efi --efi-directory=/boot/EFI --bootloader-id=GRUB
grub-mkconfig -o /boot/grub/grub.cfg
````

## Reboot
Exit the chroot environment by typing `exit` or pressing `Ctrl+d`.

Optionally manually unmount all the partitions with `umount -R /mnt`: this allows noticing any "busy" partitions, and finding the cause with [fuser](https://man.archlinux.org/man/fuser.1).

Finally, restart the machine by typing `reboot`: any partitions still mounted will be automatically unmounted by systemd. Remember to login into the new system with the root account.

## Post-Installation
### Add a user.
```
useradd -m -G wheel -s /bin/bash <username>
passwd <username>
```

### Setup sudo.
```
EDITOR=vim visudo
```
Then uncomment `%wheel ALL=(ALL) ALL`.

Reboot and check if everything works.
```
reboot
```
## Graphical User Interface
### Display server
[Xorg](https://wiki.archlinux.org/title/Xorg) is the public, open-source implementation of the [X Window System](https://en.wikipedia.org/wiki/X_Window_System) (commonly X11, or X). It is required for running applications with graphical user interfaces (GUIs), and the majority of users will want to install it.<br>
[Wayland](https://wiki.archlinux.org/title/Wayland) is a newer, alternative display server protocol and the Weston reference implementation is available.

Here I would install **Xorg** since the vast majority of native applications that exist were written for Xorg now.
```
pacman -S xorg-server xorg-apps xort-xinit
echo "exec startkde" >> ~/.xinitrc
```
One can find the diffrence between Xorg and Wayland by refering to the [Xorg, X11, Wayland? Linux Display Servers And Protocols Explained](https://linuxiac.com/xorg-x11-wayland-linux-display-servers-and-protocols-explained/)

### Display drivers
Install the appropriate driver for [AMD or NVIDIA](https://wiki.archlinux.org/title/Xorg#Driver_installation) products.
```
pacman -S xf86-video-nouveau  --> Nvidia Card Driver
```
### Desktop Environment - KDE Plasma
In Arch Linux, you can install Plasma 5 in three ways:
* plasma-desktop                --> minimun weight
* plasma 
* plasma-meta

I would install a medium weight `plasma` since it contains the most essential packages and is friendly for new beginners of Linux.
```
pacman -S plasma
```

### Install some key applications
Still, we will need to install some essential utilities. Here are the ones I have chosen:
* Web Browser: I would using **firefox** here.
* Network Manager: Kde has a package named **plasma-nm** which is included in `plasma`, that one can use to connect to a network (Wifi/Ethernet).
* Audio: `plasma` also include a package named **plasma-pa**, which is [PulseAudio](https://linuxhint.com/guide_linux_audio/) integrating for Plasma desktop.
* File Manager: **Dolphin** is the file manager that I prefer to install.
* Terminal: As for terminal, I would install **Konsole**. It is the default terminal app for KDE.
```
pacman -S firefox dolphin konsole spectacle
```
Then, configure the `networkmanager` for plasma-nm. Disable the original connection. For example, I used `dhcpcd` for the Ethernet connection.
```
systemctl stop dhcpcd
systemctl disable dhcpcd
```
Enable or restart the `NetworkManager`
```
systemctl enable NetworkManager
systemctl start NetworkManager
```

### Display manager
Most desktop environments include a [display manager](https://wiki.archlinux.org/title/Display_manager) for automatically starting the graphical environment and managing user logins.

[SDDM](https://wiki.archlinux.org/title/SDDM) is recommended for KDE Plasma.
```
pacman -S sddm
systemctl enable sddm
```
Reboot and login to the system. You should now have a GUI on Arch Linux, similar to the following picture.<br><br>
![](https://github.com/a22057916w/archlinux-kde-install-guide/blob/main/.meta/desktop.png)

### Additional applications
Apart from the essential packages that I have mentioned, you might want to install some other programs that you will need. Here are the ones I have chosen:
* Screenshot Capturer: **Spectacle** is a utillity for KDE.
* Video Player : I find **VLC** the best open source video player on both Linux and Windows.
* Photo Viewer : **digiKam** is mainly developed for KDE, but is a little heavy since it supports many features like editor, organizer, etc.
* Text Editor : **Kate** works good on KDE and has multi-tab support.
* [AUR Helper](https://wiki.archlinux.org/title/Arch_User_Repository) : I used **yay** and refered to [How to Install Yay AUR Helper in Arch Linux and Manjaro](https://www.tecmint.com/install-yay-aur-helper-in-arch-linux-and-manjaro/) for installation.
```
pacman -S spectacle vlc digikam kate
```
Kudos !! After installing the above packages and yay, you can now start to customized your Arch Linux.

## 中文化

### 輸入法框架(Input Method Framework)
Fcitx5 is the successor of Fcitx, a lightweight input method framework aimed at providing environment-independent language support for Linux.
```
pacman -S fictx5
```
Then install a GUI configuration tool for convenient.
```
pacman -S fictx5-configtool
```
Set environment variables for IM modules
Define the environment variables to register the input method modules. Without these variables, applications may fallback to XIM protocol.

As a general recommendation, define the following environment variables in `~/.pam_environment`. This file will be read by the pam_env module for all logins, including both X11 and Wayland sessions.

```
GTK_IM_MODULE DEFAULT=fcitx
QT_IM_MODULE  DEFAULT=fcitx
XMODIFIERS    DEFAULT=\@im=fcitx
```
or edit `/etc/environment `and add the above lines:

輸入法(Input Method)
```
pacman -S fcitx5-chewing
```

字體字形
```
pacman -S ttf-android adobe-source-han-sans-tw-fonts opendesktop-fonts
```
```
reboot
```

### Resource
* [(Other)UEFI? BIOS? Legacy? 淺談主機板UEFI觀念與迷思(轉錄) | by Ryan Lu | AI反斗城 | Medium](https://medium.com/ai%E5%8F%8D%E6%96%97%E5%9F%8E/other-uefi-bios-legacy-%E6%B7%BA%E8%AB%87%E4%B8%BB%E6%A9%9F%E6%9D%BFuefi%E8%A7%80%E5%BF%B5%E8%88%87%E8%BF%B7%E6%80%9D-%E8%BD%89%E9%8C%84-dc86f61b85bd)


## BACKUP and RESTORE

This is a "one-liner" exemple, you will have to add a lot of exceptions to the one command. Here is what I do.
```
sudo su
cd /
tar -cvpzf backup.tgz --exclude=/backup.tgz --exclude=/proc 
--exclude=/tmp --exclude=/mnt --exclude=/dev --exclude=/sys / 
```
will backup your root and exclude ALL mounted partitions, like /media (you do not want to backup external devices) and you absolutely do NOT want to backup something like /dev , /tmp, or /proc etc. 

In above method the backup are stored in /. It will be bettter to store it on an external media later; you can then put a directory in front of the backup.tgz.

* backup.tgz is the backup.
* The ```--exclude``` will prevent the actual backup to be backupped.
* cvpzf: create, verbose, preserve permissions, compress(gzip), use a file.

Restoring would be
```
sudo su
cd /
tar -xvzf backup.tgz
```
The ```-x``` stands for decompressoing or extracting file.
### Resoruce
* [tar(1) - Linux manual page](https://man7.org/linux/man-pages/man1/tar.1.html)
* [hard drive - How would I use tar for full backup and restore with system on SSD and home on HDD? - Ask Ubuntu](https://askubuntu.com/questions/524418/how-would-i-use-tar-for-full-backup-and-restore-with-system-on-ssd-and-home-on-h?newreg=fcaa6c1531e344a5831b547eef344fa4)
* [鳥哥的 Linux 私房菜 -- 淺談備份策略](http://linux.vbird.org/linux_basic/0580backup.php)
