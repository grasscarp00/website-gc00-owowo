---
title: "PVE内给zvol btrfs的DSM扩容"
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

看了一下别人的教程，基本都是需要在DSM内操作。我来个反其道而行之，在PVE里操作。

先关闭虚拟机。

## 1. 包

```
apt update
apt install mdadm parted
```

## 2. 硬盘扩容

先在GUI内扩容，然后打开Shell。因为我是用的zvol，并且zpool名称叫`rp_01`，所以块文件就在`/dev/zvol/rp_01/vm-200-disk-0`，对应的数据分区，也就是第三个分区，就是`vm-200-disk-0-part3`。接下来使用`parted`工具扩容：

```
parted /dev/zvol/rp_01/vm-200-disk-0 resizepart 3 100%
```

## 3. 分区扩容

先扩容raid：

```
mdadm --assemble --update=devicesize /dev/md2 /dev/zvol/rp_01/vm-200-disk-0-part3
mdadm --grow /dev/md2 --size=max
```

接下来需要挂载lvm卷。先扫描：

```
vgscan
```

后挂载。我的卷组是`vg1`，存储卷是`volume_1`，根据你的情况修改：

```
vgchange -ay vg1
mkdir -p /mnt/tmp_mount
mount -t btrfs /dev/vg1/volume_1 /mnt/tmp_mount
```

扩容：

```
btrfs filesystem resize max /mnt/tmp_mount
```

## 4. 从PVE中卸载分区

运行：

```
umount /mnt/tmp_mount
vgchange -an vg1
mdadm --stop /dev/md2
```

现在可以打开DSM了，不过还没有结束。DSM启动后，进入`Storage Manager`应用，他会提示你需要expand，同意即可。

最后还有很重要的一步！把`mdadm`包删掉。否则下次PVE启动，它会自动把检测到的raid挂载上，然后kernel就要炸了，DSM也会跟着炸：

```
apt autoremove mdadm
dpkg -P mdadm
```

到这里就完成了。


----------


参考：

https://xpenology.com/forum/topic/66292-virtual-disk-info/

https://xpenology.com/forum/topic/17105-expandresize-btrfs-syno-volume-after-disk-space-increased-esxi-hyper-v/
