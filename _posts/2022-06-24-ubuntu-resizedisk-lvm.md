---
title: Ubuntu Resize Disk LVM
date: 2022-06-24
categories: [homelab, machines, virtual-machines]
tags: [servers, ubuntu]
---

# Ubuntu Resize Disk LVM

### When you add more space to your disk in ex. Proxmox, you need to resize it in Ubuntu.
<br>

### Here are the commands that can help you achieve that:

## First check your disk
``` bash
lsblk
```
You should get something like this:
```
NAME                      MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                         8:0    0    1T  0 disk
├─sda1                      8:1    0    1M  0 part
├─sda2                      8:2    0    1G  0 part /boot
└─sda3                      8:3    0    1T  0 part
  └─ubuntu--vg-ubuntu--lv 253:0    0   31G  0 lvm  /
```

For me, I had to resize the sda3 partition, on the sda disk. Specifically the 'ubuntu--vg-ubuntu--lv' LVM. 

## To resize the sda3 partition use:
```bash
sudo parted
```
```bash
resizepart 3 100%
```
```bash
quit
```
You should get something like this:
```
GNU Parted 3.3
Using /dev/sda
Welcome to GNU Parted! Type 'help' to view a list of commands.
(parted) resizepart 3 100%
(parted) quit
Information: You may need to update /etc/fstab.
```
## Next, we need to extend the logical volume using:
```bash
sudo lvextend -r -l +100%FREE /dev/mapper ubuntu--vg-ubuntu--lv
```
You should see something like this:
```bash
$ sudo lvextend -r -l +100%FREE /dev/mapper/ubuntu--vg-ubuntu--lv
  Size of logical volume ubuntu-vg/ubuntu-lv changed from <31.00 GiB (7935 extents) to 1.03 TiB (270079 extents).
  Logical volume ubuntu-vg/ubuntu-lv successfully resized.
resize2fs 1.45.5 (07-Jan-2020)
Filesystem at /dev/mapper/ubuntu--vg-ubuntu--lv is mounted on /; on-line resizing required
old_desc_blocks = 4, new_desc_blocks = 132
The filesystem on /dev/mapper/ubuntu--vg-ubuntu--lv is now 276560896 (4k) blocks long.
```
## For the final step, you may need to resize the phyisical volume using:
```bash
sudo pvresize /dev/sda3
```
You should see something like this:
```
Physical volume "/dev/sda3" changed
1 physical volume(s) resized or updated / 0 physical volume(s) not resized
```
### Finally, we check if the resize is complete using:
```bash
lsblk
```
You should see something like this:
```
$ lsblk
NAME                      MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                         8:0    0    1T  0 disk
├─sda1                      8:1    0    1M  0 part
├─sda2                      8:2    0    1G  0 part /boot
└─sda3                      8:3    0    1T  0 part
  └─ubuntu--vg-ubuntu--lv 253:0    0    1T  0 lvm  /
```
## If lsblk output shows that the 'ubuntu--vg-ubuntu--lv' LVM has the size you wanted it to resize to, you're done!