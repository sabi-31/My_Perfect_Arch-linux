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

> While I will try to explain why and what I'm doing here, this is not a noob friendly guide. You need to know at least basic concepts of linux and know your way around the terminal.

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
	This setting of partition type, while technically not necessary, is essential, as this will associate a standard partition GUID with it.<br>
	This is then used by systemd (only if using systemd-boot) to recoginze our root partition to decrypt and mount it automatically without a crypttab or fstab file.
	<br>
	Read more about [Discoverable Partitions](https://www.freedesktop.org/software/systemd/man/latest/systemd-gpt-auto-generator.html). 
	
	Verify your partition type GUID (separate from a normal partition GUID, output with blkid) with:

	```
	lsblk -p -o NAME,PARTTYPE
	```

	Root Filesystem (/dev/sdx2 or /dev/nvmenxn1p2) should have a guid of 4f68bce3-e8cd-4db1-96e7-fbcaf984b709. <br>
	The EFI parition should have the guid of c12a7328-f81f-11d2-ba4b-00a0c93ec93b

	<br>
	Run lsblk to verify your partition sizes.
	<br><br>
	If your disk name was sda,sdb, you can skip this step. If it was a nvme drive like me, update the DISK variable as shown below. This is because partitons in nvme are named as nvme1n1p1, nvme1n1p2 while other are named as sda1,sda2,
	
	```
		DISK=/dev/nvme1n1p
	```
	<br>
	<br>
5. Encryption
	Now we will encrypt the root partition with luks2.
	> When we use luks to encrypt our drive with a password or a key, what it does is create a header. This header is what actually encrypts and decrypts the drive. Our key actually encrypts only the header. This is different that plain encryption, where our password is used to encrypt the entire disk. This allows a greater flexibility when creating and managing passwords and keys. We can also have multiple passwords and keys created that unlock the header.

	> One downside of this approach is that if our header ever gets corrupted, we lose the ability to unlock our entire drive. To mitigate this, we will also backup our drive header further in the guide.
	
	<br>

	```
		cryptsetup -v luksFormat --type luks2 ${DISK}2
		cryptsetup open --type luks --allow-discards --perf-no_read_workqueue --perf-no_write_workqueue --persistent ${DISK}2 root
	```
	<br>

	In the first command we formatted our disk with luks2, it will autogenerate the header and ask you to provide a password to unlock. Make this password strong, or better yet use a [passphrase](https://www.techtarget.com/searchsecurity/definition/passphrase)
	
	In the second command, we open our encrypted drive, and give it the name of 'root'. From here on, our root partition isn't /dev/sda2 or /dev/nvme1n1p2, but rather it's /dev/mapper/root
	
	<br>
	The 'allow discards' enables [Trim](https://wiki.archlinux.org/title/Solid_state_drive#TRIM) support for a SSD [Read More](https://wiki.archlinux.org/title/Dm-crypt/Specialties#Discard/TRIM_support_for_solid_state_drives_(SSD)).
	
	The --perf-no_read_workqueue and --perf-no_write_workqueue increases performance for SSD's [Read More](https://wiki.archlinux.org/title/Dm-crypt/Specialties#Disable_workqueue_for_increased_solid_state_drive_(SSD)_performance)

	The 'persistent' option writes these options to the header, so they are always used by default in the future.

	<br>
	You can check the flags that your drive is opened with using:
	
	<br>

	```
	cryptsetup luksDump /dev/sdaX | grep Flags
	```
	> We will be relying on systemd to automatically recognize and decrypt our root partition. If you decide to use grub, you have to so some more setup.
	
	<br>
	<br>

6. Format your Partitions
	Now we will format our efi partition as Fat32 and the root partition as btrfs.
	```
		mkfs.vfat -F32 -n ESP ${DISK}1
		mkfs.btrfs -L root /dev/mapper/root
	```
	
	Check if the partitons are correctly created:
	```
	blkid -o list
	```
		
	<br>
	<br>

7. Btrfs Subvolumes
    > This section is a lot more subjective to the type of installation you prefer. Basically, since we are on a btrfs filesystem, and will be using the rollback functionality on the root subvolume, you can choose which folders on your root won't be rolled back. They can have their own rollback logic created. To do this, they need to be mount as separate subvolumes. Choosing which folders should be subvolumes has no definitive answer.
	
	
	The archinstall script creates the following subvolumes - /, /home, /var/log, /var/cache/pacman/pkg and the '@' naming scheme.

	OpenSuse Recommends the following [Subvolumes]](https://en.opensuse.org/SDB:BTRFS) - /home, /opt, /root, /srv, /tmp, /usr/local, /var.

	<br>

	This is the subvolume layout and I personally use, and which has worked well for me.

	> @ (/) -> The root folder, which is also a separate subvolume below the btrfsroot volume, and which we will rollback in case of any issues.<br><br>
	@home (/home) -> Home Folder where your Data/Games/Configurations will reside. <br><br>
	@mozilla (/home/\$USER/.mozilla) -> The directory where your firefox data is stored. If you ever rollback your home directory, this will prevent any potential loss of browsing data. <br><br>
	@ssh (/home/\$USER/.ssh) -> Same as above, to protect any ssh keys/configs you have. <br><br>
	@opt (/opt)  -> This is where third party applications are installed. <br><br>
	@var (/var) -> Where all temporary files/logs/cache is stored.  <br><br>
 	@snapshots (/.snapshots) -> This subvolume will house snapshots of our @ subvolume (Managed via Snapper). <br><br>
 	@home-snapshots (/home/.snapshots) -> This subvolume will house snapshots of our @home subvolume (Managed via Snapper). <br><br>
	
	<br>

	```
	mount /dev/mapper/root /mnt
	cd /mnt
	btrfs subvolume create {@,@home,@mozilla,@ssh,@var,@opt,@snapshots,@home-snapshots}
	btrfs subvolume list /mnt
	```

	The final commands lists all the subvolumes with their ID.
	Get subvol id of @ and set it as default for the root (eg. 256)

	```
	btrfs subvolume set-default $ID /mnt
	```

	>I am making the whole of var into one subvolume. This is not recommended by arch, as pacman stores it's cache in the /var/cache/pacman directory. But I am going to configure pacman cache to be in the tmp directory, as I don't need it, you can read the rationale behind this [here](https://www.reddit.com/r/archlinux/comments/1hgbl1k/what_is_varcachepackagepkg_and_why_is_it_so_large/)

	Optional, if you use Thunderbird, Chrome, or Gnupg you can also create the below subvolumes to preserve their data similar to @mozilla above.
	```
	@thunderbird - /home/$USER/.thunderbird
	@chrome - /home/$USER/.config/google-chrome
	@gnupg - /home/$USER/.gnupg
	```

	> More btrfs subvolumes can also be created in the future if you need it. 
		
	<br>
	<br>

8. Mount Options

	Mount all the subvolumes you created using the same mount options as the root.
	mount -o defaults,noatime,space_cache=v2,ssd,compress-force=zstd:1,commit=120,subvol=@home /dev/mapper/root /mnt/home

	mount -o defaults,noatime,space_cache=v2,ssd,compress-force=zstd:1,commit=120,subvol=@ /dev/mapper/root /mnt
	mount -o subvol=@home /dev/mapper/root /mnt/home

	mount -o subvol=@opt /dev/mapper/root /mnt/opt
	mount -o subvol=@var /dev/mapper/root /mnt/var

	> We will mount @mozilla, @ssh, @snapshots and @home-snapshots,  later on.

	> Once a subvolume is mounted with some options, all other subvoumes follow the same options. This is fine mostly as I need the same options everywhere, except in the @var directory, where I want to have the nodatacow mount option. So instead I will set the attribute +C on the var directory, which accomplishes the same thing.
 	```
  	chattr +C /var
  	```
  
	>Nodatacow and compress/compress-force can't be used together.

	Read more about various btrfs [Mount Options](https://btrfs.readthedocs.io/en/stable/Administration.html#mount-options)

	Also mount the efi partition
	```
	mkdir -p /mnt/efi
	mount -o defaults,fmask=0077,dmask=0077 /dev/{DISK}1 /mnt/efi
	```




10. Update Mirrors and Pacstrap

	Before we download and install the necessary packages, let us update our mirrors, for the best download speed, by running:

	```
	reflector --country $COUNTRY --age 24 -l 10 --protocol https --sort rate --save /etc/pacman.d/mirrorlist
	```

	Now, we install the base required packages for a arch install using the pacstrap command on /mnt where the system root is mounted. 
	
	```
	pacman -Sy archlinux-keyring
	pacstrap -K /mnt base base-devel linux linux-firmware amd-ucode cryptsetup btrfs-progs dosfstools posix util-linux git networkmanager sudo openssh vim xorg-xwayland
	```

11. Chroot into the install and do basic setup

	>Chroot makes it so that the system behaves as if you are the root user of the new install. This is done so as to not need to reference /mnt as the root.

	```
	arch-chroot /mnt
	```

	1. Set the system timezone, Update the system time and Setup a NTP server for Clock Accuracy using the below commands (replace /Asia/Kolkata with your respective timezone)
		
		```
		ln -sf /usr/share/zoneinfo/Asia/Kolkata /etc/localtime
		timedatectl set-ntp true
		systemctl enable systemd-timesyncd
		```

		[Read this if you are dual-booting windows](https://wiki.archlinux.org/title/System_time#UTC_in_Microsoft_Windows)

	2. Locale
		#Locale

	vi /etc/locale.gen

		uncomment your desired locale by removing the # at the start.if it's not en_US.UTF-8, also uncomment it.([Why?](https://wiki.archlinux.org/title/Steam#Installation))
		Save the file and Run
		```
		locale-gen
		```
		Then run the following command and input desired locale and hostname:

		```
		systemd-firstboot --prompt
		```

	3. Pacman
		Improve pacman defaults:
		
		```
		vim /etc/pacman.conf
		```
		Under misc option, uncomment the following lines:
		Color
		ParallelDownloads = 5

		And add the line:
		ILoveCandy


		Change the pacman cache directory folder location

		```
		vim /etc/pacman.conf
		```

		Make the line starting with #CacheDir look like this and save:

		CacheDir     = /tmp/cache-pacman/pkg/

	
12. User Management
	1. Set Root Password
		```
		passwd
		```

	2. Create your user and set password.

		```
		useradd -G wheel -m $USER
		passwd $USER
		```

	3. Allow your user to run sudo commands with a password
	
		```
		visudo
		```

		Uncomment by removing the '#' of the following line and save:
		# %wheel ALL=(ALL:ALL) ALL

	4. Mount 2 of the remaining 4 subvolumes
		```
			mkdir /home/$USER/{.mozilla,.ssh}
			mount -o subvol=@mozilla /dev/mapper/root /mnt/home/$USER/.mozilla
			mount -o subvol=@ssh /dev/mapper/root /mnt/home/$USER/.ssh
		```

13. Generate fstab
	>fstab is a file referenced by the system during boot to mount your drives/partitions to the correct location. Since we have already mounted our drives, we will use the **genfstab** utility to output the fstab file with the options we chose earlier, and save it, so that in the future these options are used by default.

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
	

13. Generate Unified Kernel Images

	```
	echo "quiet rw" >/etc/kernel/cmdline
	mkdir -p /efi/EFI/Linux
	vim /etc/mkinitcpio.conf
	```
	
	Edit the hooks like in mkinitcpio.conf to look like this:
	HOOKS=(base systemd autodetect modconf kms keyboard sd-vconsole sd-encrypt block filesystems fsck)

	```
	vim /etc/mkinitcpio.d/linux.preset
	```

	Uncomment lines for uki, and comment for image

	Run below command to generate images
	```
	mkinitcpio -P
	```

14. Enable services

	Network

	```
	systemctl enable systemd-resolved NetworkManager
	systemctl mask systemd-networkd
	[Reflector](https://ostechnix.com/retrieve-latest-mirror-list-using-reflector-arch-linux/)
	sshd
	```

15. Bootloader Installation

	```
	bootctl install --esp-path=/efi
	sync
	exit 
	systemctl reboot --firmware-setup
	```

	Now in the Firmware setup menu, select the entry for Arch/The Disk as the Main Entry.
	You will be asked to enter the disk encryption password, post which you can login and use the installation.
	COngrats, you have a minimal arch linux system installed.


---

# 4. Post-Install
### While we have a working system now, we are missing a lot of utilities which make it actually usuable. In this section, I will walk through installing and configuring them.

1. Update Pacman and install essential packages:
	1. Drivers and Codecs
		
		Generic:
		```
		sudo pacman -S ffmpeg gstreamer mesa 
		```

		Amd GPU Specific:
		```
		sudo pacman -S vulkan-radeon lib32-mesa mesa-vdpau libva-mesa-driver
		paru -S amdgpu_top
		```

		>The amdgpu-top package is a tui+gui application to monitor your gpu usage. It needs paru/yay installed, which I do below.

		Nvidia GPU:
		>I don't have an Nvidia GPU, so you have to test this part yourself, refer this [link](https://github.com/korvahannu/arch-nvidia-drivers-installation-guide).
	
	2. AUR Helper
         	>I will install [paru](https://github.com/Morganamilo/paru) to help me download and install AUR packages.
			
			```
			sudo pacman -S --needed base-devel
			git clone https://aur.archlinux.org/paru.git
			cd paru
			makepkg -si
			```
	
	2. Shell and Fonts

		```
		sudo pacman -S zsh ttf-jetbrains-mono-nerd
		chsh /usr/bin/zsh
		```

	3. Audio
		
		```
		sudo pacman -S pipewire pipewire-pulse pavucontrol
		```

		>All of these won't be really relevant until we install Hyprland.

	4. Utilities
			
		```
		sudo pacman -S unzip man-db man-pages wget htop 
		```


	5. Flatpak
	
		```
		sudo pacman -S flatpak
		```

		You can search for applications you need on [Flathub](https://flathub.org/).

	6. [Optional Repositories](https://wiki.archlinux.org/title/Official_repositories#)
        I will be enabling the Multilib repo(because I need Steam).
		
		```
		vim /etc/pacman.conf
		```
		
		Uncomment the below lines by removing the '#'
		[multilib]
		Include = /etc/pacman.d/mirrorlist



2. Install Hyprland and other helpful gui applications
	
	```
	sudo pacman -S hyprland dolphin kitty wofi waybar hyprpolkitagent grim kitty qt5-wayland qt6-wayland slurp xdg-desktop-portal-hyprland xdg-utils vlc
	paru -S wlogout
	```

	Set this line in your hyprland config:
	exec-once = systemctl --user start hyprpolkitagent

	https://wiki.archlinux.org/title/XDG_MIME_Applications


4. Install rEFInd

	```
	sudo pacman -S refind
	refind-install
	```

	<details>
		<summary>Why do I need both rEFInd and systemd-boot? </summary>
		- rEFInd Btrfs
		- Customizability
		- Separation of both bootloaders
		- Management is easier of systemd-boot
	</details>

	### Refind BTRFS



6. Setup Secure Boot
	>We will generate our own secure boot keys, enroll them in our Motherboard firmware, and sign our kernel modules and other efi binaries with them. There's an amazing tool known as [sbctl](https://github.com/Foxboron/sbctl) which does most of the heavy lifting for us.

	```
	sudo pacman -S sbctl
	```

	Once sbctl is downloaded, you need to make sure your motherboard is in setup mode, so that you can enroll your keys into it. This is done typically by clearing out the existing secure boot keys in the motherboard settings. Look up how to do this for your brand of motherboard.

	>Many motherboards have the ability to restore the keys that you removed (I can confirm this for Asrock). Also if you ever update the firmware of the motherboard, you might have to enroll your own keys again.

	Check if motherboard is in Setup mode:

	```
	sbctl status
	```

	Post confirmation, we will generate our keys, enroll them into the motherboard(along with Microsoft keys for windows comapatibility).
	These keys are stored in the '/var/lib/sbctl/keys' directory. It's not a bad idea to back them up.

	```
	sudo sbctl create-keys
	sudo sbctl enroll-keys -m
	sudo sbctl status
	```

	Now we will sign our kernel, bootloader and boot manager(refind) files with our keys. We will also instruct sbctl to sign any future kernels images automatically.
	The 'sbctl verify' command lists all files that need to be signed and checks whether they are signed or not. We will then use 'sbctl sign -s $filename' to sign now and in the future with any updates.

	```
	sudo sbctl verify
	sudo sbctl sign -s /efi/EFI/Linux/arch-linux.efi
	sudo sbctl sign -s /efi/EFI/Linux/arch-linux-fallback.efi
	sudo sbctl sign -s /efi/EFI/BOOT/BOOTX64.EFI
	sudo sbctl sign -s /efi/EFI/systemd/systemd-bootx64.efi
	sudo sbctl sign -s /efi/EFI/refind/drivers_x64/btrfs_x64.efi
	sudo sbctl sign -s /efi/EFI/refind/refind_x64.efi
	```

	Now we can re-enable secure boot, and it should work fine.

	[Secure Boot 1](https://0pointer.net/blog/unlocking-luks2-volumes-with-tpm2-fido2-pkcs11-security-hardware-on-systemd-248.html)



7. Improve Encryption Experience [Read More](https://0pointer.net/blog/unlocking-luks2-volumes-with-tpm2-fido2-pkcs11-security-hardware-on-systemd-248.html)
    1. Backup LUKS Headers
		I have already explained previously why you might want to backup you luks header, which stores data about and controls your drive decryption.
		To backup the header, run:

		```
		sudo cryptsetup luksHeaderBackup /dev/gpt-auto-root-luks --header-backup-file luks-header-backup.img
		```

		This will generate a file 'luks-header-backup.img' in your current directory. Save this in another drive, in the cloud (depending on your threat model), or into a password manager like bitwarden. If your header ever gets corrupted, you can boot from another iso, and restore it using:

		```
		cryptsetup luksHeaderRestore /dev/device --header-backup-file /path-to/luks-header-backup.img
		```

    2. Enroll and backup an extra key
		While we normally use a passord to unlock the luks header, it can support multiple unlocking options, one of which is a key, auto-generate by the system with a key

    3. Setup Auto Unlocking via a Yubikey
	
    4. Setup auto unlocking via TPM
		Check if you have a tpm by running:

		```
		ls /sys/class/tpm
		```

		If you see an output like tpm0, them a tpm device exists
		
		Choose which PCR's to use. A pcr is a register on the TPM that can measure specific values, like is secure boot on or off, what is the firmware version,etc. You can see the full list [Here](https://wiki.archlinux.org/title/Trusted_Platform_Module#Accessing_PCR_registers)
		I will use PCR 0,7

		Enroll the TPM to the luks device with selected PCR's
		
		```
		sudo systemd-cryptenroll --tpm2-device=auto --tpm2-pcrs=0+7  /dev/gpt-auto-root-luks
		```






8. Swap and Hibernation
	For swap, I will use zram instaled of a swap partition
	[https://wiki.archlinux.org/title/Zram]https://wiki.archlinux.org/title/Zram


	Hibernation [Read More](https://wiki.archlinux.org/title/Power_management/Suspend_and_hibernate#)
	Without going into too many details, suspend capabilites are built into the kernel and should be available to use directly. If you want to use hibernate, you need a swap partition and some setup which is explained in the link above. 
	For my AMD system, with a B650 motherboard, I can see that the hardware supports S2Idle (aka saving data on RAM and powering all other components off). I am fine with this method.

	I did have troubles waking my system back up after sending it to suspend. On reading the wiki, I can across a [solution](https://wiki.archlinux.org/title/Power_management/Suspend_and_hibernate#PC_will_not_wake_from_sleep_on_A520I_and_B550I_motherboards), which worked perfectly. 
	Firstly, run the below command and check if GPP0 is enabled from the output

	```
	cat /proc/acpi/wakeup
	```

	If it is, run the below commands to disbale it, and test suspend is working:

	```
	su
	echo GPP0 > /proc/acpi/wakeup
	systemctl suspend
	```

	If this works fine, make this change permanent by creatin the following file:

	```
	vim /etc/systemd/system/toggle-gpp0-to-fix-wakeup.service
	```

	Add the following lines to it:
	[Unit]
	Description="Disable GPP0 to fix suspend issue"

	[Service]
	ExecStart=/bin/sh -c "/bin/echo GPP0 > /proc/acpi/wakeup"

	[Install]
	WantedBy=multi-user.target

9. Security
	1. Firewall
	2. 

10.  System Maintainence
	1. [Maintainence](https://gist.github.com/MaxXor/ba1665f47d56c24018a943bb114640d7)


	2. Btrfs filesystem
		1. [Defragmentation](https://wiki.archlinux.org/title/Btrfs#Defragmentation)
		2. Scrub
		3. Balance


	3. Snapshot Rollbacks
		1. [Snapshot Rollbacks]
		2. [Snapshot Booting](https://wiki.archlinux.org/title/Btrfs#Booting_into_snapshots)


11. Snapper and snapshots
	> Alright, all the efforts we put into btrfs and subvolumes, will help us now.

	By now, you must be aware of the btrfs snapshot feature. There is a tool developed by OpenSuse know as [snapper](https://github.com/openSUSE/snapper) which is a helper application for this purpose. Using snapper, we will create a config for our root subvolume (@) containing most of our system data, and another for the home subvolume(@home), which contains most of our user data. Then we can setup snapper to create and manager snapshots of them, to safeguard and if necessary, rollback this data (Remember, the other subvolumes we created are exempt from this). Snapper can also do automatic snapshots on a schedule, and there's a pacman hook known as snap-pac that can create snapshots whenever we use pacman to install/upgrade our system.

	```
	sudo pacman -S snapper snap-pac
	```

	Run the below commands in order to:
		1. Create Snapper Configurations
		2. Enable your user to work with them without sudo privileges
		3. Setup Automatic Creation and Cleanup of snapshots
		4. Disable Indexing of snapshtos to improve performance.

		```
		sudo snapper -c root create-config /
		
		```

	Auto Snapshots Timer and Cleanup

	```
	vim /etc/updatedb.conf
	```
	PRUNENAMES = ".snapshots "
	PRUNEPATHS = "/media"
	PRUNE_BIND_MOUNTS = no

	Disable indexing of snapshots
	[wiki-preventing-slowdowns](https://wiki.archlinux.org/title/Snapper#Preventing_slowdowns)

	There is a gui application available for snapper that can be installed with:

	```
	paru -S btrfs-assistant
	```


> At this point we have a system with a good base and security features while also offering style(kinda) and convinience. You can stop at this point and install anything else you need for your workflow, but we know you won't, which brings us to the next section.

---

# 5. Ricing
##### The stuff everyone actually cares about
1. Dark Mode
	#####Dark mode
	This is a quick and easy dark mode using the adwaita theme, [Read More](https://wiki.archlinux.org/title/Dark_mode_switching)
	
	```
	sudo pacman -S gnome-themes-extra
	paru -S qt5-adwaita-git qt6-adwaita-git
	export GTK_THEME=Adwaita:dark
	export GTK2_RC_FILES=/usr/share/themes/Adwaita-dark/gtk-2.0/gtkrc
	export QT_STYLE_OVERRIDE=Adwaita-Dark
	```
	

2. ZSH
3. Splash Image
	
	```
	sudo pacman -S imagemagick
	magick in.png -type truecolor put.bmp
	```

4. Hyprland
 5. Workspace Overview
 6. Fractional Scaling
 7. Notification Daemon
 8. [l1](https://github.com/mylinuxforwork/dotfiles/tree/main/share) and [l2](https://www.youtube.com/watch?v=J1L1qi-5dr0)
5. [Night Light](https://wiki.archlinux.org/title/Redshift)
6. Plymouth
7. Wofi
8. Waybar
9. Wlogout
10. Pywal
11. kitty
12. Plymouth
---

# 6. Extras
##### At this point, you are pretty much set up. I will document some more stuff here, which might be interesting to you, but also can be safely ignored if you wish so.
1. Gaming
   1. Steam + Gamescope
   2. Heroic Games Launcher
2. AppArmor
3. [Power Management](https://wiki.archlinux.org/title/Power_management#)
4. VS Code
5. Clipboard manager
6. [CPU Frequency Scaling](https://wiki.archlinux.org/title/CPU_frequency_scaling)
7. [fwupd](https://github.com/fwupd/fwupd)
8. Enable https://wiki.archlinux.org/title/Pkgstats to help the community
9. Extra kernels
10. Sign an Arch iso with your keys