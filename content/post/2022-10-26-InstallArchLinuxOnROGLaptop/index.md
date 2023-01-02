---
title: Installing ArchLinux on ASUS ROG Strix Scar G532
description: Install ArchLinux+Wayland+Gnome+systemd-boot(security boot)
date: 2022-10-26 19:14:09+0800
categories:
    - SoftwareGuide
tags:
    - ArchLinux
    - Linux
image: https://cdn.staticaly.com/gh/Misakaou/imagestorage@master/20221026/00002-1464201810-install-system,-webpage-head-image,-beautiful,-ArchLinux.6mm7o4tdu64g.webp
---

Source of this article: [MoeomuBlog](/posts/installing-archlinux-on-asus-rog-strix-scar-g532/)

## Introduction to this article

> Asus Strix ScarG532

- If you have the same machine as I do then you can follow this article exactly step by step and you will definitely end up with a successful installation of ArchLinux. I will exhaustively link the reference next to it so you can view and modify it to the configuration you want.
- This note above is against the Arch way, so I am kidding. Please do not follow the configuration below step by step, and if you encounter the same problem as I did, you can refer to my solution
- The components I have installed are as follows
  - Secure boot component: PreLoader
  - Bootloader: systemd-boot
  - Network and DHCP service: NetworkManager (because of better integration with Gnome)
  - Desktop environment: Gnome based on Wayland
  - Chinese input: ibus-libpinyin based on ibus (because of better integration with Gnome)

## Installing ArchLinux

> This section follows exactly the [Installation guide - ArchWiki](https://wiki.archlinux.org/title/Installation_guide), you can check this official documentation link for a more detailed explanation

### Pre-installation

#### Boot disk creation

- You can visit the [Download - ArchLinux](https://archlinux.org/download/) page to get the official boot image
- Once downloaded, it is recommended to use [balenaEtcher](https://www.balena.io/etcher/) to write the image to a USB stick
- Boot the created ArchLinux installation environment from the USB stick

> These three steps are too simple so do not repeat, if you need help please leave a message in the comments section

#### Verify UEFI mode boot

Execute the command `ls /sys/firmware/efi/efivars`, if there is an output value then it proves that it is UEFI mode boot, later in the article the bootloader will use UEFI

#### Check and connect to the network

> The following section will use the wireless card for network connection

- Run ``rfkill``, if the output is as follows then the hardware is fine

```sh
ID TYPE DEVICE SOFT HARD
 0 bluetooth hci0 unblocked unblocked
 1 wlan phy0 unblocked unblocked
```

- Execute `iwctl` to enter the network connection configuration tool interface, see [iwctl - ArchWiki](https://wiki.archlinux.org/title/Iwctl) for more details
- Execute `device list` to view the list of devices
- Execute `station wlan0 scan` to scan with wlan0 devices
- Execute `station wlan0 get-networks` to see the list of scanned networks
- Execute `station wlan0 connect SSID` to connect to the network
- Execute `ping archlinux.org` to check network connectivity

#### calibrate the time

> systemd-timesyncd will automatically calibrate the time when it detects a network connection, so there is no need to manually calibrate the time

- Run `timedatectl status` to check and make sure the system time is correct

#### disk operations (**data is priceless, be careful**)

> This section must not be copied, you may have a different number of disks than I do, and your disk partition situation must be different than mine  
> Be sure to refer to [Partition the disks - ArchWiki](https://wiki.archlinux.org/title/Installation_guide#Partition_the_disks) for disk partitioning

- Run `fdisk -l` to see how your disk is partitioned, and partition it as you see fit
- Execute `fdisk /dev/nvme0n1` for the first SSD operation, press `m` to show help, save and exit the `fdisk` interface after the operation is finished, this tool will write to the hard disk partition table only when it is saved, please feel free to operate it
- (Danger) Execute `mkfs.ext4 /dev/nvme0n1p4` to format the fourth partition of the first SSD, in this case it will be the mount point of `/`
- (Danger) Execute `mkfs.ext4 /dev/nvme2n1p1` to format the first partition of the third SSD, which in this case will be the mount point of `/home`.
- Execute `e2label /dev/nvme0n1p4 "ArchSys"` to set the label of the fourth partition of the first SSD to ArchSys, which will be used by systemd-boot
- Execute `e2label /dev/nvme2n1p1 "ArchData"` to set the label of the first partition of the third SSD to ArchData, which will be used by systemd-boot
- Execute `blkid | grep Arch` to check the disk label setting
- Execute `mount /dev/nvme0n1p4 /mnt` to mount the disk, it will be `/`
- Execute `mount --mkdir /dev/nvme0n1p1 /mnt/boot` to mount the disk, it will be `/boot`
- Execute `mount --mkdir /dev/nvme2n1p1 /mnt/home` to mount the disk, it will be `/home`
- Execute `mkswap /dev/nvme0n1p5` to format the fifth partition of the first SSD as a memory swap partition
- Execute `swapon /dev/nvme0n1p5` to enable the memory swap partition

### Start installation

#### change the address of the selected mirror

> vim is a command line text editor, if you don't know how to use it, you can switch to nano, if you are already in the vim interface, you can exit vim in normal mode by typing an English colon and q

- Execute `vim /etc/pacman.d/mirrorlist` to edit the installation source image list
- Add this line `Server = https://mirrors.ustc.edu.cn/archlinux/$repo/os/$arch` to the top, or you can follow your own custom

#### to install the necessary packages

> intel-ucode is the microcode update package for intel, recommended to install

- Execute `pacstrap -K /mnt base linux linux-firmware intel-ucode` to install the system base package
- Execute `pacstrap /mnt networkmanager bluez vim sudo bash-completion openssh man git wget zsh` to install some software you need

### Configure the new system

- Execute `genfstab -U /mnt >> /mnt/etc/fstab` to create a system partition table definition
- Execute `arch-chroot /mnt` to switch the shell to the new system and execute the command
- Execute `ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime` to set the system time zone to Asia-Shanghai
- Execute `hwclock --systohc` to synchronize the hardware time
- Execute `vim /etc/locale.gen` to edit the system's localization settings, my settings are as follows, which can be used as a reference. **Note: If you don't add `zh_CN.UTF-8 UTF-8` in this step, then you won't find Pinyin input in Gnome's `Keyboard` option**

```shell
en_US.UTF-8 UTF-8
zh_CN.UTF-8 UTF-8
```

- Execute `locale-gen` to generate localization settings
- Execute `vim /etc/locale.conf` to edit the language configuration file, I use English system, enter `LANG=en_US.UTF-8`, if you use Chinese system, change it to `LANG=zh_CN.UTF-8`
- Execute `vim /etc/hostname` to edit this host name
- Execute `mkinitcpio -P` to create a boot memory initialization image
- Execute `passwd` to change the root user password

### Configure system startup items

- Execute `bootctl --path=/boot install` to install systemd-boot into /boot
- Execute `systemctl enable systemd-boot-update.service` to start systemd-boot's automatic update service
- Execute ``vim /boot/loader/loader.conf`` to edit the systemd-boot startup configuration file

```text
default archlinux.conf
timeout 2
console-mode max
editor no
auto-entries 1
auto-firmware 1
```

- Execute ``vim /boot/loader/entries/archlinux.conf`` to edit ArchLinux boot entries

```text
title Arch Linux
linux /vmlinuz-linux
initrd /intel-ucode.img
initrd /initramfs-linux.img
options root="LABEL=ArchSys" rw quiet splash
```

- Execute ``vim /boot/loader/entries/arch-fallback.conf`` to edit the ArchLinux-Fallback boot entry

```text
title Arch Linux (fallback initramfs)
linux /vmlinuz-linux
initrd /intel-ucode.img
initrd /initramfs-linux-fallback.img
options root="LABEL=ArchSys" rw quiet splash
```

### Other

- Execute `systemctl enable NetworkManager.service` to make NetworkManager's service start on boot
- Execute `systemctl enbale bluetooth.service` to make the Bluetooth service start on boot
- Execute `exit` to exit the new system environment
- Execute `reboot` to reboot the system

---

> **Hint: You can actually end up here, but if you use it as a primary system, ArchLinux as it is now is clearly not up to the task, so here is more software installation reference**

---

## More installation configurations

### Connecting to the network

> Using the NetworkManager command line tool

- Execute `nmcli device wifi list` to see the wifi list
- Execute `nmcli device wifi connect SSID_or_BSSID password your_password` to connect to the network
- Execute `nmcli connection show` to see the current connection

> You can see more usage in the [nmcli example - ArchWiki](https://wiki.archlinux.org/title/NetworkManager#nmcli_examples) article

### Create a new user

- Execute `useradd -m misaka` to create a new user named misaka
- Execute `export EDITOR=vim && visudo` to edit this file and follow the prompts to add the new user to the privileged users group, so that the new user can use sudo for privileged operations normally
- Execute `usermod -a -G wheel misaka` to add the misaka user to the wheel user group, the advantage of this is that Gnome can correctly identify the privileges of this new user, and the authentication of this user name will pop up by default when doing root privileges in Gnome
- Execute `passwd misaka` to change the password of the new user

### Install Gnome GUI

> Hint: The command below is written to install a lot of software because there are a lot of software in the Gnome package that I don't need, so I have to manually select the software I want to install, or you can use the simpler `pacman -S gnome` instead of the installation command below, as you like

- Executing the following command will install the base components of Gnome
- `pacman -S file-roller gdm gedit gnome-backgrounds gnome-calculator gnome-control-center gnome-disk-utility gnome-logs gnome-menus gnome- settings-daemon gnome-shell gnome-shell-extensions gnome-software gnome-system-monitor gnome-terminal gvfs nautilus seahorse seahorse- nautilus xdg-user-dirs-gtk`

> If the installation gives you a choice between using `jack2` or `pipewire`, choose `pipewire`. This is the system's sound service.

- Run `systemctl enable gdm.service` to make the Gnome login screen boot up and self start, it will start Gnome on the next reboot

### Install Chinese input method and fonts

- Execute `pacman -S ttf-dejavu wqy-zenhei wqy-microhei adobe-source-han-sans-cn-fonts adobe-source-han-serif-cn-fonts` to install Chinese fonts
- Execute `ibus ibus-libpinyin` to install ibus and ibus' Smart Pinyin input method

### Reboot and login as normal user

- Execute `reboot`
- Start Terminal

### Gnome window control button settings

- Execute `gsettings set org.gnome.desktop.wm.preferences button-layout "close,minimize,maximize:"` which will show all windows close, minimize and maximize buttons and put them on the left side

### NVIDIA driver settings

> **Warning: Do not reboot while this section is running**

- Run `sudo pacman -S nvidia nvidia-settings` to install the NVIDIA driver and setup program

> It will change the default window system from Wayland to X11, if you want to continue using Wayland as the window system, please execute the following command

- Execute `sudo ln -s /dev/null /etc/udev/rules.d/61-gdm.rules` to disable udev's rules for disabling Wayland in GDM
- Execute `gsettings set org.gnome.mutter experimental-features '["kms-modifiers"]'` to enable kms-modifiers for gsettings
- Execute `sudo vim /boot/loader/entries/archlinux.conf` to edit the boot entry and change the last line to `options root="LABEL=ArchSys" rw quiet splash nvidia-drm.modeset=1`
- Run `sudo vim /etc/mkinitcpio.conf` to edit modsettings to change `MODULES=()` to `nvidia nvidia_modeset nvidia_uvm nvidia_drm` and save the file. For more help information please refer to [DRM_kernel_mode_setting - ArchWiki](https://wiki.archlinux.org/title/NVIDIA#DRM_kernel_mode_setting)
- Execute `sudo mkinitcpio -P`

> Reference source for this section: [Use Wayland with proprietary NVIDIA drivers - Manjaro Forum](https://forum.manjaro.org/t/howto-use-wayland-with-proprietary-nvidia-drivers/36130)

### Secure Boot

> Reference source for this section: [Unified Extensible Firmware Interface/Secure Boot - ArchWiki](https://wiki.archlinux.org/title/Unified_Extensible_Firmware_Interface/Secure_Boot)

- Downloaded from [Linux Foundation Secure Boot System Release - James Bottomley's Random Page](https://blog.hansenpartnership.com/linux-foundation-secure-boot-system-released/) PreLoader.efi and HashTool.efi
- If you configured them as above, then copy these two files to your `/boot/EFI/systemd` directory
- Rename ``/boot/EFI/systemd/systemd-bootx64.efi`` to ``/boot/EFI/systemd/loader.efi``.

```shell
cp your_download_path/{PreLoader,HashTool}.efi esp/EFI/systemd
cp esp/EFI/systemd/systemd-bootx64.efi esp/EFI/systemd/loader.efi
```

- Run `sudo pacman -S efibootmgr` to install efibootmgr
- **Please note that maybe your disk is different from mine**, run `sudo efibootmgr --verbose --disk /dev/nvme0n1 --part 1 --create --label "Arch Linux Security Boot" --loader /boot/EFI/ systemd/PreLoader.efi` adds an NVRAM boot entry

### Install the Gnome extensions

- Run `flatpak install com.mattjakeman.ExtensionManager` to install Gnome Extension Manager
- Search for and install the following extensions in the Gnome Extension Manager and configure them
  - TrayIcons: reinstall (highly recommended)
  - Blur My Shell
  - Gnome Clipboard
  - Compiz similar magic light effect
  - Run to the dock
  - No activity button
  - Open Weather
  - Privacy quick settings menu
  - Vital Signs

## Other issues

> The following section will use the Aur software repository, which can be found in [Aur - ArchLinux](https://aur.archlinux.org/) for more information.  
> `paru` is an Aur repositories utility that automatically fetches programs within the installed Aur repositories, visit [Morganamilo/paru - Github](https://github.com/Morganamilo/paru) to see how to install and use paru.  

### Gnome can't mount NTFS external disk properly

> It may be that gvfs in Gnome needs to use some utilities in ntfs-3g to mount NTFS disks properly. ntfs driver is built in after Linux kernel version 5.15, so you can install ntfs utilities using kernel ntfs driver separately with the above command.

- Execute `paru ntfsprogs-ntfs3`

### No sound

> This section references.
>
> - [Problem summary of installing Ubuntu on AsusStrixScarG532 - MoeomuBlog](https://blog.moeomu.com/posts/problem-summary-of-installing-ubuntu-on-asusstrixscarg532/#system-sound-problems)
> - [ASUS ROG G533QS - ArchWiki](https://wiki.archlinux.org/title/ASUS_ROG_G533QS)

There are two solutions above, please use which one according to your preference

### GPG problem

> Reference for this section: [GnuPG - ArchWiki](https://wiki.archlinux.org/title/GnuPG)

#### ioctl error

> Guide: [Invalid_IPC_response_and_Inappropriate_ioctl_for_device - ArchWiki](https://wiki.archlinux.org/title/GnuPG#Invalid_IPC_response_and_Inappropriate_ioctl_for_device)

- Add `export GPG_TTY=$(tty)` to environment variables such as `.bashrc`, `.zshrc`, etc.

#### configure pinentry as Gnome3 style

> Failure to do so will cause GPG to fail to find the GUI password entry program, which in turn will cause more GUI program errors

- Execute `echo pinentry-program /usr/bin/pinentry-gnome3 >> .gnupg/gpg-agent.conf`

### Keyboard lighting and power control

> read more at [ASUS Linux](https://asus-linux.org/)

- Execute `paru asusctl`
- Execute `paru rog-control-center`

## Reference

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
