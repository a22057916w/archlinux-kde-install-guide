## Synopsis

| Item | Detail |
| --- | --- |
| Model | ASUS ZenBook UX501J |
| Kernel | 5.12.8-arch1-1 |
| Disk Encryption | LVM |
| Bootloader | ? |
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
