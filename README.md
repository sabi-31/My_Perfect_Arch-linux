# My_Perfect_Arch-linux

### This is a guide/personal reference/explanation of an ideal arch linux base install + all the good ricing on top of it. I am mainly creating this as a personal reference guide, but hope it's also helpful to someone else learning this stuff.

### List of Resources that helped me and from which I picked up a lot of the stuff outlined in this guide, huge thanks to the original authors
1. https://walian.co.uk/arch-install-with-secure-boot-btrfs-tpm2-luks-encryption-unified-kernel-images.html
2. https://sysguides.com/install-fedora-with-snapshot-and-rollback-support

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
		DISK = /dev/nvme1n1
		```
		```
		echo $DISK
		```
	   
1. 
   
## Post-Install

## Ricing
