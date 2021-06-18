## Synopsis

| Item | Detail |
| --- | --- |
| Model | ASUS ZenBook UX501J |
| Kernel | 5.12.8-arch1-1 |
| Disk Encryption | LVM |
| Bootloader | GRUB |
| Wireless | ? |
| Graphics | ? |
| Backlight | ? |
| Audio | ? |

## Pre-installation
Secure Boot Control -> disable

Tip: The installation image uses systemd-boot for booting in UEFI mode and syslinux for booting in BIOS mode. See README.bootparams for a list of boot parameters.

make sure you have booted in UEFI mode by checking efi directory
```
ls /sys/firmware/efi/efivars
```

Ensure your network interface is listed and enabled
```
ip link
ping archlinux.org
```

ensure the system clock is accurate
```
timedatectl set-ntp true
```
To check the service status, use ```timedatectl status```.





## PARTITION
```
pvcreate /dev/sda3
pvdisplay

vgcreate vg1 /dev/sda3
vgdisplay

lvcreate -L 50G -n root vg1
lvcreate -l !))%FREE -n home vg1
```

format the partition
```
mkfs.fat -F32 /dev/sda1
mkswap /dev/sda2

mkfs.ext4 /dev/vg1/root
mkfs.ext4 /dev/vg1/home
mkfs.ext4 /dev/sdb1
```

mount
```
swapon /dev/sda2

mount /dev/vg1/root /mnt
mkdir -p /mnt/home
mount /dev/vg1/home /mnt/home
mkdir -p /mnt/boot
mount /dev/sda1 
```
Generate an fstab file (use -U or -L to define by UUID or labels, respectively):
```
genfstab -U /mnt >> /mnt/etc/fstab
```
check ```cat /mnt/etc/fstab```

Chroot
Change root into the new system:
```
arch-chroot /mnt
```

Time zone
Set the time zone:
```
ln -sf /usr/share/zoneinfo/Asia/Taiwan /etc/localtime
```
Run hwclock(8) to generate /etc/adjtime:
```
hwclock --systohc
```
check ```date```

Configuring Locale
Uncomment the locales you are going to use in `/etc/locale.gen`. Then run:
```
locale-gen
echo LANG=en_US.UTF-8 >> etc/locale.conf
```


Network configuration
Create the hostname file:
```
echo "willylaptop" >> /etc/hostname
```
Add matching entries to hosts(5):
```
vim /etc/hosts
```
```
127.0.0.1	localhost
::1		localhost
127.0.1.1	willylaptop.localdomain	willylaptop
```
If the system has a permanent IP address, it should be used instead of 127.0.1.1.

Complete the network configuration for the newly installed environment, that may include installing suitable `network management` software.
```
pacman -S networkmanager
systemctl enable NetworkManager
```

Initramfs
For `LVM`, system encryption or RAID, modify `mkinitcpio.conf` and recreate the initramfs image:
```
vim /etc/mkinitcpio.conf
```
Edit the file and insert lvm2 between block and filesystems like so:
```
HOOKS=(base udev ... block lvm2 filesystems)
```

```
pacman -S lvm2 linux
mkinitcpio -P linux
```
Bootloader
```
pacman -S grub efibootmgr
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB
grub-mkconfig -o /boot/grub/grub.cfg
````
**might get waring:** os-prober will not be executed to detect other bootable partitions arch


Add a user.
```
useradd -m -G wheel -s /bin/bash willy
passwd willy
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
