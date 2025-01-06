## Table of Contents

1. [Introduction and Credits](#introduction-and-credits)
1. [Pre-Install](#pre-install)
2. [Installation](#installation)
3. [Post-Install](#post-install)
4. [Ricing](#ricing)
5. [The Process Continues](#the-process-continues)

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

> A lot of the stuff I will talk about here, was hugely inspired by other people in the community and their work. This guide is not to diminish their efforts, but to build upon them. 

Special Credits to:

1. [This Extensive guide by Walian which was the main motivator.](https://walian.co.uk/arch-install-with-secure-boot-btrfs-tpm2-luks-encryption-unified-kernel-images.html)
2. [This guide based on fedora by Madhu@sysguides.](https://sysguides.com/install-fedora-with-snapshot-and-rollback-support)
3. [And offcourse, the Arch Wiki.](https://wiki.archlinux.org/title/Main_page)

> I will try and keep the sections as modular as possible, so if someone wants to, for example, skip setting up secure boot, they can still follow along. I will also be linking some articles and blogposts which I found extremely helpful to understand some of the concepts, which you should definitely read. I am also by no means an expert, so I welcome any and all feedback on this. Lets start:

---

# Pre-Install
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

# Installation
##### The Actual Install Process Begins

---

# Post-Install
##### First Boot and setting up some defaults as well as basic checks on the system

---

# Ricing
##### The stuff everyone actually cares about

---

# The Process Continues
##### Given the nature of FOSS software, there are always changes/improvements possible. I will document them here