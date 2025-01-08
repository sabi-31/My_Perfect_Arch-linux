## Table of Contents

1. [Introduction and Credits](#1_introduction-and-credits)
1. [Pre-Install](#2_pre-install)
2. [Installation](#3_installation)
3. [Post-Install](#4_post-install)
4. [Ricing](#5_ricing)
5. [The Process Continues](#6_the-process-continues)

---

# 1. Introduction and Credits
> This was supposed to be a personal reference document, but I ended up formatting it as a guide since I think it will be helpful to many people out there who want the same type of install. I will walk through all the steps from beginning to end to installing what I consider (subjectively) to be a perfect arch linux install. 

Key features:

1. Btrfs install for system snapshot capabilities.
2. Fully encrypted Root partition.
3. Unified Kernel Images.
4. Secure Boot Support with self signed keys.
5. Fallback kernel and Windows booting with rEFInd.
6. Snapshot Booting with rEFInd btrfs.
7. My own learnings and best practises for a stable system.
8. Some ricing examples and dotfiles that I will keep updating.

? Why refind and systemd-boot both?

> A lot of the stuff I will talk about here, was hugely inspired by other people in the community and their work. This guide is not to diminish their efforts, but to build upon them. 

Special Credits to:

1. [This Extensive guide by Walian which was the main motivator.](https://walian.co.uk/arch-install-with-secure-boot-btrfs-tpm2-luks-encryption-unified-kernel-images.html)
2. [This guide based on fedora by Madhu@sysguides.](https://sysguides.com/install-fedora-with-snapshot-and-rollback-support)
3. [And offcourse, the Arch Wiki.](https://wiki.archlinux.org/title/Main_page)

> I will try and keep the sections as modular as possible, so if someone wants to, for example, skip setting up secure boot, they can still follow along. I will also be linking some articles and blogposts which I found extremely helpful to understand some of the concepts, which you should definitely read. I am also by no means an expert, so I welcome any and all feedback on this. Lets start:

---

# 2. Pre-Install
##### This is all the preparation you need to do before attempting the arch install as well as explaining some stuff.
1. Download the arch iso
	Either use the BitTorrent download (Needs a torrent client installed on your machine) or choose your nearest https mirror from here -> [https://archlinux.org/download/](https://archlinux.org/download).

2. Verify the Downloaded Image (Optional but recommended)

3. Burn the iso to a usb stick
	Use a tool like rufus or balena etcher if you are on windows. On linux, you can use KDE ISO Image Wirte, or the Gnome Disk Utility for a good gui application to do this. Many more methods documented on the wiki [here](https://wiki.archlinux.org/title/USB_flash_installation_medium).
	
	---
	<details>
	<summary>Ventoy</summary>
		Ventoy is a fantastic software, where you need to do the iso buring step only once, of Ventoy itself. After that, you just need to place any other iso files on the usb, and Ventoy will allow you to boot from it. 
		You can also have multiple images in the same usb. While I love using it, when I was doing a baremetal install on my PC, the iso from Ventoy would not boot. 
		I was getting errors as documented [here](https://github.com/ventoy/Ventoy/issues/2825). So I just burned the iso directly to another usb, and it worked. 
		This seems to have been since fixed, or might not even occour for you. It's upto you to choose how you want to do this.
	</details>
	
	---

4. Identify the Installation Target
	> If you're like me, and are installing linux on a separate drive, while already having windows on another drive, you need to check and make sure to correctly identify the drive. In linux, you can list all your drives using the command 'lsblk'. It labels SATA drives as sda1,sda2,... and nvme drives as nvme0n1, nvme1n1,... 
	
	> To avoid nuking your windows install, make a note of the correct drive. You can identify drives by their labels, existing partition, storage capacities, etc.

# 3. Installation
##### The Actual Install Process Begins

1. Boot the ISO
2. Internet Connectivity
	If you have a wired ethernet connection, just check for connectivity and proceed to step 3.
	If you're on wifi, run the below commands to connect to your network:
	```
	iwctl
	device list
	device wlan0 set-property Powered on
	station wlan0 scna
	station wlan0 get-networks
	station wlan0 connect <wifi-name>
	```

	Check connectivity using:
	> ping google.com

3. Check for disks
	>lsblk
	Find the name of your target disk, in my case it is nvme1n1
	Save this in a variable
	```
	DISK=/dev/nvme1n1

	echo $DISK
	```

4. Partition your disk
	In this step, we will create a new gpt table on the disk, and then create 2 partitions - a 512Mb EFI partiton and a root partition with the remaining space
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

5. Encryption
	Now we will encrypt the root partition with luks2
	> For btrfs, we first need to encrypt the disk, and then create the filesystem. In ext4, the partiton can be create before
   
	```
		cryptsetup luksFormat --type luks2 ${DISK}2
		cryptsetup open --type luks ${DISK}2 root
	```

	<details>
	<summary>Extras</summary>
		<pre>
		Crypttab - possibly not needed with systemd boot
		Backup LUKS Headers
		Generate a recovery key
		Register into tpm
		</pre>
	</details>

6. Format your Partitions
	```
		mkfs.vfat -F32 -n EFI ${DISK}1
		mkfs.btrfs -f -L root /dev/mapper/root
	```

	> blkid -o list

7. Mount Partitions
	>Compression level to use and algo
	strFAT32options='rw,relatime,fmask=0022,dmask=0022,codepage=437,iocharset=iso8859-1,shortname=mixed,utf8'; ### my default options to mount FAT-32 file-systems
	strBTRFSoptions='defaults,noatime,space_cache,ssd,compress-force=zstd:1,commit=120'; ### my default options to mount BTRFS file-systems
	
	```
	mount -o compress-force=zstd:1 /dev/mapper/root /mnt
	mkdir -p /mnt/efi
	mount /dev/{DISK}1 /mnt/efi
	```
	
8. Create btrfs subvolumes
	(Arch recommended - /, /home, /var/log, /var/cache/pacman/pkg) and use @ naming scheme
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

	> also mount the above subvolumes using the same options as /

*** - SWAP ON ZRAM https://wiki.archlinux.org/title/Swap

9. Generate fstab
	genfstab -U /mnt >> /mnt/etc/fstab
	> Why am I using fstab when discoverable partitions exist?



10. Update Mirrors and Pacstrap
	reflector --country India --age 24 -l 10 --protocol http,https --sort rate --save /etc/pacman.d/mirrorlist
	? automate reflector update on reboot? (systemctl enable reflector.timer) - parameters (/etc/xdg/reflector/reflector.conf) https://ostechnix.com/retrieve-latest-mirror-list-using-reflector-arch-linux/

	pacstrap -K /mnt base base-devel linux linux-firmware amd-ucode vim cryptsetup btrfs-progs dosfstools util-linux git unzip sbctl kitty networkmanager sudo openssh vim

11. Set locales
	#Set timezone:
	ln -sf /usr/share/zoneinfo/Asia/Kolkata /etc/localtime
	
	#Update time
	hwclock --systohc
	
	#Update time
	timedatectl

	#NTP server for accuracy
	timedatectl set-ntp true (systemd-timesyncd.service)

	

	#Locale
	```
	Locale  Language - en_US
	Locale layout - us
	locale encoding - UTF-8
	```	
	#Keyboard layout
	vi /etc/vconsole.conf
	KEYMAP=us

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

	
12. User Management
	#Root Password
	passwd

	### Create a new user for yourself

13. Unified Kernel Images

14. Enable services
	systemd-resolved
	systemd-timesyncd
	NetwrokManager

15. Install your bootloader

---

# 4. Post-Install
##### First Boot and setting up some defaults as well as basic checks on the system
1. Update Pacman and install essential packages:
	ffmpeg, vaapi, gstreamer mesa, pipewire, pipewire-pulse
	zsh, fsck, man-db, man-pages, findutils
	Arch recommended for hyprland desktop - dolphin, dunst, grim, kitty, polkit-kde-agent, qt5-wayland, qt6-wayland, slurf, wofi, xdg-desktop-portal-hyprland, htop, iwd, wget, wireless_tools wpa_supplicant, xdg-utils
	https://wiki.archlinux.org/title/Xorg#Driver_installation - AMD Drivers (Arch recommended) - libva-mesa-driver, mesa, vulkan-radeon, xf86-video-amdgpu, xf86-video-ati xorg-server, xorg-xinit

	Optional Repos - multilib, testing
	Flatpaks

2. Check Journal(journalctl -exb) and systemctl failed units (systemctl --failed)


1. Snapper and snapshots
	[wiki-preventing-slowdowns](https://wiki.archlinux.org/title/Snapper#Preventing_slowdowns)
	Disbale cow for other subvolumes - chattr +C /mnt/@/var
3. Encryption
	1. Backup
	2. TPM auto unlocking
		1. [tpm - clear old keys first](https://wiki.archlinux.org/title/Trusted_Platform_Module#systemd-cryptenroll)

4. Secure Boot
	SBCTL
	[Secure Boot 1](https://0pointer.net/blog/unlocking-luks2-volumes-with-tpm2-fido2-pkcs11-security-hardware-on-systemd-248.html)


5. Installing Hyprland
6. Other stuff:
	1. Audio
	2. AppArmor
	3. [fwupd](https://github.com/fwupd/fwupd)
	4. Cleanup alsa
 	5. Check systemctl services
  	6. Swapfs
   	7. [Maintainence](https://gist.github.com/MaxXor/ba1665f47d56c24018a943bb114640d7)
	8. CLear pacman cache - https://wiki.archlinux.org/title/Pacman#Cleaning_the_package_cache, https://wiki.archlinux.org/title/Pacman/Tips_and_tricks#Removing_unused_packages_(orphans)
	9. Enable https://wiki.archlinux.org/title/Pkgstats to help the community
	10. Read Messages during system upgrade
	11. Install lts kernel for fallback
	12. Firewall
---


# 5. Ricing
##### The stuff everyone actually cares about
1. Hyprland
 1. Workspace Overview
 2. Fractional Scaling
 3. Notification Daemon
 4. [l1](https://github.com/mylinuxforwork/dotfiles/tree/main/share) and [l2](https://www.youtube.com/watch?v=J1L1qi-5dr0)
2. Wofi
3. Waybar
4. Wlogout
5. Pywal
6. Alacritty
7. Plymouth
---

# 6. The Process Continues
##### Given the nature of FOSS software, there are always changes/improvements possible. I will document them here
