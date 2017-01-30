---
layout: post
title: "how to increase disk size on Ubuntu KVM guest"
date: 2017-01-28 11:28:17 +0000
comments: true
categories: [kvm,ubuntu,libvirt,lvm]
---
# Basic process

<!-- MarkdownTOC -->

- [Stop the VM](#stop-the-vm)
- [Find disk image](#find-disk-image)
- [Expand disk image](#expand-disk-image)
  - [List Physical Volumes \(PV\)](#list-physical-volumes-pv)
  - [Increase the PV size](#increase-the-pv-size)
  - [List Volume Groups \(VG\)](#list-volume-groups-vg)
  - [Find available extents](#find-available-extents)
  - [Extend Logical Volume \(LV\)](#extend-logical-volume-lv)
- [Expand filesystem](#expand-filesystem)

<!-- /MarkdownTOC -->


  1. Stop the virtual machine
  2. Find disk image
  3. Expand disk image
  4. Start the virtual machine
  5. Expand the partition
  6. Expand the logical volume
    a.
  7. Expand the filesystem

## Stop the VM

```
kvm01 ~ # virsh shutdown pe.shadowsun.com.ar
Domain pe.shadowsun.com.ar is being shutdown
kvm01 ~ #
```

## Find disk image

While we wait for the VM to shutdown, we go and check the VM configuration to find the location of the disk image.

```
kvm01 ~ # virsh dumpxml pe.shadowsun.com.ar|grep -C5 disk
  <on_poweroff>destroy</on_poweroff>
  <on_reboot>restart</on_reboot>
  <on_crash>destroy</on_crash>
  <devices>
    <emulator>/usr/bin/kvm-spice</emulator>
    <disk type='file' device='disk'>
      <driver name='qemu' type='raw'/>
      <source file='/var/lib/virt/vmimages/pe.shadowsun.com.ar-disk1'/>
      <target dev='vda' bus='virtio'/>
      <alias name='virtio-disk0'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x04' function='0x0'/>
    </disk>
    <controller type='usb' index='0'>
      <alias name='usb0'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x01' function='0x2'/>
    </controller>
    <controller type='pci' index='0' model='pci-root'>
kvm01 ~ #
```

## Expand disk image

Before the next step, make sure that the VM is not running

Now that we have the disk image, the next step is

```
# qemu-img resize /var/lib/virt/vmimages/pe.shadowsun.com.ar-disk1 +30G
Image resized.
#
```

```
# parted
GNU Parted 3.2
Using /dev/mapper/vg00-swap
Welcome to GNU Parted! Type 'help' to view a list of commands.
(parted) help
  align-check TYPE N                        check partition N for TYPE(min|opt) alignment
  help [COMMAND]                           print general help, or help on COMMAND
  mklabel,mktable LABEL-TYPE               create a new disklabel (partition table)
  mkpart PART-TYPE [FS-TYPE] START END     make a partition
  name NUMBER NAME                         name partition NUMBER as NAME
  print [devices|free|list,all|NUMBER]     display the partition table, available devices, free space, all found partitions, or a particular partition
  quit                                     exit program
  rescue START END                         rescue a lost partition near START and END
  resizepart NUMBER END                    resize partition NUMBER
  rm NUMBER                                delete partition NUMBER
  select DEVICE                            choose the device to edit
  disk_set FLAG STATE                      change the FLAG on selected device
  disk_toggle [FLAG]                       toggle the state of FLAG on selected device
  set NUMBER FLAG STATE                    change the FLAG on partition NUMBER
  toggle [NUMBER [FLAG]]                   toggle the state of FLAG on partition NUMBER
  unit UNIT                                set the default unit to UNIT
  version                                  display the version number and copyright information of GNU Parted
(parted) select /dev/vda
Using /dev/vda
(parted) print
Model: Virtio Block Device (virtblk)
Disk /dev/vda: 42.9GB
Sector size (logical/physical): 512B/512B
Partition Table: msdos
Disk Flags:

Number  Start   End     Size    Type      File system  Flags
 1      1049kB  512MB   511MB   primary   ext4         boot
 2      513MB   10.7GB  10.2GB  extended
 5      513MB   10.7GB  10.2GB  logical                lvm

(parted) print free
Model: Virtio Block Device (virtblk)
Disk /dev/vda: 42.9GB
Sector size (logical/physical): 512B/512B
Partition Table: msdos
Disk Flags:

Number  Start   End     Size    Type      File system  Flags
        32.3kB  1049kB  1016kB            Free Space
 1      1049kB  512MB   511MB   primary   ext4         boot
        512MB   513MB   1048kB            Free Space
 2      513MB   10.7GB  10.2GB  extended
 5      513MB   10.7GB  10.2GB  logical                lvm
        10.7GB  42.9GB  32.2GB            Free Space

(parted) resizepart 2 42.9GB
(parted) resizepart 5 42.9GB
(parted) print
Model: Virtio Block Device (virtblk)
Disk /dev/vda: 42.9GB
Sector size (logical/physical): 512B/512B
Partition Table: msdos
Disk Flags:

Number  Start   End     Size    Type      File system  Flags
 1      1049kB  512MB   511MB   primary   ext4         boot
 2      513MB   42.9GB  42.4GB  extended
 5      513MB   42.9GB  42.4GB  logical                lvm
(parted) quit
Information: You may need to update /etc/fstab.

#
```

### List Physical Volumes (PV)

```
pe ~ # pvs
  PV         VG   Fmt  Attr PSize PFree
  /dev/vda5  vg00 lvm2 a--  9.52g    0
```

### Increase the PV size

```
pe ~ # pvresize /dev/vda5
  Physical volume "/dev/vda5" changed
  1 physical volume(s) resized / 0 physical volume(s) not resized
pe ~ #
```

### List Volume Groups (VG)

```
pe ~ # vgs
  VG   #PV #LV #SN Attr   VSize  VFree
  vg00   1   2   0 wz--n- 39.47g 29.95g
pe ~ #
```

### Find available extents

```
pe ~ # vgs -o vg_name,vg_free_count
  VG   Free
  vg00 7668
pe ~ #
```

```
pe ~ # lvs
  LV   VG   Attr       LSize Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  root vg00 -wi-ao---- 8.02g
  swap vg00 -wi-ao---- 1.50g
pe ~ #
```


### Extend Logical Volume (LV)

```
pe ~ # lvextend -l +7668 /dev/vg00/root
  Size of logical volume vg00/root changed from 8.02 GiB (2054 extents) to 37.98 GiB (9722 extents).
  Logical volume root successfully resized.
pe ~ #
```

## Expand filesystem

```
pe ~ # resize2fs /dev/vg00/root
resize2fs 1.42.13 (17-May-2015)
Filesystem at /dev/vg00/root is mounted on /; on-line resizing required
old_desc_blocks = 1, new_desc_blocks = 3
The filesystem on /dev/vg00/root is now 9955328 (4k) blocks long.

pe ~ #
```

