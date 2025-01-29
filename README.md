Table of Contents

- [1. Introduction and Credits](#1-introduction-and-credits)
- [2. Pre-Install](#2-pre-install)
- [3. Installation](#3-installation)
- [4. Post-Install](#4-post-install)
- [5. Extras](#5-extras)

---
---
# 1. Introduction and Credits
> I started this as a personal reference document, but I ended up formatting it as a guide since I think it will be helpful to many people out there who want the same type of install. 
I will walk through and explain the rationale behind all the steps from beginning to end, towards installing what is (subjectively) a perfect arch linux install.
Please note that this IS NOT meant to be a replacement of the official arch installation documentation. It's just how I did it.
If you haven't already, read the [official guide](https://wiki.archlinux.org/title/Installation_guide) and try that out in a VM and understand what is happening. 
Then if you want a system like me, follow this guide for reference.

### This Guide will help you have the following install:

1. BTRFS Filesystem + Snapper rollbacks.
2. Root Partition Encryption + TPM unlocking.
3. Unified Kernel Images.
4. Secure Boot.
5. Snapshot Booting with rEFInd btrfs.
6. System Maintainence and Best Practises.
7. Hyprland + Apps.
8. Ricing and Dotfiles.

### I have tested it on the following hardware:

1. AMD CPU (x86-64)
2. AMD GPU (7900 GRE)
3. Asrock Motherboard with UEFI support.

All the steps mentioned here should regardless work for you. If you have an INTEL CPU and/or a NVIDIA GPU, there are some minor differences, and I will point them out.


#### <mark style='background-color:rgb(46, 152, 154)'> While I will try to explain what I'm doing here and why, this is not a noob friendly guide. You need to have an understanding of basic concepts of linux and know your way around the terminal. </mark>


<br>
A lot of the stuff I will talk about here, was hugely inspired by other people in the community and their work. Special Credits to:

1. [This guide by Walian which was the main motivator.](https://walian.co.uk/arch-install-with-secure-boot-btrfs-tpm2-luks-encryption-unified-kernel-images.html)
2. [This guide based on fedora by Madhu@sysguides.](https://sysguides.com/install-fedora-with-snapshot-and-rollback-support)
3. [And offcourse, the Arch Wiki.](https://wiki.archlinux.org/title/Main_page)

<br>
Finally, I am also by no means an expert, so I welcome any and all feedback on this. Lets arch.

---
---

# 2. Pre-Install
This is all the preparation you need to do before attempting the install.

### 1. Download the arch iso

Either use the BitTorrent download (needs a bit-torrent client installed on your machine) or choose your nearest **https** mirror from the [Arch Download Page](https://archlinux.org/download). <br><br>


### 2. Verify the Downloaded Image (Optional but recommended)

```
gpg --keyserver-options auto-key-retrieve --verify archlinux-version-x86_64.iso.sig
```	

Where gpg is GnuPG, which needs to be installed on an existing system, and archlinux-version-x86_64.iso.sig is the file you download from the same webpage as the iso. <br><br>

### 3. Create an Installation Medium

You need a flash drive that can be formmated to hold the installation image.

On windows, use a tool like [Rufus](https://rufus.ie/en/) or [Balena Etcher](https://etcher.balena.io/) . On linux, [KDE ISO Image Write](https://apps.kde.org/isoimagewriter/) or [Gnome Disk Utility](https://gitlab.gnome.org/GNOME/gnome-disk-utility) are some  good Gui applications to do this. Many more methods documented on the wiki [here](https://wiki.archlinux.org/title/USB_flash_installation_medium). <br>
	
<details>
<summary>Ventoy</summary>

You can alternatively use a software like ventoy, which prevents the need to burn a iso image to a drive, while also allowing you to have multiple iso images on the same disk. I personally use and love it. You can check out more about here on it's [Github page](https://github.com/ventoy/Ventoy).
</details>
<br>

### 4. Identify the Installation Target (Optional but recommended)

If you're like me, and are installing linux on a separate drive, while already having windows on another drive, you need to check and make sure to correctly identify the drive. In linux, you can list all your drives using the command 'lsblk'. It labels SATA drives as sda1,sda2,... and nvme drives as nvme0n1, nvme1n1, etc. To avoid nuking your windows install, make a note of the correct drive. You can identify drives by their labels, existing partition layouts, storage capacity, etc.

---
---

# 3. Installation

In this section, we will boot off the Arch USB disk created in the previous section and use it to install a minimal Arch Linux with all the base utilities.

### 1. Boot from the ISO

Plug in your usb with the Arch ISO, reboot to your PC's Motherboard settings (lookup how to for your model, generally it's by pressing the F2, F10 or Del key during boot), and in the boot priority, set the usb as the first to boot from. 

**If secure boot is enabled, you will have to disable it.**
	
>Further in this guide, I will show you how to generate custom secure boot keys. We will sign our boot files with those keys, so that we can then re-enable secure boot.

> As a bonus step, I will also show how you can sign the arch iso with the same secure boot keys, so that if you ever need to fix an issue on your system using the iso, you don't have to turn off secure boot.

<br>

### 2. Internet Connectivity

If you have a wired ethernet connection, just check for connectivity using the *ping* command and proceed to step 3.<br>

If you're on wifi, run the below commands sequentially to connect to your network:

```
iwctl
device list
device wlan0 set-property Powered on
station wlan0 scan
station wlan0 get-networks
station wlan0 connect <wifi-name>
<Enter you wifi password>
exit
```

> Here wlan0 is the name of my wifi device as output by *device list*, and \<wifi-name> will be substituted by the name of my Wifi Connection.


Check connectivity using:
```
ping -c 5 archlinux.org
```

<br>

### 3. Check for disks
Run  the below command to list all the disks available on your system:

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
<br>

### 4. Partition your disk
In this step, we will create a new GPT table on the disk, and then create 2 partitions - a 1GB EFI partiton (also referred to as the ESP) and a Root partition on the remaining space.
<br>

> I am seriously debating the size of the EFI partition. While 512Mb has been more than enough for me is the past, I have seen people recommend 1 GB. 
I have a 1 Tb SSD on which I will be installing arch, so it's not a big deal for me, and as I plan to use this system long term, I don't want to have to deal with resizing the partition later, so I am making it 1 GB. 
If you want to have it as 512Mb, use +512M instead of +1G in the command below:

Type in the following commands in order:

````
fdisk $DISK        ---> Starts the fdisk partition tool on the selected disk
g				   ---> Command to create a new gpt disk label
n				   ---> Command to create a new partition
default            ---> Asks for partition number, press enter as default selection is 1, which is correct.
default			   ---> Asks for first sector, press enter as default is start of the Disk Space.
+1G                ---> Asks for last sector, sets size of partition 1, this will make it 1 GB.
n                  ---> Command to create a new partition
default            ---> Asks for partition number, press enter as default selection is 2, which is correct.
default            ---> Asks for first sector, press enter as default is start of the Disk Space after previous partition.
default            ---> Asks for last sector, press enter as default is end of the Disk Space.
t                  ---> Command to change type of partition
1				   ---> Select Partition 1
1                  ---> Set partition type as 1 (EFI System)
t   			   ---> Command to change type of partition
2				   ---> Select Partition 2
23				   ---> Set partition type as 23 (Linux root x86-64)
w                  ---> Command to save all the changes we have done till now
````

For the first partition, we set the parition type as EFI, and for the second one it was set as Linux Root (x86_64) instead of the default (Linux Filesystem).<br>

This setting of partition type, while technically not necessary, is essential, as this will associate a standard partition GUID with it.<br>

This is then used by systemd (only if using systemd-boot) to recoginze our root partition to decrypt and mount it automatically without a crypttab or fstab file.


[Read more about Discoverable Partitions](https://www.freedesktop.org/software/systemd/man/latest/systemd-gpt-auto-generator.html). 

Verify your partition type GUID (separate from a normal partition GUID, output with blkid) with:

```
lsblk -p -o NAME,PARTTYPE
```

Your Root Filesystem (/dev/sdx2 or /dev/nvmenxn1p2) should have a guid of 4f68bce3-e8cd-4db1-96e7-fbcaf984b709. <br>
The EFI parition should have the guid of c12a7328-f81f-11d2-ba4b-00a0c93ec93b


Run lsblk to verify your partition sizes.

If your disk name was sda,sdb, you can skip this step. If it was a nvme drive like me, update the DISK variable as shown below. This is because partitons in nvme are named as nvme1n1p1, nvme1n1p2 while other are named as sda1,sda2,
	
```
DISK=/dev/nvme1n1p
```
<br>


### 5. Encryption
Now we will encrypt the root partition with luks2.
> When we use luks to encrypt our drive with a password or a key, what it does is create a header. This header is what actually encrypts and decrypts the drive. 
The password is saved in a 'keyslot', which unlocks only the header. This is different that plain encryption, where the password is used to encrypt the entire disk. This allows a greater flexibility when creating and managing passwords and keys. We can also have multiple 'keyslots' created that can unlock the header.

> One downside of this approach is that if our header ever gets corrupted, we lose the ability to unlock our entire drive. To account for this, we will also backup our drive header further in the guide, so that it can be restored if the need arises.
	
<br>

Run the following commands to encrypt the Root Partition:

```
cryptsetup -v luksFormat --type luks2 ${DISK}2
cryptsetup open --type luks --allow-discards --perf-no_read_workqueue --perf-no_write_workqueue --persistent ${DISK}2 root
```
<br>

In the first command we formatted our disk with luks2, it will autogenerate the header and ask you to provide a password to unlock. Make this password strong, or better yet use a [passphrase](https://www.techtarget.com/searchsecurity/definition/passphrase)
	
In the second command, we open our encrypted drive, and give it the name of 'root'. From here on, our root partition isn't /dev/sda2 or /dev/nvme1n1p2, but rather it's /dev/mapper/root
	
<br>
I also used some options while opening the encrypted drive. They do the following:

<br>

The *--allow-discards* enables [TRIM](https://wiki.archlinux.org/title/Solid_state_drive#TRIM)  support for a SSD [Read More](https://wiki.archlinux.org/title/Dm-crypt/Specialties#Discard/TRIM_support_for_solid_state_drives_(SSD)).
	
The *--perf-no_read_workqueue* and *--perf-no_write_workqueue* increases performance for SSD's [Read More](https://wiki.archlinux.org/title/Dm-crypt/Specialties#Disable_workqueue_for_increased_solid_state_drive_(SSD)_performance)

The *--persistent* option saves these options to the header, so they are always used by default in the future.

<br>
You can check the flags that your drive is opened with using:
	
<br>

```
cryptsetup luksDump /dev/sdaX | grep Flags
```
> I will be relying on systemd-boot to automatically recognize and decrypt the root partition. If you decide to use grub, you have to so some more setup in the crypttab file.

<br>


### 6. Format your Partitions

Now I will format the efi partition as a Fat32 and the root partition as BTRFS filesystem.

```
mkfs.vfat -F32 -n ESP ${DISK}1
mkfs.btrfs -L root /dev/mapper/root
```
	
Check if the partitons are correctly created:
```
blkid -o list
```
		
<br>

### 6. Btrfs Subvolumes
> This section is a lot more subjective to the type of installation you prefer. Basically, since we are on a btrfs filesystem, and will be using the rollback functionality on the root subvolume, you can choose which folders on your root won't be rolled back. 
They can have their own rollback logic created. To do this, they need to be mounted as separate subvolumes. Choosing which folders should be subvolumes has no definitive answer.
	
	
The archinstall script creates the following subvolumes - /, /home, /var/log, /var/cache/pacman/pkg and the '@' naming scheme.

[OpenSuse Recommends the following Subvolumes](https://en.opensuse.org/SDB:BTRFS) - /home, /opt, /root, /srv, /tmp, /usr/local, /var.

<br>

This is the subvolume layout and I personally use, and which has worked well for me:

| Subvolume Name    | Mount Point | Purpose |
| ----------------- | ----------- | ------- |
| @ | / | The root folder, which is also a separate subvolume below the btrfsroot volume, and which we will rollback in case of any issues. |
| @home | /home |  Home Folder where all your Data/Games/Configurations will reside. |
| @mozilla | /home/$USER/.mozilla | The directory where your firefox data is stored. If you ever rollback your home directory, this will prevent any potential loss of browsing data. |
| @ssh | /home/$USER/.ssh | Same as above, to protect any ssh keys/configs you have. |
| @opt | /opt |  This is where third party applications are installed.. |
| @var | /var |  Where all temporary files/logs/cache is stored. |
| @snapshots | /.snapshots |  This subvolume will store snapshots of our @ subvolume (Managed via Snapper). |
| @home-snapshots | /home/.snapshots | This subvolume will store snapshots of our @home subvolume (Managed via Snapper). |


Run the below commands to create all the subvolumes:

```
mount /dev/mapper/root /mnt
cd /mnt
btrfs subvolume create {@,@home,@mozilla,@ssh,@var,@opt,@snapshots,@home-snapshots}
btrfs subvolume list /mnt
```

The final commands lists all the subvolumes with their ID.
Get subvol id of @ and set it as default for the root (eg. 256)

```
btrfs subvolume set-default 256 /mnt
```

>I am making the whole of var into one subvolume. This is not recommended by arch, as pacman stores it's cache in the /var/cache/pacman directory. 
But I am going to configure pacman cache to be in the tmp directory, you can read the rationale behind this [here](https://www.reddit.com/r/archlinux/comments/1hgbl1k/what_is_varcachepackagepkg_and_why_is_it_so_large/). 
But essentially, I am not saving any pacman cache.

Optionally, if you use Thunderbird, Chrome, or Gnupg you can also create the below subvolumes to preserve their data similar to @mozilla above.
| Subvolume Name    | Mount Point |
| ----------------- | ----------- | 
| @thunderbird | /home/$USER/.thunderbird |
| @chrome | /home/$USER/.config/google-chrome |
| @gnupg | /home/$USER/.gnupg |



>Btrfs subvolumes can also be created in the future if you need it. 
		
<br>

### 7. Mount Options

Mount all the subvolumes you created using the same mount options as the root.

```
umount /mnt
mount -o defaults,noatime,space_cache=v2,ssd,compress-force=zstd:1,commit=120,subvol=@ /dev/mapper/root /mnt
mount -o subvol=@home /dev/mapper/root /mnt/home
mount -o subvol=@home /dev/mapper/root /mnt/home
mount -o subvol=@opt /dev/mapper/root /mnt/opt
mount -o subvol=@var /dev/mapper/root /mnt/var
```

We will mount the @mozilla, @ssh, @snapshots and @home-snapshots subvolumes,  later on.

> Once a subvolume is mounted with a set of options, all other subvoumes follow the same options. This is fine mostly as I need the same options everywhere, except in the @var directory, where I want to have the [Nodatacow](https://www.reddit.com/r/btrfs/comments/n6slx3/what_is_the_advantage_of_nodatacowdisabling_cow/) mount option. 
So instead I will set the attribute +C on the var directory, which accomplishes the same thing.

Run this command to disbale CoW in the @var subvolume

```
chattr +C /mnt/var
```

[Read more about various BTRFS mount Options](https://btrfs.readthedocs.io/en/stable/Administration.html#mount-options)
<br>
<br>

Also mount the efi partition using:

```
mkdir -p /mnt/efi
mount -o defaults,fmask=0077,dmask=0077 /dev/{DISK}1 /mnt/efi
```


### 8. Update Mirrors and Pacstrap

Before we download and install the necessary packages, let us update our mirrors, for the best download speed, by running the refelctor command. Replace $COUNTRY with your country name:

```
reflector --country $COUNTRY --age 24 -l 10 --protocol https --sort rate --save /etc/pacman.d/mirrorlist
```

Now, we install the base required packages for an Arch Linux install using the pacstrap command on /mnt where the system root is mounted. Press enter after running these commands:
	
```
pacman -Sy archlinux-keyring
pacstrap -K /mnt base base-devel linux linux-firmware amd-ucode cryptsetup btrfs-progs dosfstools posix util-linux git networkmanager sudo openssh vim reflector
```

#### <mark style='background-color:rgb(46, 152, 154)'> If you have an intel cpu, replace amd-ucode with intel-ucode above. </mark>


### 9. Chroot into the install and do basic setup

>Chroot or Change-Root makes it so that the system behaves as if you are the root user of the new install. This way you can setup some basic things as the Root user without needing to reference /mnt as the root.

```
arch-chroot /mnt
```

<br>

1. Set the system timezone, Update the system time and Setup a NTP server for Clock Accuracy using the below commands (replace /Asia/Kolkata with your respective timezone)
		
	```
	ln -sf /usr/share/zoneinfo/Asia/Kolkata /etc/localtime
	timedatectl set-ntp true
	systemctl enable systemd-timesyncd
	```

	[Read this if you are dual-booting windows](https://wiki.archlinux.org/title/System_time#UTC_in_Microsoft_Windows)

2. Locale

	```
	vim /etc/locale.gen
	```

	Uncomment your desired locale by removing the # at the start. Also uncomment en_US.UTF-8 ([Why?](https://wiki.archlinux.org/title/Steam#Installation))
	
	Save the file and run:

	```
	locale-gen
	```

	Then run the following command and provide appropriate input as asked (go with defaults if in doubt):

	```
	systemd-firstboot --prompt
	```

3. Pacman

	Improve pacman defaults:
			
	```
	vim /etc/pacman.conf
	```
	Under misc option, uncomment (blue) and add(purple) the following lines:

	---
	#### <mark style='background-color:rgb(46, 152, 154)'> Color </mark>
	#### <mark style='background-color:rgb(46, 152, 154)'> ParallelDownloads = 5 </mark>
	#### <mark style='background-color:rgb(82, 80, 240)'> ILoveCandy </mark>
	---

	Save and Exit.

	<br>
	Changing the Pacman cache directory location:

	```
	vim /etc/pacman.conf
	```

	Remove the '#' from the line starting with #CacheDir save:

	---
	#### <mark style='background-color:rgb(46, 152, 154)'> CacheDir     = /tmp/cache-pacman/pkg/ </mark>
	
	---

	
### 10. User Management
1. Set Root Password
	```
	passwd
	```

2. Create your user and set a password.

	```
	useradd -G wheel -m $USER
	passwd $USER
	```

3. Allow your user to run sudo commands with a password
	
	```
	visudo
	```

	Uncomment by removing the '#' of the **FIRST** line starting wiht '# %wheel'. Use

	---
	#### <mark style='background-color:rgb(46, 152, 154)'> %wheel ALL=(ALL:ALL) ALL </mark>

	---


4. Mount 2 of the remaining 4 subvolumes:
	```
	mkdir /home/$USER/{.mozilla,.ssh}
	mount -o subvol=@mozilla /dev/mapper/root /mnt/home/$USER/.mozilla
	mount -o subvol=@ssh /dev/mapper/root /mnt/home/$USER/.ssh
	```
<br>

### 11. Generate fstab

> Fstab is a file referenced by the system during boot to mount your drives/partitions to the correct location. Since we have already mounted our drives, we will use the **genfstab** utility to output the fstab file with the options we chose earlier, and save it, so that in the future these options are used by default.

```
mkdir /etc
touch /etc/fstab
genfstab -U / >> /etc/fstab
```

<details>
<summary>Why am I using fstab when discoverable partitions exist?</summary>

Discoverable partitons is a cool feature which can automount your drives, and can fully be used in this install, as I will be using systemd-boot as my bootloader.
However, I will still manually define my mount points in fstab. This is because:
	a. I don't see an option of defining my mount options, and I don't want them to be the system defaults
	b. I will still have to mount my btrfs subvolumes manually, so why not do it all myself?
	c. I don't like the abstraction, I prefer to setup my system myself.
</details>

<br>	

### 12. Generate Unified Kernel Images
[Unified Kernel Images](https://wiki.archlinux.org/title/Unified_kernel_image) is a concept of generating a single efi file to boot from. This helps in maintainence and signing for secure boot.

While this may sound difficult, it's been made very easy by various tools. I will use the [mkinitcpio](https://wiki.archlinux.org/title/Mkinitcpio) in this instance. By just changing a few Configuration options, it will generate the UKI's automatically now and in the future.

Run the below commands:
```
echo "quiet rw" >/etc/kernel/cmdline
mkdir -p /efi/EFI/Linux
vim /etc/mkinitcpio.conf
```
	
Edit the line starting with *HOOKS* in mkinitcpio.conf to look like this:

---
#### <mark style='background-color:rgb(46, 152, 154)'> HOOKS=(base systemd autodetect modconf kms keyboard sd-vconsole sd-encrypt block filesystems fsck) </mark>
---

Now we need to edit the linux kernel preset file. (If you have installed additional kernels, you need to update their preset files as well.)

```
vim /etc/mkinitcpio.d/linux.preset
```
Comment the line starting with default_image and fallback_image and uncomment the lines starting with ALL_config, default_uki, default_options and fallback_uki.
Make the file look like this:

---

All_config="/etc/mkinitcpio.conf"

ALL_kver="/boot/vmlinuz-linux"

PRESETS=('default', 'fallback')

#default_config="/etc/mkinitcpio.conf"

#default_image="/boot/initramfs-linux.img"

default_uki="/efi/EFI/Linux/arch-linux.efi"

default_options="--splash /usr/share/systemd/bootctl/splash-arch.bmp"

#fallback_config="/etc/mkinitcpio.conf"

fallback_image="/boot/initramfs-linux-fallback.img"

fallback_uki="/efi/EFI/Linux/arch-linux-fallback.efi"

fallback_options="-S autodetect"


---


Run below command to generate the unified kernel images:
```
mkinitcpio -P
```

### 13. Enable services

Network Services:

```
systemctl enable systemd-resolved NetworkManager
systemctl mask systemd-networkd
systemctl enable reflector
systemctl enable sshd
```
[Read more about Reflector config](https://ostechnix.com/retrieve-latest-mirror-list-using-reflector-arch-linux/).

<br>

### 14. Bootloader Installation
In this step, I will install the systemd-boot bootloader on the /efi partition. Then I will reboot to the Motherboard settings:

```
bootctl install --esp-path=/efi
sync
exit 
systemctl reboot --firmware-setup
```

In the Firmware setup menu, select the entry for Arch/The Disk as the First Priority Entry and Save the settings.
You will be asked to enter the disk encryption password, post which you can login and use the your new Arch System.


**Congrats, we have a minimal arch linux system installed.**

<br>

---
---

# 4. Post-Install
> While we have a working system now, we are missing a lot of utilities which make it actually usuable. In this section, I will walk through installing and configuring them.

Boot into your system, and login as your user.
### 1. Update Pacman and install some essential packages:
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

	>The amdgpu-top package is a tui+gui application to monitor your gpu usage. It needs paru/yay installed, which I do in step 2.

	#### <mark style='background-color:rgb(46, 152, 154)'> Nvidia GPU:</mark>
	>I don't have an Nvidia GPU, so you have to test this part yourself, refer to this [link](https://github.com/korvahannu/arch-nvidia-drivers-installation-guide).
	
2. AUR Helper
	>I will install [paru](https://github.com/Morganamilo/paru) to help me download and install AUR packages. See also: [Yay](https://github.com/Jguer/yay)
				
	```
	sudo pacman -S --needed base-devel
	git clone https://aur.archlinux.org/paru.git
	cd paru
	makepkg -si
	```
	
	>After this step, you will be able to use the command *paru -S \<package-name>* to install a package from the[AUR](https://aur.archlinux.org/). Be very careful installing packages from here.

3. Shell and Fonts

	```
	sudo pacman -S zsh ttf-jetbrains-mono-nerd
	chsh /usr/bin/zsh
	```

4. Audio
		
	```
	sudo pacman -S pipewire pipewire-pulse pavucontrol
	```

	>These won't be really relevant until we install Hyprland.

5. Utilities
			
	```
	sudo pacman -S unzip man-db man-pages wget htop tpm2-tss
	```


6. Flatpaks
	
	```
	sudo pacman -S flatpak
	```

	You can search for applications you need on [Flathub](https://flathub.org/).

   
7. [Optional Repositories](https://wiki.archlinux.org/title/Official_repositories#)

    I will be enabling the Multilib repo (because Steam isn't available on the default repo as it's a 32 bit application).
		
	```
	vim /etc/pacman.conf
	```
		
	Uncomment the below lines by removing the '#'
	
	---
	[multilib]

	Include = /etc/pacman.d/mirrorlist
	
	---

	Then run:

	```
	sudo pacman -Sy
	```


### 2. Install Hyprland and other helpful gui applications
	
```
sudo pacman -S hyprland firefox dolphin kitty wofi waybar hyprpolkitagent grim kitty qt5-wayland qt6-wayland slurp xdg-desktop-portal-hyprland xdg-utils xorg-xwayland vlc

paru -S wlogout
```

Set this line in your hyprland config:

---

exec-once = systemctl --user start hyprpolkitagent

---

https://wiki.archlinux.org/title/XDG_MIME_Applications


### 3. Install rEFInd
Refind is a boot manager. Simply put, when starting your PC, it will allow you to choose between Windows and Linux. It can also be themed to make the boot process look pretty.
```
sudo pacman -S refind
sudo refind-install
```

<details>
<summary>Why do I need both rEFInd and systemd-boot? </summary>

- rEFInd supports Btrfs Snapshot Booting
- It is customizable
- I want the main linux bootloader to do just that, and not have other fancy features, improving realiability.
</details>

[Refind BTRFS](https://github.com/Venom1991/refind-btrfs) will be configured later



### 4. Setup Secure Boot
We will generate our own secure boot keys, enroll them in our Motherboard firmware, and sign our kernel modules and other efi binaries with them. There's an amazing tool known as [sbctl](https://github.com/Foxboron/sbctl) which does most of the heavy lifting for us.

```
sudo pacman -S sbctl
```

Once sbctl is downloaded, you need to make sure your motherboard is in setup mode, so that you can enroll your keys into it. This is done typically by clearing out the existing secure boot keys in the motherboard settings. Look up how to do this for your brand of motherboard.

>Many motherboards have the ability to restore the keys that you removed (I can confirm this for Asrock). Also if you ever update the firmware of the motherboard, you might have to enroll your own keys again.

Check if motherboard is in Setup mode from the output of:

```
sbctl status
```

Now we will generate our keys, enroll them into the motherboard(along with Microsoft keys for windows comapatibility).

These keys are stored in the '/var/lib/sbctl/keys' directory. It's not a bad idea to back them up.

```
sudo sbctl create-keys
sudo sbctl enroll-keys -m
sudo sbctl status
```

After confirming the enrollment of keys, we will sign our kernel, bootloader and boot manager(refind) files with our keys. 

If any of these images get updated, we will also instruct sbctl to sign them again automatically.

The *sbctl verify* command lists all files that need to be signed and checks whether they are signed or not. 

We will then use *sbctl sign -s \<filename>* to sign them.

```
sudo sbctl verify
sudo sbctl sign -s /efi/EFI/Linux/arch-linux.efi
sudo sbctl sign -s /efi/EFI/Linux/arch-linux-fallback.efi
sudo sbctl sign -s /efi/EFI/BOOT/BOOTX64.EFI
sudo sbctl sign -s /efi/EFI/systemd/systemd-bootx64.efi
sudo sbctl sign -s /efi/EFI/refind/drivers_x64/btrfs_x64.efi
sudo sbctl sign -s /efi/EFI/refind/refind_x64.efi
```

You might have to sign more files, if you have installed additional kernels. Use the *sbctl verify* command to check if any more files need signing.
Now we can re-enable secure boot, and it should work fine.



### 5. Improve Encryption Setup 
[Reference Guide](https://0pointer.net/blog/unlocking-luks2-volumes-with-tpm2-fido2-pkcs11-security-hardware-on-systemd-248.html)

1. Backup LUKS Headers

	I have already explained previously why you might want to backup your encrypted drive's Luks header, which stores data about and controls your drive decryption.
	To backup the header, run:

	```
	sudo cryptsetup luksHeaderBackup /dev/gpt-auto-root-luks --header-backup-file luks-header-backup.img
	```

	This will generate a file 'luks-header-backup.img' in your current directory. 
	
	Save this in another drive, in the cloud (depending on your threat model), or into a password manager like bitwarden. If your header ever gets corrupted, you can boot from another iso, and restore it using:

	```
	cryptsetup luksHeaderRestore /dev/device --header-backup-file /path-to/luks-header-backup.img
	```

2. Manage Keyslots:

	While we normally use a passord to unlock the luks header, it can support multiple unlocking options, one of which is a keyfile. 
	
	In this optional step, I will generate a keyfile with random data, and enroll it into the header. This can then be used to unlock the drive.

	Generate a strong key into a file named my-encryption-key in your current folder:

	```
	openssl genrsa -out my-encryption-key 4096
	```

	Enroll this key into your root drive:
	```
	sudo cryptsetup luksAddKey /dev/gpt-auto-root-luks my-encryption-key
	```

	You can use this key to unlock the drive (assuming you are booted from an iso) using:
	```
	cryptsetup open --type luks /dev/sda2 root --key-file my-encryption-key
	```

	<br>

	View all keyslots in your drive's luks header:
	```
	cryptsetup luksDump /dev/gpt-auto-root-luks
	```

	or
	```
	sudo systemd-cryptenroll /dev/gpt-auto-root-luks
	```

	The keyslots are numbered starting from 0, in the order that you added them.

	You can delete an exsting keyslot using:
	```
	cryptsetup luksKillSlot /dev/gpt-auto-root-luks \<keyslot-number>
	```

	Be careful not to remove all the keys as this might render your drive unlockable. (You can recover from such an accident if you hvae the Luks Header Backed up)


3. Setup auto unlocking via TPM

	We can setup our encrypted drive to be unlocked automatically by saving a key on the TPM chip (present in most modern PC's) and enrolling it into the luks header with the help of [Systemd-cryptenroll](https://wiki.archlinux.org/title/Systemd-cryptenroll).

	This key can also be bound the PCR's on the TPM (basically they do some verifications on your system beforr allowing the key to be used.)
	
	Check TPM support on your PC using:

	```
	systemd-analyze has-tpm2	
	```

	If you see an output like tpm0, them a tpm device exists
		
	Choose which PCR's to use. A pcr is a register on the TPM that can measure specific values, like is secure boot on or off, what is the firmware version,etc. You can see the full list [Here](https://wiki.archlinux.org/title/Trusted_Platform_Module#Accessing_PCR_registers)

	I will use PCR 0,7. This makes it so that if I upgrade my Motherboard firmware or turn off secure boot, the TPM won't be able to unlock my drive. You can use more PCR's if you wish so.

	The following command generates a key, and enrolls it both in the TPM and the Luks header, while also binding it to the PCR's 0 and 7:
		
	```
	sudo systemd-cryptenroll --tpm2-device=auto --tpm2-pcrs=0+7  /dev/gpt-auto-root-luks
	```


### 6. Swap and Hibernation
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

[Unit] <br>
Description="Disable GPP0 to fix suspend issue"

[Service]<br>
ExecStart=/bin/sh -c "/bin/echo GPP0 > /proc/acpi/wakeup"

[Install]<br>
WantedBy=multi-user.target

### 7. Security
	1. Firewall
	2. 

### 8. System Maintainence

1. [Maintainence](https://gist.github.com/MaxXor/ba1665f47d56c24018a943bb114640d7)


2. Btrfs filesystem
	1. [Defragmentation](https://wiki.archlinux.org/title/Btrfs#Defragmentation)
	2. Scrub
	3. Balance


3. Snapshot Rollbacks
	1. [Snapshot Rollbacks]
	2. [Snapshot Booting](https://wiki.archlinux.org/title/Btrfs#Booting_into_snapshots)


### 9. Snapper and snapshots
> Alright, all the efforts we put into btrfs and subvolumes, will help us now.

By now, you must be aware of the btrfs snapshot feature. There is a tool developed by OpenSuse know as [snapper](https://github.com/openSUSE/snapper) which is a helper application for this purpose. Using snapper, we will create a config for our root subvolume (@) containing most of our system data, and another for the home subvolume(@home), which contains most of our user data. Then we can setup snapper to create and manager snapshots of them, to safeguard and if necessary, rollback this data (Remember, the other subvolumes we created are exempt from this). Snapper can also do automatic snapshots on a schedule, and there's a pacman hook known as snap-pac that can create snapshots whenever we use pacman to install/upgrade our system.

```
sudo pacman -S snapper snap-pac
paru -S btrfs-assistant
```

Run the below commands in order to:
	1. Create Snapper Configurations
	2. Enable your user to work with them without sudo privileges
	3. Setup Automatic Creation and Cleanup of snapshots
	4. Disable Indexing of snapshtos to improve performance.

	```
	sudo snapper -c root create-config /
	sudo snapper -c home create-config /home
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



> At this point we have a system with a good base and security features while also offering style(kinda) and convinience. You can stop using the guide at this point and make most of the further decisions on your own as to what type of theming and workflow you prefer. I will document my own rice and dotfiles in another Repo. Meanwhile, there are more things that can be done with Arch linux, refer the next section for that.

---
---

# 5. Extras
> At this point, you are pretty much set up. I will document some more stuff here, which might be interesting to you, but also can be safely ignored if you wish so.
1. Gaming
   1. Steam + Gamescope
   2. Heroic Games Launcher
2. Virtualization
3. AppArmor
4. Stay upto date with arch news
5. [Power Management](https://wiki.archlinux.org/title/Power_management#)
6. VS Code
7. Clipboard manager
8. [CPU Frequency Scaling](https://wiki.archlinux.org/title/CPU_frequency_scaling)
9. [fwupd](https://github.com/fwupd/fwupd)
10. Enable https://wiki.archlinux.org/title/Pkgstats to help the community
11. Extra kernels
12. Sign an Arch iso with your keys
