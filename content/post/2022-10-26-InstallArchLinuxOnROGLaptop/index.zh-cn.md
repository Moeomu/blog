---
title: 在华硕玩家国度枪神4上安装ArchLinux
description: 安装ArchLinux+Wayland+Gnome+systemd-boot(安全启动)
date: 2022-10-26 19:14:09+0800
categories:
    - InstallationGuide
    - Linux
    - Solution
tags:
    - ArchLinux
    - ROG
    - Gnome
    - systemd-boot
    - NetworkManager
    - Wayland
image: https://cdn.staticaly.com/gh/Misakaou/imagestorage@master/20221026/00002-1464201810-install-system,-webpage-head-image,-beautiful,-ArchLinux.6mm7o4tdu64g.webp
---

本文来源: [MoeomuBlog](/zh-cn/posts/在华硕玩家国度枪神4上安装archlinux/)

## 本文简介

> Asus Strix ScarG532

- 如果您和我是一样的机器那么完全可以遵循本文一步一步地执行命令，最终一定会成功安装ArchLinux。我会详尽地将参考资料链接在旁边，您可以查看和修改成您想要的配置。
- 上面这条说明有违Arch之道，所以我是在开玩笑。请勿一步一步遵循下文配置，如果您遇到了和我一样的问题，您可以参考我的解决方案
- 我所安装的组件内容如下
  - 安全启动组件：PreLoader
  - 启动引导程序：systemd-boot
  - 网络和DHCP服务：NetworkManager (因为与Gnome集成更好)
  - 桌面环境：Gnome based on Wayland
  - 中文输入：ibus-libpinyin based on ibus (因为与Gnome集成更好)

## 安装ArchLinux

> 这一节完全遵循[Installation guide - ArchWiki](https://wiki.archlinux.org/title/Installation_guide)，您可以查看此官方文档链接获得更详细的解释

### 预安装

#### 启动盘制作

- 您可以访问[Download - ArchLinux](https://archlinux.org/download/)页面来获取官方启动镜像
- 下载好后推荐使用[balenaEtcher](https://www.balena.io/etcher/)将镜像写入U盘
- 从U盘启动制作好的ArchLinux安装环境系统

> 这三步由于过于简单所以不多赘述，如果需要帮助请在评论区留言

#### 验证UEFI模式启动

执行命令`ls /sys/firmware/efi/efivars`，如果有输出值那么证明是UEFI模式启动，后文中引导程序将使用UEFI

#### 检查并连接网络

> 下文将使用无线网卡进行网络连接

- 执行`rfkill`，如果输出如下则证明硬件正常

```sh
ID TYPE      DEVICE      SOFT      HARD
 0 bluetooth hci0   unblocked unblocked
 1 wlan      phy0   unblocked unblocked
```

- 执行`iwctl`进入网络连接配置工具界面，更多内容查看[iwctl - ArchWiki](https://wiki.archlinux.org/title/Iwctl)
- 执行`device list`查看设备列表
- 执行`station wlan0 scan`使用wlan0设备进行扫描
- 执行`station wlan0 get-networks`查看扫描到的网络列表
- 执行`station wlan0 connect SSID`连接网络
- 执行`ping archlinux.org`检测网络连接情况

#### 校准时间

> systemd-timesyncd检测到网络连接将会自动校准时间，因此不需要手动校准时间

- 执行`timedatectl status`查看并确保系统时间正确

#### 磁盘操作(**数据无价，务必谨慎操作**)

> 本节是一定不可以照搬的，您的磁盘数量可能和我不一样，您的磁盘分区情况也一定和我不一样  
> 务必参考[Partition the disks - ArchWiki](https://wiki.archlinux.org/title/Installation_guide#Partition_the_disks)进行磁盘分区

- 执行`fdisk -l`查看磁盘分区情况，按照您的需要自己分区
- 执行`fdisk /dev/nvme0n1`进行第一块SSD的操作，按`m`显示帮助，操作完成后保存退出`fdisk`界面，此工具在保存时才会写入硬盘分区表，请放心操作
- (危险)执行`mkfs.ext4 /dev/nvme0n1p4`格式化第一块SSD的第四个分区，在本例中将会是`/`的挂载点
- (危险)执行`mkfs.ext4 /dev/nvme2n1p1`格式化第三块SSD的第一个分区，在本例中将会是`/home`的挂载点
- 执行`e2label /dev/nvme0n1p4 "ArchSys"`将第一块SSD的第四个分区的标签设置为ArchSys，systemd-boot将会用到它
- 执行`e2label /dev/nvme2n1p1 "ArchData"`将第三块SSD的第一个分区的标签设置为ArchData，systemd-boot将会用到它
- 执行`blkid | grep Arch`查看磁盘标签的设置情况
- 执行`mount /dev/nvme0n1p4 /mnt`挂载磁盘，它将会是`/`
- 执行`mount --mkdir /dev/nvme0n1p1 /mnt/boot`挂载磁盘，它将会是`/boot`
- 执行`mount --mkdir /dev/nvme2n1p1 /mnt/home`挂载磁盘，它将会是`/home`
- 执行`mkswap /dev/nvme0n1p5`格式化第一块SSD的第五个分区为内存交换分区
- 执行`swapon /dev/nvme0n1p5`启用内存交换分区

### 开始安装

#### 修改选择镜像地址

> vim是一款命令行文本编辑器，如果不会用的话可以换为nano，如果已经进入了vim界面，可以在普通模式下输入英文冒号和q退出vim

- 执行`vim /etc/pacman.d/mirrorlist`编辑安装源镜像列表
- 将本行`Server = https://mirrors.ustc.edu.cn/archlinux/$repo/os/$arch`加到最上方，或者可以遵循自己的习惯

#### 安装必须的包

> intel-ucode是intel的微码更新包，推荐安装

- 执行`pacstrap -K /mnt base linux linux-firmware intel-ucode`安装系统基础包
- 执行`pacstrap /mnt networkmanager bluez vim sudo bash-completion openssh man git wget zsh`安装一些自己需要的软件

### 配置新系统

- 执行`genfstab -U /mnt >> /mnt/etc/fstab`创建系统分区表定义
- 执行`arch-chroot /mnt`将shell切换到新系统中执行命令
- 执行`ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime`将系统时区设置为亚洲-上海
- 执行`hwclock --systohc`同步硬件时间
- 执行`vim /etc/locale.gen`编辑系统的本地化设置，我的设定如下，可以作为参考。**注意：如果在这一步中不添加`zh_CN.UTF-8 UTF-8`，那么在Gnome的`Keyboard`选项中会找不到拼音输入**

```shell
en_US.UTF-8 UTF-8
zh_CN.UTF-8 UTF-8
```

- 执行`locale-gen`生成本地化设置
- 执行`vim /etc/locale.conf`编辑语言配置文件，我使用英文系统，输入了`LANG=en_US.UTF-8`，如果使用中文系统，改成`LANG=zh_CN.UTF-8`
- 执行`vim /etc/hostname`编辑本主机名称
- 执行`mkinitcpio -P`创建启动内存初始化镜像
- 执行`passwd`修改root用户密码

### 配置系统启动项

- 执行`bootctl --path=/boot install`将systemd-boot安装到/boot中
- 执行`systemctl enable systemd-boot-update.service`启动systemd-boot的自动更新服务
- 执行`vim /boot/loader/loader.conf`编辑systemd-boot启动配置文件

```text
default archlinux.conf
timeout 2
console-mode max
editor no
auto-entries 1
auto-firmware 1
```

- 执行`vim /boot/loader/entries/archlinux.conf`编辑ArchLinux启动项

```text
title Arch Linux
linux /vmlinuz-linux
initrd /intel-ucode.img
initrd /initramfs-linux.img
options root="LABEL=ArchSys" rw quiet splash
```

- 执行`vim /boot/loader/entries/arch-fallback.conf`编辑ArchLinux-Fallback启动项

```text
title   Arch Linux (fallback initramfs)
linux   /vmlinuz-linux
initrd  /intel-ucode.img
initrd  /initramfs-linux-fallback.img
options root="LABEL=ArchSys" rw quiet splash
```

### 其它

- 执行`systemctl enable NetworkManager.service`让NetworkManager的服务开机自启动
- 执行`systemctl enbale bluetooth.service`让蓝牙服务开机自启动
- 执行`exit`退出新系统环境
- 执行`reboot`重新启动系统

---

> **提示：到这里其实就可以结束了，但是如果作为主力系统来使用，ArchLinux现在的样子显然不合格，所以下文是更多的软件安装参考**

---

## 更多安装配置

### 连接网络

> 使用NetworkManager命令行工具

- 执行`nmcli device wifi list`查看wifi列表
- 执行`nmcli device wifi connect SSID_or_BSSID password your_password`连接网络
- 执行`nmcli connection show`查看当前连接

> 您可以在[nmcli example - ArchWiki](https://wiki.archlinux.org/title/NetworkManager#nmcli_examples)文章中查看更多用法

### 新建用户

- 执行`useradd -m misaka`来新建一个名为misaka的用户
- 执行`export EDITOR=vim && visudo`编辑此文件，按照提示，将新用户添加到特权用户组，让新用户可以正常使用sudo进行提权操作
- 执行`usermod -a -G wheel misaka`将misaka用户添加到wheel用户组中，这样做的好处是让Gnome可以正确识别这个新用户的权限，在Gnome中进行root权限操作时默认弹出这个用户名的验证
- 执行`passwd misaka`修改新用户的密码

### 安装图形界面Gnome

> 提示：下方写的命令要安装的软件很多，是因为Gnome包中有很多我不需要的软件，所以只能手动选择希望安装的软件，您也同样可以用更简单的`pacman -S gnome`来代替下方的安装命令，随您喜好

- 执行下方的命令将安装Gnome的基础组件
- `pacman -S file-roller gdm gedit gnome-backgrounds gnome-calculator gnome-control-center gnome-disk-utility gnome-logs gnome-menus gnome-settings-daemon gnome-shell gnome-shell-extensions gnome-software gnome-system-monitor gnome-terminal  gvfs nautilus seahorse seahorse-nautilus xdg-user-dirs-gtk`

> 如果安装时让您选择使用`jack2`还是`pipewire`，请选择`pipewire`。这是系统的声音服务。

- 执行`systemctl enable gdm.service`让Gnome登陆界面开机自启动，它将会在下一次重启时启动Gnome

### 安装中文输入法和字体

- 执行`pacman -S ttf-dejavu wqy-zenhei wqy-microhei adobe-source-han-sans-cn-fonts adobe-source-han-serif-cn-fonts`安装中文字体
- 执行`ibus ibus-libpinyin`安装ibus和ibus的智能拼音输入法

### 重新启动并以普通用户身份登陆

- 执行`reboot`
- 启动Terminal

### Gnome窗口控制按钮设置

- 执行`gsettings set org.gnome.desktop.wm.preferences button-layout "close,minimize,maximize:"`将会把所有窗口的关闭、最小化、最大化按钮显示出来并放到左侧

### NVIDIA驱动设置

> **警告：本小节执行之中请勿重新启动**

- 执行`sudo pacman -S nvidia nvidia-settings`安装NVIDIA显卡驱动和设置程序

> 它将会把默认的窗口系统从Wayland换成X11，如果想继续使用Wayland作为窗口系统，请执行以下命令

- 执行`sudo ln -s /dev/null /etc/udev/rules.d/61-gdm.rules`禁用udev的在GDM中禁用Wayland的规则
- 执行`gsettings set org.gnome.mutter experimental-features '["kms-modifiers"]'`启用gsettings的kms-modifiers
- 执行`sudo vim /boot/loader/entries/archlinux.conf`编辑启动引导项将最后一行改成`options root="LABEL=ArchSys" rw quiet splash nvidia-drm.modeset=1`
- 执行`sudo vim /etc/mkinitcpio.conf`编辑modsettings将`MODULES=()`改成`nvidia nvidia_modeset nvidia_uvm nvidia_drm`并保存文件。更多帮助信息请参考[DRM_kernel_mode_setting - ArchWiki](https://wiki.archlinux.org/title/NVIDIA#DRM_kernel_mode_setting)
- 执行`sudo mkinitcpio -P`

> 本节参考来源：[Use Wayland with proprietary NVIDIA drivers - Manjaro Forum](https://forum.manjaro.org/t/howto-use-wayland-with-proprietary-nvidia-drivers/36130)

### 安全启动

> 本节参考来源：[Unified Extensible Firmware Interface/Secure Boot - ArchWiki](https://wiki.archlinux.org/title/Unified_Extensible_Firmware_Interface/Secure_Boot)

- 从[Linux Foundation Secure Boot System Released - James Bottomley's random Pages](https://blog.hansenpartnership.com/linux-foundation-secure-boot-system-released/)下载PreLoader.efi和HashTool.efi
- 如果您按照上文配置，那么将这两个文件复制到您的`/boot/EFI/systemd`目录中
- 将`/boot/EFI/systemd/systemd-bootx64.efi`重命名为`/boot/EFI/systemd/loader.efi`

```shell
cp your_download_path/{PreLoader,HashTool}.efi esp/EFI/systemd
cp esp/EFI/systemd/systemd-bootx64.efi esp/EFI/systemd/loader.efi
```

- 执行`sudo pacman -S efibootmgr`安装efibootmgr
- **请注意可能您的磁盘和我不同**，执行`sudo efibootmgr --verbose --disk /dev/nvme0n1 --part 1 --create --label "Arch Linux Security Boot" --loader /boot/EFI/systemd/PreLoader.efi`添加一个NVRAM启动条目

### 安装Gnome扩展程序

- 执行`flatpak install com.mattjakeman.ExtensionManager`安装Gnome扩展程序管理器
- 在Gnome扩展程序管理器中搜索安装并配置下列扩展程序
  - TrayIcons: Reloaded(强烈推荐)
  - Blur My Shell
  - Gnome Clipboard
  - Compiz alike magic lamp effect
  - Dash to dock
  - No activities button
  - Open Weather
  - Privacy Quick Settings Menu
  - Vitals

## 其他问题

> 下文会用到Aur软件源，可以在[Aur - ArchLinux](https://aur.archlinux.org/)中找到更多信息。  
> `paru`是一个Aur软件源的实用工具，可以自动获取安装Aur软件源内的程序，请访问[Morganamilo/paru - Github](https://github.com/Morganamilo/paru)了解查看paru的安装和使用。  

### Gnome无法正常挂载NTFS外接磁盘

> 可能是Gnome中的gvfs需要使用ntfs-3g中的一些实用工具才可以正确挂载NTFS磁盘。Linux内核5.15版本之后将ntfs驱动内置，因此可以通过以上命令单独安装使用内核ntfs驱动的ntfs实用工具。

- 执行`paru ntfsprogs-ntfs3`

### 没有声音

> 本节参考：
>
> - [Problem summary of installing Ubuntu on AsusStrixScarG532 - MoeomuBlog](https://blog.moeomu.com/posts/problem-summary-of-installing-ubuntu-on-asusstrixscarg532/#system-sound-problems)
> - [ASUS ROG G533QS - ArchWiki](https://wiki.archlinux.org/title/ASUS_ROG_G533QS)

有以上两个解决方式，具体使用哪个请根据自己的喜好

### GPG问题

> 本节参考：[GnuPG - ArchWiki](https://wiki.archlinux.org/title/GnuPG)

#### ioctl错误

> 指南：[Invalid_IPC_response_and_Inappropriate_ioctl_for_device - ArchWiki](https://wiki.archlinux.org/title/GnuPG#Invalid_IPC_response_and_Inappropriate_ioctl_for_device)

- 将`export GPG_TTY=$(tty)`添加到环境变量中，例如`.bashrc`，`.zshrc`等文件中

#### 将pinentry配置为Gnome3样式

> 不配置将会导致GPG无法找到图形界面的输入密码界面程序，进而导致更多的图形界面程序错误

- 执行`echo pinentry-program /usr/bin/pinentry-gnome3 >> .gnupg/gpg-agent.conf`

### 键盘灯光和电源控制

> 在[ASUS Linux](https://asus-linux.org/)阅读更多

- 执行`paru asusctl`
- 执行`paru rog-control-center`

## 参考文献

- [HowTo] Use Wayland with proprietary NVIDIA drivers. (2022, February 10). Manjaro Linux Forum. <https://forum.manjaro.org/t/howto-use-wayland-with-proprietary-nvidia-drivers/36130>
- Arch Linux - Downloads. (n.d.). Archlinux.org. <https://archlinux.org/download/>
- Arch Linux 源使用帮助 — USTC Mirror Help 文档. (n.d.). Mirrors.ustc.edu.cn. Retrieved October 26, 2022, from <https://mirrors.ustc.edu.cn/help/archlinux.html>
- ASUS NoteBook Linux. (n.d.). Asus-Linux.org. Retrieved October 26, 2022, from <https://asus-linux.org/>
- ASUS ROG G533QS - ArchWiki. (n.d.). Wiki.archlinux.org. Retrieved October 26, 2022, from <https://wiki.archlinux.org/title/ASUS_ROG_G533QS>
- AUR (en) - Home. (n.d.). Aur.archlinux.org. <https://aur.archlinux.org/>
- balenaEtcher - Home. (2020). BalenaEtcher. <https://www.balena.io/etcher/>
- GnuPG - ArchWiki. (2018). Archlinux.org. <https://wiki.archlinux.org/title/GnuPG>
- Installation guide - ArchWiki. (n.d.). Wiki.archlinux.org. <https://wiki.archlinux.org/title/Installation_guide>
- iwd - ArchWiki. (n.d.). Wiki.archlinux.org. Retrieved October 26, 2022, from <https://wiki.archlinux.org/title/Iwctl>
- Linux Foundation Secure Boot System Released | James Bottomley’s random Pages. (n.d.). Retrieved October 26, 2022, from <https://blog.hansenpartnership.com/linux-foundation-secure-boot-system-released/>
- Lulu. (2022, October 26). Paru. GitHub. <https://github.com/Morganamilo/paru>
- Misaka. (2022, October 20). Problem summary of installing Ubuntu on AsusStrixScarG532. Moeomu Blog. <https://blog.moeomu.com/posts/problem-summary-of-installing-ubuntu-on-asusstrixscarg532/#system-sound-problems>
- NetworkManager - ArchWiki. (n.d.). Wiki.archlinux.org. <https://wiki.archlinux.org/title/NetworkManager>
- NVIDIA - ArchWiki. (n.d.). Wiki.archlinux.org. Retrieved October 26, 2022, from <https://wiki.archlinux.org/title/NVIDIA>
- Unified Extensible Firmware Interface/Secure Boot - ArchWiki. (n.d.). Wiki.archlinux.org. <https://wiki.archlinux.org/title/Unified_Extensible_Firmware_Interface/Secure_Boot>
