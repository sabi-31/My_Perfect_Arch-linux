# My_Perfect_Arch-linux

### This is a guide/personal reference/explanation of an ideal arch linux base install + all the good ricing on top of it. I am mainly creating this as a personal reference guide, but hope it's also helpful to someone else learning this stuff.

### List of Resources that helped me and from which I picked up a lot of the stuff outlined in this guide, huge thanks to the original authors
1. [This Extensive guide by Walian which was the main motivator.](https://walian.co.uk/arch-install-with-secure-boot-btrfs-tpm2-luks-encryption-unified-kernel-images.html)
2. [This guide based on fedora by Madhu@ sysguides.](https://sysguides.com/install-fedora-with-snapshot-and-rollback-support)
3. [And offcourse, the Arch btw Wiki.](https://wiki.archlinux.org/title/Main_page)
4. [Good article on UEFI stubs and UKI's](https://linderud.dev/blog/mkinitcpio-v31-and-uefi-stubs/)

### Summary
In this guide, I will do a minimal arch installation from scratch. I will be installing the OS on a fresh drive. I will partition it using btrfs, as I want to setup up snapper and rollback features. I will also encrypt my root partition using LUKS2. On top of all this, I will generate my own secure boot keys and sign all the bootable binaries. The kernel images will be generated my mkinitpcio as UKI's. Finally I will install Hyprland and other related utilities to make it look good and perform well. I will try to make it as modular and platform agnostic as possible to account for other people using this guide. Let's go:

# Table of Contents
1. [Pre-Install](#Pre-Install) 
2. [Installation](#Installation)
3. [Post-Install](#Post-Install)
4. [Ricing](#Ricing)

## Pre-Install
1. Download the arch.iso file and burn it to a usb drive (diy).
2. Disable secure boot from your motherboard's firmware if it's enabled, as otherwise your usb wont boot.
3. If you are dual-booting and/or have multiple disks in your system, make sure to identify which drive you want to install the linux system on. I would highly highly recommend a separate drive for arch or any other linux distro for that matter in case of dual booting with windows. 
4. Identify your locale and timezone data.

## Installation
1. Boot into the iso
2. Connect to the internet if using wifi (if you have ethernet, skip this step)
	```
	iwctl
	```
	```
	device list
	```
	```
	device wlan0 set-property Powered on
	```
	```
	station wlan0 scan
	```
	```
	station wlan0 get-networks
	```
	```
	station wlan0 connect <wifi-name>
	```
	
	Check connection using 
	```
	ping google.com
	```
3.  Check all disks
    ```
	   lsblk
	```
	Find the name of your target disk, in my case it is nvme1n1
	Save this in a variable:
		```
		DISK=/dev/nvme1n1
		```

		```
		echo $DISK
		```
	   
4. Create partitions using fdisk
this will create a new gpt table on the disk, and then create two partitons - a 512MB eFI partiton and a root partition with the rest of the space.
````
	a. fdisk $DISK
	b. g
	c. n
	d. default
	e. default
	f. +512M
	g. n
	h. default
	i. default
	j. default
	k. t
	l. 1
	m. p
	n. w
````
Run lsblk to verify partitions

5. Encrypt Root Partiton 
	#### For btrfs, we need to encrypt first and then create the btrfs partitions, for ext 4, we can create partitions first and then encrypt
   ```
		cryptsetup luksFormat --type luks2 ${DISK}2
		cryptsetup open --type luks ${DISK}2 root
   ```

6. Format Partitons
	```
		mkfs.vfat -F32 -n EFI ${DISK}1
		mkfs.btrfs -f -L root /dev/mapper/root
	```

	### blkid -o list

7. Mount Partitions
	Compression level to use and algo
	strFAT32options='rw,relatime,fmask=0022,dmask=0022,codepage=437,iocharset=iso8859-1,shortname=mixed,utf8'; ### my default options to mount FAT-32 file-systems
	strBTRFSoptions='defaults,noatime,space_cache,ssd,compress-force=zstd:1,commit=120'; ### my default options to mount BTRFS file-systems
	```
	mount -o compress-force=zstd:1 /dev/mapper/root /mnt
	mkdir -p /mnt/efi
	mount /dev/{DISK}1 /mnt/efi
	```
	
8. Create btrfs subvolumes
	```
	home
	opt
	srv
	home/$USER/.mozilla
	home/$USER/.ssh
	var/lib/sddm
	var/cache
	var/crash
	var/log
	var/tmp
	```

	Optional:
	```
	var/lib/libvirt/images
	var/spool
	var/www
	/home/$USER/.thunderbird
	/home/$USER/.config/google-chrome
	/home/$USER/.gnupg
	```

9. Generate fstab
	genfstab -U /mnt >> /mnt/etc/fstab

	also mount the above subvolumes using the same options as /

10. Update mirrors and pacstrap
	reflector --country India --age 24 -l 10 --protocol http,https --sort rate --save /etc/pacman.d/mirrorlist
	? automate reflector update on reboot? (systemctl enable reflector.timer) - parameters (/etc/xdg/reflector/reflector.conf) https://ostechnix.com/retrieve-latest-mirror-list-using-reflector-arch-linux/

    pacstrap -K /mnt base base-devel linux linux-firmware amd-ucode vim cryptsetup btrfs-progs dosfstools util-linux git unzip sbctl kitty networkmanager sudo 


11. Some Setup

	#Set timezone:
	ln -sf /usr/share/zoneinfo/Asia/Kolkata /etc/localtime
	#Update time
	hwclock --systohc

	#NTP server for accuracy
	timedatectl set-ntp true (systemd-timesyncd.service)

	#Keyboard layout
	vi /etc/vconsole.conf
	KEYMAP=us

	#Locale
	vi /etc/locale.gen
	uncomment your desired locale
	save and run locale-gen
	(prefer UTF-8)
	check current locale by running:
	locale
	Set locale by running:
	localectl set-locale LANG=en_US.UTF-8
	
	#Hostname
	vi /etc/hostname and type desired hostname

	#Root Password
	passwd


12. Create USER

13. Create UKI and update initramfs

14. Bootloader

Packages: ffmpeg, vaapi, gstreamerm mesa,


## Post-Install
1. Snapper
2. Hyprland, polkit
3. Audio
4. AppArmor
5. [fwupd](https://github.com/fwupd/fwupd)
6. [Secure Boot 1](https://0pointer.net/blog/unlocking-luks2-volumes-with-tpm2-fido2-pkcs11-security-hardware-on-systemd-248.html)
7. [tpm - clear old keys first](https://wiki.archlinux.org/title/Trusted_Platform_Module#systemd-cryptenroll)

## Ricing
1. Hyprland
2. Wofi
3. Waybar
4. Wlogout
5. Pywal
6. Alacritty
7. Thunar

