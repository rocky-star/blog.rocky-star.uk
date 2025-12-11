+++
date = "2025-11-26T05:25:34+08:00"
draft = false
title = "在Proxmox VE上安装群晖 Virtual DSM"
description = "更可靠地安装黑群晖虚拟机，免去对各类第三方引导程序的依赖。"
+++

要在非群晖官方硬件上运行群晖DiskStation Manager（DSM）系统（即黑群晖），一般有两种方式：直接在实体机上安装DSM，或是先在实体机上安装虚拟化平台，再在虚拟机中安装DSM。这两种方式都依赖于第三方引导程序（如[ARPL](https://github.com/fbelavenuto/arpl)和[RR](https://github.com/RROrg/rr)）。本文将指引您在Proxmox VE上安装群晖Virtual DSM，即群晖的官方虚拟化系统。相比使用第三方引导程序，选用Virtual DSM有一些优势，包括：

* 使用VirGL半虚拟化显卡以轻松启用Synology Photos人脸识别等功能，而无需独占透传硬件显卡；
* 无需使用第三方引导程序，只使用群晖的官方代码；
* 系统自带virtio半虚拟化设备驱动程序，无需注入驱动程序；

但Virtual DSM也有一些缺陷：

* 不支持NVMe控制器，因此您的NVMe固态硬盘需要以虚拟磁盘的形式（而非透传）连接进虚拟机；
* 不支持查看SMART数据，因此您需要在Proxmox VE上配置SMART监测；
* 磁盘分区结构与官方群晖硬件和黑群晖不同，Virtual DSM的每块磁盘都只包含一个Btrfs分区；
* 不支持Virtual Machine Manager套件；
* Surveillance Station套件不包含任何免费许可证。

本文给出的步骤来自于[vdsm/virtual-dsm](https://github.com/vdsm/virtual-dsm)项目，该项目让您能在Docker中以容器形式安装Virtual DSM，其实现方式是将宿主机的/dev/kvm设备挂载到容器内，再在容器内运行QEMU/KVM。由于Proxmox VE不支持Docker，因此要使用此项目，必须先创建一个LXC容器，再在容器中安装Docker，最后在容器内的Docker中使用此项目运行QEMU/KVM。考虑到Proxmox VE为QEMU/KVM虚拟机提供了原生支持，这样层层嵌套似乎显得有些多余。本文的目的即为指引您直接在Proxmox VE中安装Virtual DSM，而无需利用此项目。

## 先决条件

您的宿主机上应安装有Proxmox VE 8或更高版本。如没有，您可从此[下载安装映像文件](https://mirrors.tuna.tsinghua.edu.cn/proxmox/iso/proxmox-ve_9.1-1.iso)。

## 创建虚拟机

您需要在Proxmox VE中为Virtual DSM创建虚拟机。创建时，请注意：

* 选用VirtIO SCSI single作SCSI控制器；
* 不启用QEMU客户机代理；
* 在scsi9创建一个大小为128 MiB（0.125 GiB）的引导磁盘；
* 为网络设备选择到局域网的网桥，并禁用防火墙。

创建完成后，请在scsi10创建一个大小为10 GiB的系统磁盘。用于数据存储的磁盘应于scsi11、scsi12等位置添加，以此类推。

![配置SCSI控制器并禁用QEMU客户机代理](no-qemu-agent.png)

![禁用网络设备的防火墙](disable-firewall.png)

![配置scsi9磁盘](scsi9.png)

![配置scsi10磁盘](scsi10.png)

## 获取补丁文件提取工具

较新的DSM补丁文件无法用tar提取，因此您需要从DSM 7.0.1-42218的补丁文件中获取用以提取文件的工具syno_<wbr>extract_<wbr>system_<wbr>patch。连接到Proxmox VE服务器并执行下列命令，以下载DSM 7.0.1-42218的补丁文件：

```console
# wget https://cndl.synology.cn/download/DSM/release/7.0.1/42218/DSM_VirtualDSM_42218.pat
--2024-11-29 22:29:07--  https://cndl.synology.cn/download/DSM/release/7.0.1/42218/DSM_VirtualDSM_42218.pat
Resolving cndl.synology.cn (cndl.synology.cn)... 180.163.57.90, 180.163.57.82, 180.163.57.70, ...
Connecting to cndl.synology.cn (cndl.synology.cn)|180.163.57.90|:443... connected.
HTTP request sent, awaiting response... 301 Moved Permanently
Location: https://global.synologydownload.com/download/DSM/release/7.0.1/42218/DSM_VirtualDSM_42218.pat [following]
--2024-11-29 22:29:08--  https://global.synologydownload.com/download/DSM/release/7.0.1/42218/DSM_VirtualDSM_42218.pat
Resolving global.synologydownload.com (global.synologydownload.com)... 104.22.1.171, 104.22.0.171, 172.67.28.107, ...
Connecting to global.synologydownload.com (global.synologydownload.com)|104.22.1.171|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 379637760 (362M) [application/octet-stream]
Saving to: ‘DSM_VirtualDSM_42218.pat’

DSM_VirtualDSM_42218.pat     100%[===========================================>] 362.05M  4.51MB/s    in 79s

2024-11-29 22:30:30 (4.56 MB/s) - ‘DSM_VirtualDSM_42218.pat’ saved [379637760/379637760]

```

执行下列命令以从补丁文件中提取rd.gz文件：

```console
# tar -xvf DSM_VirtualDSM_42218.pat rd.gz
rd.gz
```

执行下列命令以提取rd.gz文件的内容：
```console
# xz -dc < rd.gz > rd.cpio
xz: (stdin): Compressed data is corrupt
# mkdir rd
# cd rd
# cpio -idm < ../rd.cpio
36797 blocks
# cd ..
```

执行下列命令以准备syno_<wbr>extract_<wbr>system_<wbr>patch的文件：

```console
# mkdir extract
# cp rd/usr/lib/libcurl.so.4 \
> rd/usr/lib/libmbedcrypto.so.5 \
> rd/usr/lib/libmbedtls.so.13 \
> rd/usr/lib/libmbedx509.so.1 \
> rd/usr/lib/libmsgpackc.so.2 \
> rd/usr/lib/libsodium.so \
> rd/usr/lib/libsynocodesign-ng-virtual-junior-wins.so.7 \
> extract
# cp rd/usr/syno/bin/scemd extract/syno_extract_system_patch
# chmod +x extract/syno_extract_system_patch
```

## 下载DSM并准备根文件系统

执行下列命令，以下载DSM 7.2.2-72806的补丁文件：

```console
# wget https://cndl.synology.cn/download/DSM/release/7.2.2/72806/DSM_VirtualDSM_72806.pat
--2024-11-29 22:46:48--  https://cndl.synology.cn/download/DSM/release/7.2.2/72806/DSM_VirtualDSM_72806.pat
Resolving cndl.synology.cn (cndl.synology.cn)... 180.163.57.90, 180.163.57.82, 180.163.57.70, ...
Connecting to cndl.synology.cn (cndl.synology.cn)|180.163.57.90|:443... connected.
HTTP request sent, awaiting response... 301 Moved Permanently
Location: https://global.synologydownload.com/download/DSM/release/7.2.2/72806/DSM_VirtualDSM_72806.pat [following]
--2024-11-29 22:46:49--  https://global.synologydownload.com/download/DSM/release/7.2.2/72806/DSM_VirtualDSM_72806.pat
Resolving global.synologydownload.com (global.synologydownload.com)... 104.22.1.171, 104.22.0.171, 172.67.28.107, ...
Connecting to global.synologydownload.com (global.synologydownload.com)|104.22.1.171|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 361010261 (344M) [application/octet-stream]
Saving to: ‘DSM_VirtualDSM_72806.pat’

DSM_VirtualDSM_72806.pat     100%[===========================================>] 344.29M   650KB/s    in 5m 30s

2024-11-29 22:52:21 (1.04 MB/s) - ‘DSM_VirtualDSM_72806.pat’ saved [361010261/361010261]

```

执行下列命令，以提取补丁文件的内容到patch目录：

```console
# mkdir patch
# LD_LIBRARY_PATH=extract extract/syno_extract_system_patch DSM_VirtualDSM_72806.pat patch
```

执行下列命令，以提取引导程序：

```console
# unzip patch/synology_kvmx64_virtualdsm_72806_8A784779__.bin.zip
Archive:  patch/synology_kvmx64_virtualdsm_72806_8A784779__.bin.zip
  inflating: synology_kvmx64_virtualdsm_72806_8A784779__.bin
```

**注意!** 若找不到unzip命令，执行`apt install unzip`以安装，然后再试一次。

执行下列命令，以准备根文件系统：

```console
# mkdir rootfs
# mv patch/hda1.tgz patch/hda1.txz
# mv patch/packages rootfs/.SynoUpgradePackages
# rm rootfs/.SynoUpgradePackages/ActiveInsight-x86_64-2.2.3-23123.spk
# tar -xJpf patch/synohdpack_img.txz -C rootfs
# mkdir -p rootfs/usr/syno/synoman/indexdb
# tar -xJpf patch/indexdb.txz -C rootfs/usr/syno/synoman/indexdb
# tar -xJpf patch/hda1.txz --skip-old-files -C rootfs
```

## 建立硬盘分区并安装DSM

请记住您的虚拟机ID，其一般为三位数字（如100）。在之后的命令中，我将以&lt;VMID&gt;来标注它。遇到此标记时，您应将您的虚拟机ID用以替换。此外，↵则代表您应按回车键，以同意默认选项或默认值。

执行下列命令，以使用fdisk建立磁盘分区：

```console
# fdisk /dev/pve/vm-<VMID>-disk-1

Welcome to fdisk (util-linux 2.38.1).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Device does not contain a recognized partition table.
Created a new DOS (MBR) disklabel with disk identifier 0x3ab8643b.

Command (m for help): x

Expert command (m for help): i

Enter the new disk identifier: 0x6f9ee2e9

Disk identifier changed from 0x3ab8643b to 0x6f9ee2e9.

Expert command (m for help): r

Command (m for help): n
Partition type
   p   primary (0 primary, 0 extended, 4 free)
   e   extended (container for logical partitions)
Select (default p): ↵

Using default response p.
Partition number (1-4, default 1): ↵
First sector (2048-20971519, default 2048): ↵
Last sector, +/-sectors or +/-size{K,M,G,T,P} (2048-20971519, default 20971519): -2G

Created a new partition 1 of type 'Linux' and of size 8 GiB.

Command (m for help): n
Partition type
   p   primary (1 primary, 0 extended, 3 free)
   e   extended (container for logical partitions)
Select (default p): ↵

Using default response p.
Partition number (2-4, default 2): ↵
First sector (16777216-20971519, default 16777216): ↵
Last sector, +/-sectors or +/-size{K,M,G,T,P} (16777216-20971519, default 20971519): ↵

Created a new partition 2 of type 'Linux' and of size 2 GiB.

Command (m for help): t
Partition number (1,2, default 2): ↵
Hex code or alias (type L to list all): 82

Changed type of partition 'Linux' to 'Linux swap / Solaris'.

Command (m for help): w
The partition table has been altered.
Calling ioctl() to re-read partition table.
Re-reading the partition table failed.: Invalid argument

The kernel still uses the old table. The new table will be used at the next reboot or after you run partprobe(8) or partx(8).
```

执行下列命令，以建立文件系统：

```console
# losetup -P /dev/loop0 /dev/pve/vm-<VMID>-disk-1
# mke2fs -t ext4 -b 4096 -d rootfs -L 1.44.1-42218 /dev/loop0p1
mke2fs 1.47.0 (5-Feb-2023)
Creating filesystem with 2096896 4k blocks and 524288 inodes
Filesystem UUID: 48a083ee-8164-4138-8b3c-a6281077d7a0
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632

Allocating group tables: done
Writing inode tables: done
Creating journal (16384 blocks): done
Copying files into the device: done
Writing superblocks and filesystem accounting information: done

# losetup -d /dev/loop0
```

执行下列命令，以写入引导程序：

```console
# dd if=synology_kvmx64_virtualdsm_72806_8A784779__.bin of=/dev/pve/vm-<VMID>-disk-0 bs=1M
```

## 配置qemu-host并添加graceful shutdown支持

Virtual DSM在标准的QEMU客户机代理（QEMU guest agent）协议上以专有协议与宿主机通信，并获取序列号等硬件信息。但Proxmox VE所附带的QEMU客户机代理支持无法支持这一协议。因此，您需要安装第三方工具[qemu-host](https://github.com/qemus/qemu-host)，以手动指定序列号等参数。

有多种方式可用以运行qemu-host工具，本文则使用systemd管理它的启停。此外，由于让Virtual DSM正确关机的方法是给它的QEMU客户机代理发送QMP指令，而要发送此种指令，必须使用qemu-host的HTTP API。因此，此graceful shutdown功能亦将用hookscript提供支持。

### 下载qemu-host

执行下列命令，以下载qemu-host工具：

```console
# wget -O /usr/local/bin/qemu-host https://github.com/qemus/qemu-host/releases/download/v2.05/qemu-host.bin
--2024-11-29 23:45:58--  https://github.com/qemus/qemu-host/releases/download/v2.05/qemu-host.bin
Resolving github.com (github.com)... 20.27.177.113
Connecting to github.com (github.com)|20.27.177.113|:443... connected.
HTTP request sent, awaiting response... 302 Found
Location: https://objects.githubusercontent.com/github-production-release-asset-2e65be/632726151/48da73b8-9845-4d85-9301-3d71346f3ff2?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=releaseassetproduction%2F20241129%2Fus-east-1%2Fs3%2Faws4_request&X-Amz-Date=20241129T154600Z&X-Amz-Expires=300&X-Amz-Signature=a6153112e3262b37797983b29eaf6cd0525b4eb027da83db29cca4522f6f9f91&X-Amz-SignedHeaders=host&response-content-disposition=attachment%3B%20filename%3Dqemu-host.bin&response-content-type=application%2Foctet-stream [following]
--2024-11-29 23:46:00--  https://objects.githubusercontent.com/github-production-release-asset-2e65be/632726151/48da73b8-9845-4d85-9301-3d71346f3ff2?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=releaseassetproduction%2F20241129%2Fus-east-1%2Fs3%2Faws4_request&X-Amz-Date=20241129T154600Z&X-Amz-Expires=300&X-Amz-Signature=a6153112e3262b37797983b29eaf6cd0525b4eb027da83db29cca4522f6f9f91&X-Amz-SignedHeaders=host&response-content-disposition=attachment%3B%20filename%3Dqemu-host.bin&response-content-type=application%2Foctet-stream
Resolving objects.githubusercontent.com (objects.githubusercontent.com)... 185.199.111.133, 185.199.108.133, 185.199.109.133, ...
Connecting to objects.githubusercontent.com (objects.githubusercontent.com)|185.199.111.133|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 7247431 (6.9M) [application/octet-stream]
Saving to: ‘/usr/local/bin/qemu-host’

/usr/local/bin/qemu-host     100%[===========================================>]   6.91M  1.62MB/s    in 4.7s

2024-11-29 23:46:08 (1.47 MB/s) - ‘/usr/local/bin/qemu-host’ saved [7247431/7247431]

```

执行下列命令，以获取您的CPU型号：

```console
# lscpu | grep -m 1 'Model name' | cut -f 2 -d ":" | awk '{$1=$1}1' | sed 's# @.*##g' | sed s/"(R)"//g | sed 's/[^[:alnum:] ]\+/ /g' | sed 's/  */ /g'
<CPU>
```

### 创建并启用qemu-host systemd服务

新建文件/etc/systemd/system/qemu-host.service，并填入以下内容：

```ini
[Unit]
Description=QEMU Host for Virtual DSM
Before=pve-guests.service

[Service]
ExecStart=/usr/local/bin/qemu-host -cpu <N-CORES> -cpu_arch '<CPU>,,'

[Install]
WantedBy=multi-user.target
```

将&lt;N-CORES&gt;替换为您为虚拟机分配的核心数量，并将&lt;CPU&gt;替换为您在上一步获得的CPU型号。

若要在Virtual DSM中登录群晖账户，或激活Surveillance Station的许可证，您需要将ExecStart行替换为下列内容：

```ini
ExecStart=/usr/local/bin/qemu-host -cpu <N-CORES> -cpu_arch '<CPU>,,' -hostsn <HOST-SN> -guestsn <GUEST-SN>
```

将&lt;HOST-SN&gt;替换为某台官方群晖设备的序列号，并将&lt;GUEST-SN&gt;替换为有效的Virtual DSM序列号。要获得有效的Virtual DSM序列号，您应在某台官方群晖设备上创建Virtual DSM，并获取它的序列号。

若要在安装有DSM 7.2或更高版本的Virtual DSM中启用Advanced Media Extensions，您需要将ExecStart行替换为下列内容：

```ini
ExecStart=/usr/local/bin/qemu-host -cpu <N-CORES> -cpu_arch '<CPU>,,' -mac <MAC> -model <MODEL> -hostsn <HOST-SN> -guestsn <GUEST-SN>
```

将&lt;MAC&gt;替换为某台官方群晖设备的MAC地址（形如aa:bb:cc:dd:ee:ff），并将&lt;MODEL&gt;替换为这台官方群晖设备的型号。

重新加载systemd单元并启用该服务：

```console
# systemctl daemon-reload
# systemctl enable --now qemu-host
```

### 设置hookscript

新建文件/var/lib/vz/snippets/hookscript-vdsm.sh，并填入以下内容：

```bash
#!/bin/sh
if [ "$2" = pre-stop ]; then
        echo 'Sending graceful shutdown command to qemu-host'
        curl -sSm 52 'http://127.0.0.1:2210/read?command=6&timeout=50'
        echo
else
        echo "Unknown phase $2, ignoring"
fi
```

执行下列命令，以将上述脚本指定为虚拟机的hookscript：

```console
# chmod +x /var/lib/vz/snippets/hookscript-vdsm.sh
# qm set <VMID> -hookscript local:snippets/hookscript-vdsm.sh
update VM <VMID>: -hookscript local:snippets/hookscript-vdsm.sh
```

### 为虚拟机设置QGA设备

编辑虚拟机配置文件/etc/pve/qemu-server/&lt;VMID&gt;.conf，并将下列内容加入文件：

```text
args: -chardev 'socket,host=127.0.0.1,port=12345,server=off,reconnect=10,id=qga0' -device 'virtio-serial,id=qga0,bus=pci.0,addr=0x8' -device 'virtserialport,chardev=qga0,name=vchannel'
```

## 启动Virtual DSM

恭喜！您的Virtual DSM虚拟机现已就绪。请启动您的虚拟机，然后在浏览器的地址栏中输入虚拟机的IP地址，以连接到DSM。若您找不到虚拟机的IP地址，也可使用[Synology Web Assistant](https://finds.synology.com/)以连接到DSM。

![Synology Web Assistant](synology-web-assistant.png)

![DSM桌面](dsm-desktop.jpg)

如需要，您可执行下列命令，以清理安装过程中产生的文件。

```console
# rm -rf DSM_VirtualDSM_*.pat extract patch rd*
```

您可保留引导程序文件（synology_<wbr>kvmx64_<wbr>virtualdsm_<wbr>72806_<wbr>8A784779__.bin）和根文件系统目录（rootfs），以备您安装更多Virtual DSM虚拟机。若要删除它们，您可继续执行下列命令：

```console
# rm -rf synology_kvmx64_virtualdsm_*.bin rootfs
```
