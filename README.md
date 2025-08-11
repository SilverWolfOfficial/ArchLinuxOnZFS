# 在ZFS文件系统上安装Arch Linux 最后更新于2025年08月10日

### 1.构建带有ZFS文件系统支持的系统镜像

以下步骤请在一个可用的Arch Linux上进行。如果没有可用的Arch Linux本节末尾提供了笔者已经构建好的系统镜像。

```bash
$ sudo pacman -S archiso
#安装archiso

$ mkdir ~/Temps
#创建~/Temps目录

$ cp -r /usr/share/archiso/configs/releng ~/Temps
#复制/usr/share/archiso/configs/releng目录到~/Temps目录

$ cd ~/Temps/releng
#切换到~/Temps/releng目录

$ vim ~/Temps/releng/pacman.conf
#编辑~/Temps/releng/pacman.conf文件，在末尾加入以下内容：
[archzfs]
SigLevel = Never
Server = https://github.com/archzfs/archzfs/releases/download/experimental
#添加archzfs软件源

$ vim ~/Temps/releng/packages.x86_64
#编辑~/Temps/releng/packages.x86_64文件，在末尾加入以下内容（为美观也可按首字母顺序加入对应位置）：
linux-headers
zfs-linux
zfs-linux-headers
zfs-utils
#添加ZFS所需要的包

$ cp ~/Temps/releng/pacman.conf ~/Temps/releng/airootfs/etc
#复制~/Temps/releng/pacman.conf文件到~/Temps/releng/airootfs/etc目录

sudo mkarchiso -v -w ./work -o ./out ./
#构建系统镜像（构建的系统镜像在~/Temps/releng/out目录下,请将系统镜像复制到其他位置备用）
```

[也可以点击此处下载笔者已经构建好的系统镜像](https://github.com/SilverWolfOfficial/archiso-zfs/releases/download/latest/archlinux-zfs-2025.08.11-x86_64.iso)

验证引导方式、联网、换源、分区等和一般的Arch Linux安装无差别，不再赘述。关于分区部分，请用自己熟悉的分区工具（如cfdisk、fdisk、gdisk等）进行分区，但不要格式化和挂载。请确保分区至少有三个，即：一个EFI System分区、一个Linux swap分区、一个Linux filesystem分区。

### 2.格式化 & 挂载

以下格式化和挂载相关步骤是以NVME协议硬盘为例，如果是SATA协议硬盘，请记住：nvme0n1等于sda；nvme0n1p1等于sda1；nvme0n1p2等于sda2；nvme0n1p3等于sda3。

```bash
$ mkfs.vfat -F 32 -n BOOT /dev/nvme0n1p1
#将nvme0n1p1格式化为vfat（设置为FAT32格式），创建标签为BOOT

$ mkswap -L SWAP /dev/nvme0n1p2
#将nvme0n1p2设置为swap，创建标签为SWAP

$ zpool create -f -o ashift=12 \
               -O acltype=posixacl \
               -O relatime=on \
               -O xattr=sa \
               -O dnodesize=legacy \
               -O normalization=formD \
               -O mountpoint=none \
               -O canmount=off \
               -O devices=off \
               -O compression=lz4 \
               -R /mnt ARCH /dev/nvme0n1p3
#在/dev/nvme0n1p3创建名为ARCH的ZFS存储池，开启ZFS相关功能并将其挂载到/mnt目录

$ zfs create -o mountpoint=none ARCH/DATA
#创建ARCH/DATA数据集且不设置挂载点

$ zfs create -o mountpoint=none ARCH/SYSTEM
#创建ARCH/SYSTEM数据集且不设置挂载点

$ zfs create -o mountpoint=/ -o canmount=noauto ARCH/SYSTEM/ROOT
#创建ARCH/SYSTEM/ROOT数据集并设置挂载点为/且不自动挂载

$ zfs create -o mountpoint=/home ARCH/DATA/HOME
#创建ARCH/DATA/HOME数据集且设置挂载点为/home

$ zpool export ARCH
#导出ARCH存储池

$ zpool import -d /dev/nvme0n1p3 -R /mnt ARCH -N
#重新导入ARCH存储池

$ zfs mount ARCH/SYSTEM/ROOT
#挂载ARCH/SYSTEM/ROOT数据集

$ zfs mount -a
#挂载所有数据集

$ zpool set bootfs=ARCH/SYSTEM/ROOT ARCH
#启动ARCH存储池时挂载ARCH/SYSTEM/ROOT数据集

$ zpool set cachefile=/etc/zfs/zpool.cache ARCH
#生成ARCH存储池的配置信息

$ mkdir -p /mnt/etc/zfs
#创建/mnt/etc/zfs目录

$ cp /etc/zfs/zpool.cache /mnt/etc/zfs
#复制/etc/zfs/zpool.cache文件到/mnt/etc/zfs目录

$ mkdir /mnt/boot
#创建/mnt/boot目录（不要创建/mnt/boot/efi目录，会导致在后续步骤中系统无法启动）

$ mount /dev/nvme0n1p1 /mnt/boot
#将nvme0n1p1挂载到/mnt/boot目录（不要挂载到/mnt/boot/efi目录，会导致在后续步骤中系统无法启动）

$ swapon /dev/nvme0n1p2
#激活nvme0n1p2为交换分区
```

### 3.安装基本的包 & 关于ZFS文件系统的配置

```bash
$ pacstrap -K /mnt linux linux-firmware sof-firmware linux-headers base base-devel vim bash-completion networkmanager dosfstools exfatprogs ntfs-3g pacman-contrib zfs-linux zfs-linux-headers zfs-utils
#安装必要的软件包

$ genfstab -U -p /mnt > /mnt/etc/fstab
#生成/mnt/etc/fstab文件

$ vim /mnt/etc/fstab
#编辑/mnt/etc/fstab文件，删除所有关于ZFS的内容，ZFS文件系统不依赖该文件进行挂载，仅保留/boot所在的行和swap所在的行即可

$ arch-chroot /mnt
#进入目标系统

$ vim /etc/pacman.conf
#编辑/etc/pacman.conf文件，在末尾加入以下内容：
[archzfs]
Server = https://github.com/archzfs/archzfs/releases/download/experimental
#添加archzfs软件源

$ pacman-key --init && pacman-key --recv-keys 3A9917BF0DED5C13F69AC68FABEC0A1208037BE9 && pacman-key --lsign-key 3A9917BF0DED5C13F69AC68FABEC0A1208037BE9
#导入archzfs软件源的公钥

$ pacman -Sy
#同步软件源数据

$ vim /etc/mkinitcpio.conf
#编辑/etc/mkinitcpio.conf，将MODULES=()修改为MODULES=(zfs)，将HOOKS=(base udev autodetect microcode modconf kms keyboard keymap consolefont block filesystems fsck)修改为HOOKS=(base udev autodetect microcode modconf kms keyboard keymap consolefont block zfs filesystems)

$ zgenhostid $(hostid)
#生成hostid

$ mkinitcpio -P
#重新生成initramfs

$ systemctl enable zfs.target zfs-import.target zfs-volumes.target zfs-import-cache.service zfs-volume-wait.service
#开机自启ZFS文件系统相关服务
```

其他安装Arch Linux所需的步骤和一般的Arch Linux安装无差别，请自主进行配置，不再赘述。

### 4.引导管理器的安装 & 退出目标系统

```bash
$ bootctl install
#使用systemd-boot作为引导管理器，grub亦可，不再赘述

$ vim /boot/loader/entries/archlinux.conf
#创建/boot/loader/entries/archlinux.conf文件，加入以下内容：
title   Arch Linux
linux   /vmlinuz-linux
initrd  /initramfs-linux.img
options zfs=ARCH/SYSTEM/ROOT rw
#配置Arch Linux的引导条目

$ exit
#退出目标系统

$ umount /mnt/boot
#卸载/mnt/boot目录

$ zfs umount -a
#卸载所有数据集

$ zpool export ARCH
#导出ARCH存储池

$ reboot
#重启
```

# THE END
