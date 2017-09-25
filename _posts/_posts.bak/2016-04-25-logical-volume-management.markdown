---
layout: post
title: Logical Volume Management
date: '2016-04-25 21:36:20'
---

Although <a href="https://btrfs.wiki.kernel.org/index.php/Main_Page">Btrf</a> is certainly going to be moving into the place that LVM management has traditionally had in Unix disk management, it certainly exists for legacy systems, especially systems that have kernels before v3.0. It should also be noted that as of Redhat 7 and Ubuntu 14.04, partprobe can be run on a mounted disk, so that a partition table can be reread and thus, be resized live. That said, this is how you would create a new physical disk, volume group, and logical volume old school!

Physical volumes are simply the hard drives that we will be using. This can be any number. In our example, I have an empty 1 TB disk that we'll be using.

<pre>[root@myserver ~]# fdisk -l

Disk /dev/sda: 150.3 GB, 150323855360 bytes
255 heads, 63 sectors/track, 18275 cylinders
Units = cylinders of 16065 * 512 = 8225280 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0x000a4239

Device Boot      Start         End      Blocks   Id  System
/dev/sda1   *           1       18276   146799616   83  Linux

<strong>Disk /dev/sdb: 1099.5 GB</strong>, 1099511627776 bytes
255 heads, 63 sectors/track, 133674 cylinders
Units = cylinders of 16065 * 512 = 8225280 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0x00000000</pre>

<br>Nowadays, it's rather common to get an HBA from the SAN. In this case, you're getting an entire disk that you will undoubtedly never need more than one partition. In this case, you can use the entire disk, ie /dev/sdb.</br>

<a href="http://www.tldp.org/HOWTO/LVM-HOWTO/initdisks.html">This</a> is NOT best practice.
<blockquote>Using the whole disk as a PV (as opposed to a partition spanning the whole disk) is not recommended because of the management issues it can create. Any other OS that looks at the disk will not recognize the LVM metadata and display the disk as being free, so it is likely it will be overwritten. LVM itself will work fine with whole disk PVs.</blockquote>
If you feel comfortable with your colleagues, and yourself, you can use the entire disk . Otherwise there is a risk that someone will run "fdisk" and deem the disk free and overwrite it. Or your SAN administrator will provide you with another HBA and it would appear in your OS that you have two free disks! Which one do you choose to format?

On the other hand, using the entire disk has the nice added benefit that you can simply expand the current disk without a need to rescan the partition table. There is no partition table! This allows you to grow your disk on-line in versions prior to Redhat 7 and Ubuntu 14.04. If you choose to have a partition on your disk, you cannot do this, as you can only remap a partition table in the kernel on an unmounted disk.

In our case, we'll use a partition table. We have a variety of different people working with our infrastructure. Moreover, we can just as easily add an additional disk to our configuration to grow our disk on-line.

Here's our disk after partitioning:
<pre># fdisk -l

Disk /dev/sda: 150.3 GB, 150323855360 bytes
255 heads, 63 sectors/track, 18275 cylinders
Units = cylinders of 16065 * 512 = 8225280 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0x000a4239

Device Boot      Start         End      Blocks   Id  System
/dev/sda1   *           1       18276   146799616   83  Linux

Disk /dev/sdb: 1649.3 GB, 1649267441664 bytes
4 heads, 3 sectors/track, 268435456 cylinders
Units = cylinders of 12 * 512 = 6144 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0x45c85def

Device Boot      Start         End      Blocks   Id  System
/dev/sdb1             171   268435456  1610611712   8e  Linux LVM

Disk /dev/mapper/mysql_vg-mysql: 1649.3 GB, 1649263247360 bytes
255 heads, 63 sectors/track, 200511 cylinders
Units = cylinders of 16065 * 512 = 8225280 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0x00000000</pre>
<br>Now let's create the physical volume:</br>
<pre>[root@myserver~]# pvcreate /dev/sdb1
Physical volume "/dev/sdb1" successfully created</pre>
<br>And now the volume group... The volume group is the collection of physical volumes (like the one we just created) that will be used for our group. I'm only using the one disk.</br>
<pre>[root@myserver ~]# vgcreate mysql_vg /dev/sdb1
Volume group "mysql_vg" successfully created

[root@myserver ~]# vgdisplay
--- Volume group ---
VG Name               mysql_vg
System ID
Format                lvm2
Metadata Areas        1
Metadata Sequence No  1
VG Access             read/write
VG Status             resizable
MAX LV                0
Cur LV                0
Open LV               0
Max PV                0
Cur PV                1
Act PV                1
VG Size               1024.00 GiB
PE Size               4.00 MiB
Total PE              262143
Alloc PE / Size       0 / 0
Free  PE / Size       262143 / 1024.00 GiB
VG UUID               jhPoOI-q1RP-XqGS-yQlF-aV3F-nQM1-XdxL8D</pre>
<br>And now you'll need to create the logical volume. In almost all cases, when working with LVM management, I'll receive an HBA from a SAN and I'll want to use the entire disk. In this case, look at the "Total PE" size from "vgdisplay" and use this number to create the logical volume. Notice the use of "-l" flag to specify the amount of physical extents you want to allocate.</br>
<pre>[root@myserver ~]# lvcreate -l 262143 -n mysql mysql_vg
Logical volume "mysql_1" created.

[root@vgms0477 ~]# lvdisplay
--- Logical volume ---
LV Path                /dev/mysql_vg/mysql
LV Name                mysql
VG Name                mysql_vg
LV UUID                AzmKum-8Nte-FN4i-JqBs-Hj9H-bOGc-dEtVfC
LV Write Access        read/write
LV Creation host, time vgms0477.vgregion.se, 2016-01-26 08:38:11 +0100
LV Status              available
# open                 0
LV Size                1024.00 GiB
Current LE             262143
Segments               1
Allocation             inherit
Read ahead sectors     auto
- currently set to     256
Block device           253:0</pre>
<br>Everything with the LVM is now finished. Unfortunately, a disk without a file system is pointless. To finish up everything, let's format an ext4 filesystem on top of our brand new logical volume:</br>
<pre>[root@myserver ~]# mkfs.ext4 /dev/mysql_vg/mysql
mke2fs 1.41.12 (17-May-2010)
Filesystem label=
OS type: Linux
Block size=4096 (log=2)
Fragment size=4096 (log=2)
Stride=0 blocks, Stripe width=0 blocks
67108864 inodes, 268434432 blocks
13421721 blocks (5.00%) reserved for the super user
First data block=0
Maximum filesystem blocks=4294967296
8192 block groups
32768 blocks per group, 32768 fragments per group
8192 inodes per group
Superblock backups stored on blocks:
32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208,
4096000, 7962624, 11239424, 20480000, 23887872, 71663616, 78675968,
102400000, 214990848

Writing inode tables: done
Creating journal (32768 blocks): done
Writing superblocks and filesystem accounting information: done

This filesystem will be automatically checked every 28 mounts or
180 days, whichever comes first.  Use tune2fs -c or -i to override.</pre>
<br>And the mount our disk. You'll need to update fstab to make the disk mount persistent across reboots. Check the mount point with "lvdisplay" and then add the entry in fstab.</br>
<pre>[root@vgms0477 ~]# lvdisplay
--- Logical volume ---
<strong>  LV Path                /dev/mysql_vg/mysql</strong>
LV Name                mysql
VG Name                mysql_vg
LV UUID                AzmKum-8Nte-FN4i-JqBs-Hj9H-bOGc-dEtVfC
LV Write Access        read/write
LV Creation host, time vgms0477.vgregion.se, 2016-01-26 08:38:11 +0100
LV Status              available
# open                 0
LV Size                1024.00 GiB
Current LE             262143
Segments               1
Allocation             inherit
Read ahead sectors     auto
- currently set to     256
Block device           253:0

[root@myserver ~]# cat /etc/fstab

#
# /etc/fstab
# Created by anaconda on Fri Oct 30 13:05:37 2015
#
# Accessible filesystems, by reference, are maintained under '/dev/disk'
# See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info
#
UUID=723d2582-4c93-4a2a-b28e-75363945f6fa /                       ext4    defaults        1 1
tmpfs                   /dev/shm                tmpfs   defaults        0 0
devpts                  /dev/pts                devpts  gid=5,mode=620  0 0
sysfs                   /sys                    sysfs   defaults        0 0
proc                    /proc                   proc    defaults        0 0
<strong>/dev/mysql_vg/mysql     /var/lib/mysql          ext4    defaults        1 1</strong></pre>
<br>Use the command "mount -a" to mount all entries in fstab and voilá.</br>