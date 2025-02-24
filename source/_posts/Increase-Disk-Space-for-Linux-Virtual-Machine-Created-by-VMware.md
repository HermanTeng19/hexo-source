---
title: Increase Disk Space for Linux Virtual Machine Created by VMware
date: 2021-01-08 17:31:16
categories: [IT, BI]
tags: [Linux, LVM, VMware, Disk management]
toc: true
cover: /img/linuxlvmribbon.jpg
thumbnail: /img/linuxlvmicon.jpg
---

Virtual machine is a major way to setup `Linus` development environment in `Windows PC`. One of the great things about newly Linux distro such as `RHEL, Ubuntu, Centos` etc is that most of them adopt to a LVM (Logical Volume Manager) filesystem which is a natural fit for Linux virtualized system like `VMware` or `VitualBox`.

The main characteristic for LVM is that it is able to adjust filesystem volume dynamically by integrating multiple partitions which feel like one partition on one disk after LVM adjustment, moreover, adding and remove partitions to increase and shrink disk space also become very easy than before and this feature applies virtualized hard drive and makes it very easy to grow the disk space within few steps setup. You might ask what is the point to do that, I would say if you make virtual machine as your main development environment, the disk space is going to run out quickly as time goes by when more and more tools and libraries are installed. I have my VM hard disk initial 20GB by default setting but after I setup all my environment items my root directory only has 300MB space left. 

I will grow disk space with my Linux virtual machine to 40GB:

Before we make our hands dirty, it's necessary to get some basic understanding about the LVM in terms of 3 key things, they are PV (physical volumes), VG (volume groups) and LV (logical volumes). LVM integrates multiple partitions or disks into a big independent partition (VG), then formats it to create multiple logical volumes (LV) to be able to mount filesystem.

<!-- more -->

## Assign more space or partition on virtual machine setting

The first step is that assign space you want on virtual machine console, bear in mind, the host must be shutdown before doing anything change. In this case, we will use WMware as example, the VIrtualBox is similar.

![wmwaresetting.png](/img/screenshots/wmwaresetting.png)

![wmwaresetting2.png](/img/screenshots/wmwaresetting2.png)

The fast and alternative way is use command line:

```powershell
vmware-vdiskmanager.exe -x 40GB "your virtual disk name.vmdk"
```

Things may be tricky here if you were new to virtual machine as above highlight "**Expand increases only the size of a virtual disk. Sizes of partitions and file systems are not affected**", it looks like no that easy to expand the disk space only by `VMWare` setup, because now it only adds more unpartitioned space to the virtual hard disk.

Convert the unpartitioned space into usable filesystem so it can be included within the LVM filesystem.

## LVM configuration to extend new partition to mount filesystem

Boot Linux VM and switch to `root` account. 

We first check out our disk partition status in advanced so that we can see clearly the change before and after, there are many commands in Linux e.g `df -h`, `mount`, here we use `lsblk` which is abbreviation of list block device

![df_h_cmd_bf.png](/img/screenshots/df_h_cmd_bf.png)

```bash
lsblk -ip /dev/sda
```

![lsblkcmd.png](/img/screenshots/lsblkcmd.png)

from return data, we know our total disk space is 40G, new unpartitioned disk `/dev/sda3` with 20G is new added space, now let's see how to create a new partition that takes up 20G space.

### Create a new partition for unallocated space

There are 2 disk partition tools we can utilize to create a new partition in according to your partition table format, `fdisk` is for `MBR` and `gdisk` is for `GPT`. We can use `gdisk` to check our partition format

![gdiskcmd.png](/img/screenshots/gdiskcmd.png)

we can see MBR is the partition table for current system so that we will use `fdisk` to create partition. 

```bash
fdisk /dev/sda
```

```bash
 n (new)
 p (primary)
 3 (partition number, since 1st and 2nd partition already exists)
 select default first available cylinder to the default last cylinder.
 t (type)
 3 (partition number)
 8e (set type to LVM)
 p (view the new partitions layout)
 w (write out the new partitions layout to disk)
```

Reboot the system so the new partition is recognized by the system; if you don't want to reboot, there is a command to update partition table like `source` that is `partprobe`

```bash
partprobe -s
```

![partprobecmd.png](/img/screenshots/partprobecmd.png)

We first check our disk ID to make sure "**8e**" is presented as Linux file system type ID and under **Linux LVM system**

![fdiskcmd.png](/img/screenshots/fdiskcmd.png)

### LVM operation to mount new partition to file system

The next step is to use LVM to take the newly formed partition and turn it into a new Physical Volume, add it to a Volume Group, and finally assimilate its free space into a Logical Volume.

Convert /dev/sda3 partition into a Physical Volume so LVM can make use of it:

```bash
pvcreate /dev/sda3
```

Add the new Physical Volume to the Volume Group as additional free space:

```bash
vgextend centos /dev/sda3
```

Note the free space now in the `Volume Group` which can now be assigned to a `Logical Volume`

```bash
vgdisplay
```

![vgdisplaycmd.png](/img/screenshots/vgdisplaycmd.png)

Have the Logical Volume (within the Volume Group) overtake the remaining free space of the Volume Group:

```bash
lvextend -l +100%FREE /dev/centos/root
```

```bash
vgdisplay
```

![vgdisplaycmd2.png](/img/screenshots/vgdisplaycmd2.png)

You can see all free space is now allocated.

The last step, we use `xfs_growfs` command to make logical volume live and mount filesystem so the new disk space can be used right away:

```bash
xfs_growfs /dev/centos/root
```

![df_h_cmd_aft.png](/img/screenshots/df_h_cmd_aft.png)

from above, you might feel the useful of LVM but extend is only one feature of it, we summarize all features of LVM by below table for reference:

| Task      | PV        | VG        | LV                  | Filesystem (XFS/EXT4) |
| --------- | --------- | --------- | ------------------- | --------------------- |
| Scan      | pvscan    | vgscan    | lvscan              | lsblk, blkid          |
| Create    | pvcreate  | vgcreate  | lvcreate            | mkfs.xfs/mkfs.ext4    |
| Display   | pvdisplay | vgdisplay | lvdisplay           | df, mount             |
| Extend    | N/A       | vgextend  | lvextend (lvresize) | xfs_growfs/resize2fs  |
| Reduce    | N/A       | vgreduce  | lvreduce (lvresize) | N/A/resize2fs         |
| Remove    | pvremove  | vgremove  | lvremove            | umount                |
| Resize    | N/A       | N/A       | lvresize            | xfs_growfs/resize2fs  |
| Attribute | pvchange  | vgchange  | lvchange            | /etc/fstab, remount   |











