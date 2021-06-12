## BACKUP and RESTORE

This is a "one-liner" exemple, you will have to add a lot of exceptions to that one command. Here is what I do.
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
The "x" stands for decompressoing or extracting file.
### Resoruce
* [tar(1) - Linux manual page](https://man7.org/linux/man-pages/man1/tar.1.html)
* [hard drive - How would I use tar for full backup and restore with system on SSD and home on HDD? - Ask Ubuntu](https://askubuntu.com/questions/524418/how-would-i-use-tar-for-full-backup-and-restore-with-system-on-ssd-and-home-on-h?newreg=fcaa6c1531e344a5831b547eef344fa4)
* [linux - List the directories on depthlevel one in tar.gz - Stack Overflow](https://stackoverflow.com/questions/10837287/list-the-directories-on-depthlevel-one-in-tar-gz)
