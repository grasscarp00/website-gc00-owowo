---
title: "Linux+QEMU+Looking Glass实现200hz的虚拟机"
slug: 20250728-linux-qemu-looking-glass-200hz-vm
date: 2025-07-28T19:59:16+08:00
draft: false
comments: true
featured: false
categories: ["original"]
tags: ["linux", "vm"]
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

在尝试使用wine跑Chunithm的HDD失败了之后，我转向了另一条道路：双系统。

但是在分区的时候，看着电脑窘迫的储存空间，我在想能不能把Windows装在虚拟硬盘里。问了一下GPT，他说不行。然后我又问，又没有办法让虚拟机突破虚拟桌面60hz的限制，实现和屏幕一样的刷新率，他说可以：独显直通+[Looking Glass][1]。

Looking Glass使用Guest机和Host机的共享内存来传输画面，所以延迟能够做到1ms左右。至少他们是这么说的，我用的时候也没有感觉到有延迟的情况。

我的发行版是Arch，所以下面的步骤都是在Arch+AUR (paru) 的环境下配置的。其他发行版没有区别，就是包名可能不一样（可能还没有包）。

因为需要配置独显直通，所以我的解决方案是新建一个GRUB启动项，可以在开机时选择是否进入独显直通的系统。

## 1. 系统要求

[Looking Glass的官方文档][2]是这么说的：

> 使用Looking Glass的最基础要求是有两块GPU，下面的几种配置都可以：
> - 两块独立显卡（dGPU）
> - 一块独立显卡和一块集成显卡（iGPU），一般的笔记本都是这样
> - 一块独立显卡或集成显卡，和某些硬件支持的虚拟GPU（vGPU）。
>  
> 要注意的是，有些iGPU用户可能会由于内存带宽的限制导致只有有限的分辨率和刷新率可用，因为iGPU使用系统内存。
> PCIe的带宽也是限制的因素，因此两块GPU都至少应有PCIe3 x8或PCIe4 x4的速度。
> 
> ...
> 
> ~~Guest机使用的显卡必须连有一块物理显示器，或者一个便宜的诱骗器。如果显卡没有连接任何设备，那么Windows默认会关闭GPU的输出，这样的话Looking Glass就无法工作了。如果你用的是vGPU，那么它应该连接到一个虚拟显示器以符合要求。~~ **——这个可以通过装虚拟驱动的方式解决，接下来就是讲的这种方法。**
> 
> ...

他们还推荐使用有超线程功能的CPU。

本教程是为`一块独立显卡和一块集成显卡`的配置所做的。

以及基础的，你的电脑要支持VT-x/AMD-V和IOMMU（VT-d/AMD-Vi）技术。

你可以通过运行下面的命令来检查IOMMU：

```bash
sudo dmesg | grep -i iommu
```

没开的话先在BIOS中打开对应的开关，然后在`/etc/default/grub`中的`GRUB_CMDLINE_LINUX_DEFAULT`项添加`intel_iommu=on` `iommu=pt`或`amd_iommu=on` `iommu=pt`。最后执行：

```bash
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

重载配置，接着重启。重启后运行

```bash
sudo dmesg | grep -e DMAR -e IOMMU
```

若存在`enabled` `IOMMU`字样则说明当前IOMMU已经启用。接下来需要检查显卡是否处于独立的IOMMU Group中。如果不在（比如说声卡也在里面）理论上也能用，只要不是系统核心组件。本篇仅介绍独显在独立IOMMU Group下的情况。运行：

```bash
lspci -nn | grep -E 'VGA|3D'
```

查看独显那一行的第一块内容（总线地址），类似`01:00.0`。如果你还想直通别的PCIe设备也可以这么查看IOMMU Group。记住它，接下来运行：

```bash
find /sys/kernel/iommu_groups/ -type l
```

会出来不少东西。以我的举例：

```
  [...]
5 /sys/kernel/iommu_groups/15/devices/0000:01:00.0
6 /sys/kernel/iommu_groups/8/devices/0000:00:14.2
7 /sys/kernel/iommu_groups/8/devices/0000:00:14.0
  [...]
```

我的显卡的总线地址是`01:00.0`，那么就是上面的第5行。这一行的中间的数字`15`即为它所在的Group，接下来就要在输出的所有内容里查找有没有另一个也在组`15`的设备。如果没有，说明你的独显就在一个独立的IOMMU Group中了，可以继续。

假如说，我还希望直通总线地址`00:14.0`的设备，但是在第二个命令中发现它的组是`8`，并且还有另一个位于组`8`的设备，那么就要使用：

```bash
lspci -nn
```

查看这个设备的名称和功能，并判断它要不要、能不能跟随我想要直通的设备一起通过去。



## 2. 配置GRUB启动项

因为独显直通需要重启应用，所以放在GRUB里其实是一个很好的选择。当然也有不重启的办法，即实时挂载`vfio`内核模块，但还是不建议，因为有时候应用关不干净挺麻烦的。

- 注意，这里默认你的独显是在一个独立的IOMMU Group中的。

- 注意，下面的步骤将`vfio`内核模块的默认设置改为了直通独显，即一旦打开vfio模块就会自动把独显抢走，在正常启动的模式下也会如此。

 - 如果不希望默认是这个行为的话，可以将`/etc/modprobe.d/vfio.conf`换个位置（比如`/etc/modprobe.custom.d/vfio.conf`），并在对应的`/etc/mkinitcpio.conf.d/vfio.conf`文件中使用`HOOKS`项进行修改。具体操作可以搜索一下，问问AI也没问题。

### 1. 配置内核

首先执行：

```bash
lspci -nn | grep -E 'VGA|3D'
```

找到你的独立显卡的Vendor:Device ID，我的是`10de:1f97`，根据你的情况修改。然后运行下面的命令，其中`[VDID]`改成你自己的：

```bash
sudo tee /etc/modprobe.d/vfio.conf <<EOF
options vfio-pci ids=[VDID]
EOF
```

接着运行：

```bash
sudo cp /etc/mkinitcpio.conf /etc/mkinitcpio.conf.d/vfio.conf
```

然后编辑`/etc/mkinitcpio.conf.d/vfio.conf`中的`MODULES`项：

```
MODULES=(vfio_pci vfio vfio_iommu_type1)
```

编辑`/etc/mkinitcpio.d/linux.preset`。

- 注意，如果你使用的是linux-zen或者是别的内核，这里的名字可能也会跟随着变化。

在`PRESETS`一栏中添加`'vfio'`，然后再下面添加两项（或者多项，根据你的情况修改）：

```
vfio_config="/etc/mkinitcpio.conf.d/vfio.conf"
vfio_image="/boot/initramfs-linux-vfio.img"
```

我的文件最终长这样（使用`linux-zen`内核）：

```
# mkinitcpio preset file for the 'linux-zen' package

#ALL_config="/etc/mkinitcpio.conf"
ALL_kver="/boot/vmlinuz-linux-zen"

PRESETS=('default' 'vfio' 'fallback')

default_config="/etc/mkinitcpio.conf"
default_image="/boot/initramfs-linux-zen.img"
#default_uki="/efi/EFI/Linux/arch-linux-zen.efi"
#default_options="--splash /usr/share/systemd/bootctl/splash-arch.bmp"

vfio_config="/etc/mkinitcpio.conf.d/vfio.conf"
vfio_image="/boot/initramfs-linux-zen-vfio.img"

fallback_config="/etc/mkinitcpio.conf"
fallback_image="/boot/initramfs-linux-zen-fallback.img"
#fallback_uki="/efi/EFI/Linux/arch-linux-zen-fallback.efi"
fallback_options="-S autodetect"
```

然后运行：

```bash
sudo mkinitcpio -P
```

来更新配置。

### 2. 配置GRUB启动项

现在默认你已经修改了`/etc/default/grub`中的`GRUB_CMDLINE_LINUX_DEFAULT`项。没有的话就去上面的步骤中修改并更新。

然后打开`/boot/grub/grub.cfg`文件，寻找`### BEGIN /etc/grub.d/10_linux ###`这一行。第一个menuentry应该是你的主引导项，把它全部复制，就像这样：

```
menuentry 'Arch Linux' --class arch --class gnu-linux --class gnu --class os $menuentry_id_option 'gnulinux-simple-[...]' {
	load_video
	set gfxpayload=keep
	insmod gzio
	insmod part_gpt
	insmod fat
	search --no-floppy --fs-uuid --set=root [...]
	echo	'Loading Linux linux-zen ...'
	linux	/vmlinuz-linux-zen root=UUID=[...] rw rootflags=subvol=@  loglevel=5 intel_iommu=on iommu=pt
	echo	'Loading initial ramdisk ...'
	initrd	/intel-ucode.img /initramfs-linux-zen.img
}
```

其中的几个项被我换成了`[...]`，那里实际上是有内容的，直接复制就可以。然后打开`/etc/grub.d/40_custom`，拉到最下面，把它粘贴进去。

接下来

- 修改一下entry的名字

- 在`linux`一行的最后添加`intel_iommu=on iommu=pt`（有了就不用管了）

- 将`initrd`一行的第二个`.img`的名字修改为`initramfs-linux-vfio.img`（上面在`/etc/mkinitcpio.d/linux.preset`中写的名字）。

我的配置文件最终长这样：

```
[...]
menuentry 'Arch Linux (VFIO Passthrough)' --class arch --class gnu-linux --class gnu --class os $menuentry_id_option 'gnulinux-simple-[...]' {
        load_video
        set gfxpayload=keep
        insmod gzio
        insmod part_gpt
        insmod fat
        search --no-floppy --fs-uuid --set=root [...]
        echo    'Loading Linux linux-zen ...'
        linux   /vmlinuz-linux-zen root=UUID=[...] rw rootflags=subvol=@  loglevel=5 intel_iommu=on iommu=pt
        echo    'Loading initial ramdisk ...'
        initrd  /intel-ucode.img /initramfs-linux-zen-vfio.img
}
```

别照抄。如果你希望修改GRUB界面的等待时间的话可以去改`/etc/default/grub`。然后运行：

```bash
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

重载配置，最后重启。重启时可以直接选择进入VFIO的启动项了，并且应该顺利进入桌面。此时运行：

```bash
lspci -nnk -d 10de: # 对于 Nvidia 显卡
lspci -nnk -d 1002: # 对于 AMD 显卡（未测试）
```

可以检查独显是否已经分离。若有`Kernel driver in use: vfio-pci`则成功。



## 3. 安装

###1 . 包

我这边就统一使用paru进行安装了：

```bash
paru -Syu qemu libvirt virt-manager ovmf dnsmasq looking-glass looking-glass-module-dkms
sudo systemctl enable --now libvirtd
```

接下来要计算所需的共享内存。这个内存和你想要的最大分辨率有关，和刷新率无关，公式可以在[文档][3]中找到。他们同时提供了一个[参照表][4]，直接套就可以了。

虽然这个表提供了SDR和HDR的对应值，但他们其实并不推荐开HDR，因为

- Xorg和Wayland都用不了这个HDR

- GPU的驱动或者是硬件会把他转成SDR

- 吃内存，吃带宽，吃CPU，还吃别的（大胃王）。

总之除非想找茬，不然没必要开。

### 2. 配置IVSHMEM

一般情况下用KVMFR内核模块就可以，另一种方法用不上。这个模块在上面已经装好了，还需要设置一下。

配置自动加载。运行以下，其中的`32`根据情况修改为你的共享内存：

```bash
sudo cat > /etc/modprobe.d/kvmfr.conf <<EOF
#KVMFR Looking Glass module
options kvmfr static_size_mb=32
EOF
```

然后运行：

```bash
sudo cat > /etc/modules-load.d/kvmfr.conf <<EOF
# 3. KVMFR Looking Glass module
kvmfr
EOF
```

这个模块会自动创建一个`/dev/kvmfr0`的文件，接下来修改这个文件的权限。编辑`/etc/udev/rules.d/99-kvmfr.rules`，其中的`user`修改成你的用户名：

```
SUBSYSTEM=="kvmfr", OWNER="user", GROUP="kvm", MODE="0660"
```

此时可以重启了。你也可以运行：

```bash
sudo modprobe kvmfr static_size_mb=32
```

来手动启动KVMFR模块，其中的`32`更换为你的共享内存。运行以下来检查是否启动：

```bash
sudo dmesg | grep kvmfr
```

若有`kvmfr: creating 1 static devices`则正常。

### 3. 配置libvirt-qemu

修改`/etc/libvirt/qemu.conf`，在其中寻找`cgroup_device_acl`，取消注释并在最后一行添加`"/dev/kvmfr0"`（别忘了加逗号）。修改完后应该像这样：

```
[...]
# This is the basic set of devices allowed / required by
# all virtual machines.
#
# As well as this, any configured block backed disks,
# all sound device, and all PTY devices are allowed.
#  
# This will only need setting if newer QEMU suddenly
# wants some device we don't already know about.
# 
cgroup_device_acl = [
    "/dev/null", "/dev/full", "/dev/zero",
    "/dev/random", "/dev/urandom",
    "/dev/ptmx", "/dev/kvm",
    "/dev/userfaultfd", "/dev/kvmfr0"
]
[...]
```

接着运行：

```bash
sudo virsh net-autostart default
sudo virsh net-start default
```

以启动内建的NAT。

- 注意，如果你启动了代理的DNS覆写功能，应该是无法启动的。此时请修改代理设置中的DNS监听端口为非53端口，然后重新运行命令即可。

然后重启服务：

```bash
sudo systemctl restart libvirtd
```

### 4. 配置虚拟机

现在可以打开virt-manager了。在创建虚拟机之前，先在主页点击`Edit -> Preference`中勾选`Enable XML editing`。然后点击新建按钮创建一个虚拟机。

在创建结束的确认界面，勾选`Customize configuration before install`选项，然后点击`Finish`。接下来进行一些详细的配置：

- 在`Overview -> Firmware`中，选择带有`x64 OVMF secboot`字样的选项

- 如果你的CPU支持超线程，那么应当在`CPUs -> Topology`中手动配置

- 删除`Tablet`

- `Video QXL`中的`Model`项切换为`VGA`。

- Looking Glass推荐添加Vitrio Keyboard和Vitrio Mouse，不过我没找到Mouse在哪里，所以只加了个键盘。

接下来点击`Add Hardware -> PCI Host Device`，寻找你的显卡，然后点击`Finish`。其他保持默认即可。最后点击上方的`Begin Installation`开始安装。

然后就是安装。嗯。安装。嗯。好激动啊。你不激动吗。

### 5. 虚拟机中的设置

安装完毕之后在虚拟机里把驱动打了，然后去[IDDSampleDriver的Release界面][5]下载虚拟显示器驱动，并将里面的文件夹解压到C盘根目录下，就像这样：

![目录结构][6]

接下来装虚拟显示器驱动：
- 以管理员身份运行`installCert.bat`，安装驱动证书。
- 右键`iddsampledriver.inf`，点击`Install`。
- 打开Device Manager，随便点一个设备，再点击上方的：

```
Action 
-> Add legacy hardware 
-> Next 
-> Install the hardware that I manually select from a list 
-> Display adapters`
```

然后这样：

![Add Hardware][7]

然后一路Next到结束，此时你应该听见设备连接的声音。

你可以在`C:\IddSampleDriver/option.txt`中修改分辨率和帧率。建议修改为你要使用的屏幕的规格参数。

接下来关闭虚拟机，打开配置界面，在`Overview`界面点击右侧的`XML`标签页。

进行如下修改

- 将第一行`<domain type="kvm">`修改为`<domain type='kvm' xmlns:qemu='http://libvirt.org/schemas/domain/qemu/1.0'>`

- 将如下配置粘贴到第一行的下面。其中，`'size':33554432`的数字应当改为`你的共享内存(MB) * 1024 * 1024`：

    ```xml
    <qemu:commandline>
      <qemu:arg value="-device"/>
      <qemu:arg value="{'driver':'ivshmem-plain','id':'shmem0','memdev':'looking-glass'}"/>
      <qemu:arg value="-object"/>
      <qemu:arg value="{'qom-type':'memory-backend-file','id':'looking-glass','mem-path':'/dev/kvmfr0','size':33554432,'share':true}"/>
    </qemu:commandline>
    ```

- 滑动到最底部，找到`<memballoon model="virtio">`，将其修改为`<memballoon model="none">`。

 - 注意，这一项会导致虚拟机一口气占用全部内存，所以要好好分配它。

接下来启动虚拟机。在虚拟机内打开[Looking Glass **Host**的下载界面][8]（是Host），双击安装。

## 4. 使用

打开虚拟机内的系统设置，修改一下虚拟屏幕的分辨率。接着在Host机运行：

```bash
looking-glass-client -m KEY_RIGHTSHIFT
```

现在你应当已经连接到了虚拟机！

不过进去之后还有一些需要做的...

- 关闭虚拟机内的鼠标加速，否则两个鼠标加速会叠到一块去

- 不可以在虚拟机设置内删除`Display Spice`设备，因为Looking Glass使用`spice`做fallback和一些设备连接

- 如果你希望关闭`spice`的屏幕，因为Looking Glass会占用`spice`设备，导致无法从virt-manager访问桌面，所以可以先在虚拟机内把设置窗口向虚拟显示器的那个位置拖一下，然后再使用`looking-glass-client`连接虚拟机

还有几个tips

- 要使用剪切板同步功能，可以在虚拟机内安装`spice-guset-tools`。注意，建议都配置好之后再装，我在安装驱动的那一步曾试过，结果黑屏了...

- 可以在Looking Glass的文档查看[可用的options][9]。上面的`-m KEY_RIGHTSHIFT`是我自己设置的，因为他默认的快捷键是`ScrLk`，这个我没有。使用‘-F’自动进入全屏，使用‘-s’关闭Spice，使用‘-S’可以关掉虚拟机的屏保。

- 如果`looking-glass-client`窗口全屏之后有阴影边框出现，可以尝试先最大化再使用快捷键进入全屏。


----------


Looking Glass团队最近在把IDD也塞进去，以后就不用再手动配置虚拟显示器了，只需要安装Looking Glass Host即可一键完成，这下方便不少了。


----------


~~彩蛋（并非）~~

![SEGAY：孩子们这并不好笑][10]


----------


参考链接：

https://looking-glass.io/

https://gist.github.com/Ruakij/dd40b3d7cacf5d0f196d1116771b6e42

https://github.com/ge9/IddSampleDriver/releases/latest

https://www.reddit.com/r/VFIO/comments/wj6zhz/gpu_passthrough_looking_glass_no_external/


  [1]: https://looking-glass.io
  [2]: https://looking-glass.io/docs/B7/requirements/
  [3]: https://looking-glass.io/docs/B7/install_libvirt/#determining-memory
  [4]: https://looking-glass.io/docs/B7/install_libvirt/#id1
  [5]: https://github.com/ge9/IddSampleDriver/releases/latest
  [6]: 1.png
  [7]: 2.png
  [8]: https://looking-glass.io/artifact/stable/host
  [9]: https://looking-glass.io/docs/B7/usage/#all-command-line-options
  [10]: 3.png
