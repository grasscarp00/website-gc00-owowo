---
title: "Expanding DSM with zvol btrfs in PVE"
slug: 20250802-expand-zvol-btrfs-dsm-in-pve-host
date: 2025-08-02T15:07:45+08:00
draft: false
comments: true
featured: false
categories: ["original"]
tags: ["linux", "pve", "vm"]
#readingTime: false
#license: false
#series: [""]
#author: ""
#aliases: ""
#cover: 
#expiryDate: 
#links:
#    - title: ""
#      description: ""
#      website: ""
#      image: 
---

**WARNING: This English version is translated by AI from the Chinese version.**

After reviewing other tutorials, I found that most require operations within DSM. I'll take a different approach and perform the operations directly in PVE.

First, power down the virtual machine.

## 1. Required Packages

```
apt update
apt install mdadm parted
```

## 2. Disk Expansion

Start by expanding the disk through the GUI interface, then open the Shell. Since I'm using zvol with a zpool named `rp_01`, the block device is located at `/dev/zvol/rp_01/vm-200-disk-0`, and the corresponding data partition (the third partition) is `vm-200-disk-0-part3`. Use the `parted` tool to expand the partition:

```
parted /dev/zvol/rp_01/vm-200-disk-0 resizepart 3 100%
```

## 3. Partition Expansion

Begin by expanding the RAID array:

```
mdadm --assemble --update=devicesize /dev/md2 /dev/zvol/rp_01/vm-200-disk-0-part3
mdadm --grow /dev/md2 --size=max
```

Next, mount the LVM volume. Start with a scan:

```
vgscan
```

Then proceed with mounting. My volume group is named `vg1` and the storage volume is `volume_1` - adjust these names according to your setup:

```
vgchange -ay vg1
mkdir -p /mnt/tmp_mount
mount -t btrfs /dev/vg1/volume_1 /mnt/tmp_mount
```

Perform the expansion:

```
btrfs filesystem resize max /mnt/tmp_mount
```

## 4. Unmounting from PVE

Execute the following commands:

```
umount /mnt/tmp_mount
vgchange -an vg1
mdadm --stop /dev/md2
```

You can now start DSM, but the process isn't complete yet. Once DSM boots up, open the `Storage Manager` application. It will prompt you to expand the storage - simply accept the prompt.

Finally, here's a crucial step! Remove the `mdadm` package. If you don't, PVE will automatically mount any detected RAID arrays on the next boot, which will cause kernel crashes and DSM failures:

```
apt autoremove mdadm
dpkg -P mdadm
```

The expansion process is now complete.

----------

References:

https://xpenology.com/forum/topic/66292-virtual-disk-info/

https://xpenology.com/forum/topic/17105-expandresize-btrfs-syno-volume-after-disk-space-increased-esxi-hyper-v/
