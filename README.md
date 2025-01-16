## Table of Contents

1. [Introduction and Credits](#1_introduction-and-credits)
1. [Pre-Install](#2_pre-install)
2. [Installation](#3_installation)
3. [Post-Install](#4_post-install)
4. [Ricing](#5_ricing)
5. [The Process Continues](#6_the-process-continues)

---

# 1. Introduction and Credits
> This was supposed to be a personal reference document, but I ended up formatting it as a guide since I think it will be helpful to many people out there who want the same type of install. I will walk through and explain the rationale behind all the steps from beginning to end, towards installing what is (subjectively) a perfect arch linux install. 

Key features:

1. BTRFS + Snapper rollbacks.
2. Fully encrypted Root partition.
3. Unified Kernel Images.
4. Secure Boot.
5. Dual Boot.
6. Snapshot Booting with rEFInd btrfs.
7. Hyprland + Apps.
8. System Maintainence and Best Practises.
9. Ricing and Dotfiles.

This guide assumes that you are on a UEFI system.

> A lot of the stuff I will talk about here, was hugely inspired by other people in the community and their work. 

Special Credits to:

1. [This Extensive guide by Walian which was the main motivator.](https://walian.co.uk/arch-install-with-secure-boot-btrfs-tpm2-luks-encryption-unified-kernel-images.html)
2. [This guide based on fedora by Madhu@sysguides.](https://sysguides.com/install-fedora-with-snapshot-and-rollback-support)
3. [And offcourse, the Arch Wiki.](https://wiki.archlinux.org/title/Main_page)

> I will try and keep the sections as modular as possible, so if someone wants to, for example, skip setting up secure boot, they can still follow along. I will also be linking some articles and blogposts which I found extremely helpful to understand some of the concepts, which you should definitely read. 

> I am by no means an expert, so I welcome any and all feedback on this. Lets arch.

---

# 2. Pre-Install
### This is all the preparation you need to do before attempting the install.
1. Download the arch iso
	Either use the BitTorrent download (needs a bit-torrent client installed on your machine) or choose your nearest **https** mirror from here -> [https://archlinux.org/download](https://archlinux.org/download). <br><br><br>

2. Verify the Downloaded Image (Optional but recommended)
	```
	gpg --keyserver-options auto-key-retrieve --verify archlinux-version-x86_64.iso.sig
	```

	where gpg is GnuPG, which needs to be installed on your existing system, and archlinux-version-x86_64.iso.sig is the file you download from the same webpage as the iso. <br><br><br>

3. Burn the iso to a usb stick
	Use a tool like rufus or balena etcher if you are on windows. On linux, you can use KDE ISO Image Wirte, or the Gnome Disk Utility for a good gui application to do this. Many more methods documented on the wiki [here](https://wiki.archlinux.org/title/USB_flash_installation_medium). <br><br>
	
	<details>
	<summary>Ventoy</summary>
		You can alternatively use a software like ventoy, which prevents the need to burn a iso image to a drive, while also allowing you to have multiple iso images on the same disk. I personally use and love it. You can check out more about here on it's github page -> https://github.com/ventoy/Ventoy
	</details>
	<br><br>

4. Identify the Installation Target
	> If you're like me, and are installing linux on a separate drive, while already having windows on another drive, you need to check and make sure to correctly identify the drive. In linux, you can list all your drives using the command 'lsblk'. It labels SATA drives as sda1,sda2,... and nvme drives as nvme0n1, nvme1n1, etc. To avoid nuking your windows install, make a note of the correct drive. You can identify drives by their labels, existing partition layouts, storage capacity, etc.

---

# 3. Installation
### The Actual Install Process

1. Boot from the ISO
	> Plug in your usb with the Arch ISO, reboot to your PC's Motherboard settings (lookup how to for your model, generally it's by pressing the F2 or del key during boot), and in the boot priority, set the usb as the first to boot from. If secure boot is enabled,you have to disable it. 
	
	>Further in this guide, I will show you how to generate custom secure boot keys. We will sign our arch install, and our arch iso with those keys. We can then have secure boot enabled.

	<br><br>

2. Internet Connectivity
	If you have a wired ethernet connection, just check for connectivity and proceed to step 3.<br>
	If you're on wifi, run the below commands to connect to your network:
	```
	iwctl
	device list
	device wlan0 set-property Powered on
	station wlan0 scan
	station wlan0 get-networks
	station wlan0 connect <wifi-name>
	<Enter password>
	exit
	```

	> Here wlan0 is the name of my wifi device as output by device list, and \<wifi-name> will be substituted by the name of my Wifi Connection.
	


	Check connectivity using:
	```
	ping google.com
	```
	<br><br>

3. Check for disks
	```
	lsblk
	```
	Find the name of your target disk, in my case it is nvme1n1. <br>
	Save this in a variable as to avoid any mistakes during partitioning, especially in a dual boot environment. <br>
	If you don't have a nvme SSD, your disk may be listed as sda,sdb, etc. <br>

	```
	DISK=/dev/nvme1n1
	echo $DISK
	```
	<br><br>
4. Partition your disk
	In this step, we will create a new gpt table on the disk, and then create 2 partitions - a 1GB EFI partiton (also referred to as the ESP) and a Root partition with the remaining space.
	<br>
	
	> I am seriously debating the size of the EFI partition. While 512Mb has been more than enough for me is the past, I have seen people recommend 1 GB. I have a 1tb SSD on which I will be installing arch, so it's not a big deal for me, and as I plan to use this system long term, I don't want to have to deal with rezising the partition later, so I am making it 1GB. If you want to have it as 512Mb, use +512M instead of +1G in the command below:

	````
	a. fdisk $DISK
	b. g
	c. n
	d. default
	e. default
	f. +1G
	g. n
	h. default
	i. default
	j. default
	k. t
	l. 1
	m. 1
	n. t
	o. 2
	p. 23
	q. w
	````

	For the first partition, we set the parition type as EFI, and for the second one it was set as Linux Root (x86_64) instead of the default (Linux Filesystem).<br>
	The second step, while technically not necessary, is essential, as this will associate a standard partition GUID with it.<br>
	This is then used by systemd (only if using systemd-boot) to recoginze our root partition to decrypt and mount it automatically without a crypttab or fstab file.
	<br>
	Read more about [Discoverable Partitions](https://www.freedesktop.org/software/systemd/man/latest/systemd-gpt-auto-generator.html). 
	
	Verify your partition type GUID (separate from a normal partition GUID output with blkid) with:

	```
	lsblk -p -o NAME,PARTTYPE
	```

	Root Filesystem (dev/sda2) should have a guid of 4f68bce3-e8cd-4db1-96e7-fbcaf984b709.
	The EFI should have the guid of c12a7328-f81f-11d2-ba4b-00a0c93ec93b


	Run lsblk to verify partitions

	If your disk name was sda,sdb, you can skip this step. If it was a nvme drive like me, update the DISK variable as shown below. This is because partitons in nvme are named as nvme1n1p1, nvme1n1p2 while other are named as sda1,sda2,
	```
		DISK=/dev/nvme1n1p
	```

5. Encryption
	Now we will encrypt the root partition with luks2
   
	```
		cryptsetup luksFormat --type luks2 ${DISK}2
		cryptsetup open --type luks --allow-discards --perf-no_read_workqueue --perf-no_write_workqueue --persistent ${DISK}2 root
	```
	The 'allow discards' enables trim support for a SSD (https://wiki.archlinux.org/title/Dm-crypt/Specialties#Discard/TRIM_support_for_solid_state_drives_(SSD) ) 
	(https://wiki.archlinux.org/title/Solid_state_drive#TRIM)
	the persistent options writes these options to the header, so they are always used in the future
	the --perf-no_read_workqueue and --perf-no_write_workqueue increases performance for SSD's (https://wiki.archlinux.org/title/Dm-crypt/Specialties#Disable_workqueue_for_increased_solid_state_drive_(SSD)_performance)
	You can check the flags with:
	cryptsetup luksDump /dev/sdaX | grep Flags

	>From here on, our root isn't /dev/sda2 or /dev/nvme1n2, but rather /dev/mapper/root
	<details>
	<summary>Crypttab</summary>	
		Crypttab - possibly not needed with systemd boot
		https://wiki.archlinux.org/title/Dm-crypt/System_configuration#Using_systemd-cryptsetup-generator
	</details>

6. Format your Partitions
	```
		mkfs.vfat -F32 -n ESP ${DISK}1
		mkfs.btrfs -L root /dev/mapper/root
	```
	
	Check if the partitons are correctly created:
	```
	blkid -o list
	```

7. Btrfs
	1. Subvolumes
	
	> This section is a lot more subjective to the type of installation you prefer. Basically, since we are on a btrfs filesystem, and will be using the rollback functionality on the root subvolume, you can choose which folders on your root won't be rolled back. They can have their own rollback logic created. To do this, they need to be mount as separate subvolumes. Choosing which folders should be subvolumes has no definitive answer.
	
	
	The archinstall script uses - /, /home, /var/log, /var/cache/pacman/pkg and use @ naming scheme.

	[OpenSuse Recommends](https://en.opensuse.org/SDB:BTRFS) - /home, /opt, /root, /srv, /tmp, /usr/local, /var.

	This is the layout I personally use, and which has worked well for me. It's very well expalined in this [sysguides article](https://sysguides.com/install-fedora-41-with-full-disk-encryption-snapshot-and-rollback-support#1-partitions-and-subvolumes-layout).

	```

	@home -> Home Folder where your main configs/data/games will reside
	@mozilla -> The directory where your firefox data is stored. If you ever rollback your home directory, this will prevent any potential loss of browsing data
	@ssh -> Same as above, to protect any ssh keys/configs you have
	@opt -> Third party applications
	@var
 	@snapshots
 	@home-snapshots
	```

	btrfs subvolume create {@,@home,@mozilla,@ssh,@var,@opt,@snapshots,@home-snapshots}

	get subvol id of @
	btrfs subvolume set-default $ID /mnt

	I am making the whole of var into one subvolume. This is not recommended by arch, as pacman stores it's cache in the /var/cache/pacman directory. But I am going to configure pacman cache to be in the tmp directory, as I don't need it, you can read the rationale behind this [here](https://www.reddit.com/r/archlinux/comments/1hgbl1k/what_is_varcachepackagepkg_and_why_is_it_so_large/)

	Optional, if you use Thunderbird, Chrome, or Gnupg.
	```
	@thunderbird - /home/$USER/.thunderbird
	@chrome - /home/$USER/.config/google-chrome
	@gnupg - /home/$USER/.gnupg
	```

	> You can also create btrfs subvolumes in the future if you need it. 
	
	2. Mount Options

	Mount all the subvolumes you created using the same mount options as the root.
	mount -o defaults,noatime,space_cache=v2,ssd,compress-force=zstd:1,commit=120,subvol=@home /dev/mapper/root /mnt/home

	mount -o defaults,noatime,space_cache=v2,ssd,compress-force=zstd:1,commit=120,subvol=@ /dev/mapper/root /mnt
	mount -o subvol=@home /dev/mapper/root /mnt/home
	mount -o subvol=@mozilla /dev/mapper/root /mnt/home/$USER/.mozilla
	mount -o subvol=@ssh /dev/mapper/root /mnt/home/$USER/.ssh
	mount -o subvol=@opt /dev/mapper/root /mnt/opt
	mount -o subvol=@var /dev/mapper/root /mnt/var

	> Once a subvolume is mounted with some options, all other subvoumes follow the same options. This is fine mostly as I need the same options everywhere, except in the @var directory, where I want to have the nodatacow mount option. So instead I will set the attribute +C on the var directory, which accomplishes the same thing.
 	```
  	chattr +C /var
  	```
  
	Note: nodatacow and compress/compress-force can't be used together.

	Mount Options:
	[All](https://btrfs.readthedocs.io/en/stable/Administration.html#mount-options)

	Also mount the efi partition
	mkdir -p /mnt/efi
	mount -o srw,relatime,fmask=0077,dmask=0077,codepage=437,iocharset=iso8859-1,shortname=mixed,utf8 /dev/{DISK}1 /mnt/efi
	OR
	mount -o defaults,fmask=0077,dmask=0077 /dev/{DISK}1 /mnt/efi

8. Generate fstab

	```
	mkdir /mnt/etc
	touch /mnt/etc/fstab
	genfstab -U /mnt >> /mnt/etc/fstab
	```

	<details>
		<summary>Why am I using fstab when discoverable partitions exist?</summary>
		[Discoverable partitons](https://www.freedesktop.org/wiki/Specifications/DiscoverablePartitionsSpec/) is a cool feature which can automount your drives, and can fully be used in this install, as I will be using systemd-boot as my bootloader.
		However, I will still manually define my mount points in fstab. This is because:
			a. I don't see an option of defining my mount options, and I don't want them to be the system defaults
			b. I will still have to mount my btrfs subvolumes manually, so why not do it all myself?
			c. I don't like the abstraction, I prefer to setup my system myself.
	</details>
	


10. Update Mirrors and Pacstrap

	Before we download and install the necessary packages, let us update our mirrors, for the best speed, by running:

	```
	reflector --country India --age 24 -l 10 --protocol http,https --sort rate --save /etc/pacman.d/mirrorlist
	```

	<details>
		<summary>Automate Reflector in a Running System</summary>
		Mirrors that you use may get outdated or put out of commission. You can set the reflector service to run on boot by enabling the systemd service (systemctl enable reflector.timer).
		[Guide](https://ostechnix.com/retrieve-latest-mirror-list-using-reflector-arch-linux/)
	</details>

	Now, I will install the base required packages for my arch install using the pacstrap command on my /mnt where the system root is mounted. 
	
	```
	pacman -Sy archlinux-keyring
	pacstrap -K /mnt base base-devel linux linux-firmware amd-ucode cryptsetup btrfs-progs dosfstools util-linux git unzip sbctl kitty networkmanager sudo openssh vim
	```

11. Chroot into the install and do basic setup

	```
	arch-chroot /mnt
	```


	#Set timezone:
	ln -sf /usr/share/zoneinfo/Asia/Kolkata /etc/localtime
	
	#Update time
	hwclock --systohc

	#NTP server for accuracy
	timedatectl set-ntp true (systemd-timesyncd.service)
	systemctl enable systemd-timesyncd

	
	#Locale

	vi /etc/locale.gen

	uncomment your desired locale
	save and run 
	locale-gen

	check current locale by running:
	locale

	 systemd-firstboot --prompt

	#Pacman Cache
	vim /etc/pacman.conf
	CacheDir     = /tmp/cache-pacman/pkg/

	
12. User Management
	#Root Password
	passwd

	### Create a new user for yourself
	useradd -G wheel -m $USER
	passwd $USER

	visudo
	uncomment by removing the '#' of the following line:
	# %wheel ALL=(ALL:ALL) ALL

	#Change ownership of user's home
	chown -R $USER:$USER /home/$USER


13. Unified Kernel Images
	```
	echo "quiet rw" >/etc/kernel/cmdline

	mkdir -p /efi/EFI/Linux

	vim /etc/mkinitcpio.conf"
	HOOKS=(base systemd autodetect modconf kms keyboard sd-vconsole sd-encrypt block filesystems fsck)

	vim /etc/mkinitcpio.d/linux.preset
	Uncomment lines for uki, and comment for image

	mkinitcpio -P
	```

14. Enable services

	Networking

	```
	systemctl enable systemd-resolved NetworkManager
	systemctl mask systemd-networkd
	```

15. Install your bootloader
	```
	bootctl install --esp-path=/efi
	sync
	exit 

	systemctl reboot --firmware-setup

	```


---

# 4. Post-Install
##### First Boot and setting up some defaults as well as basic checks on the system
1. Update Pacman and install essential packages:
	ffmpeg, *vaapi*, gstreamer mesa, pipewire, pipewire-pulse
	zsh, *fsck*, man-db, man-pages, findutils
	Arch recommended for hyprland desktop - dolphin, dunst, grim, kitty, polkit-kde-agent, qt5-wayland, qt6-wayland, slurp, wofi, xdg-desktop-portal-hyprland, htop, iwd, wget, wireless_tools wpa_supplicant, xdg-utils
	https://wiki.archlinux.org/title/Xorg#Driver_installation - AMD Drivers (Arch recommended) - libva-mesa-driver, mesa, vulkan-radeon, xf86-video-amdgpu, xf86-video-ati xorg-server, xorg-xinit, mesa-vdpau

	Optional Repos - multilib, testing
	Flatpaks

2. Check Journal(journalctl -exb) and systemctl failed units (systemctl --failed)


1. Snapper and snapshots
	Btrfs-Assistant
	Auto Snapshots and Cleanup
	Disable indexing of snapshots
	[wiki-preventing-slowdowns](https://wiki.archlinux.org/title/Snapper#Preventing_slowdowns)
	Disbale cow for other subvolumes - chattr +C /mnt/@/var


3. Encryption
	1. Backup
	2. TPM auto unlocking
		1. [tpm - clear old keys first](https://wiki.archlinux.org/title/Trusted_Platform_Module#systemd-cryptenroll)

4. Install rEFInd
	<details>
		<summary>Why do I need both rEFInd and systemd-boot? </summary>
		- rEFInd Btrfs
		- Customizability
		- Separation of both bootloaders
		- Management is easier of systemd-boot
	</details>
For Extras

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
	13. Enable reflector on boot

7. System Maintainence
	1. Btrfs filesystem
		1. [Defragmentation](https://wiki.archlinux.org/title/Btrfs#Defragmentation)
		2. 

	2. Recovering from issues with the help of Snapshots
		1. [Snapshot Rollbacks]
		1. [Snapshot Booting](https://wiki.archlinux.org/title/Btrfs#Booting_into_snapshots)
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
