<script>
  function openCity(evt, cityName) {
    // Declare all variables
    var i, tabcontent, tablinks;

    // Get all elements with class="tabcontent" and hide them
    tabcontent = document.getElementsByClassName("tabcontent");
    for (i = 0; i < tabcontent.length; i++) {
      tabcontent[i].style.display = "none";
    }

    // Get all elements with class="tablinks" and remove the class "active"
    tablinks = document.getElementsByClassName("tablinks");
    for (i = 0; i < tablinks.length; i++) {
      tablinks[i].className = tablinks[i].className.replace(" active", "");
    }

    // Show the current tab, and add an "active" class to the link that opened the tab
    document.getElementById(cityName).style.display = "block";
    evt.currentTarget.className += " active";
  } 
</script>

(This page is a work in progress and is currently being edited on 2022-07-05)

## What is ZFSBootMenu?

ZFSBootMenu is a _boot manager_ for Linux - just like GRUB, only better. A boot manager is the first software that runs when you start your computer, and it's responsible for finding your Linux installation and starting it. 

ZFSBootMenu is designed to start Linux installations that have been installed directly on to ZFS, and has a number of advantages over GRUB, for example:

* It can allow you to boot a previous "snapshot" of your Linux installation if something has gone wrong during one of your regular system updates.
* It provides a full linux system recovery shell with ZFS installed, making it easy to do recovery of a system with a damaged ZFS pool.
* It's much simpler to install than GRUB when building a Root-on-ZFS Linux system.

## Great! How can I use it on my machine?

Assuming that your machine supports EFI booting (and most machines made after 2011 do), then there are three things that you need to use ZFSBootMenu:

1. An EFI system partition with ZFSBootMenu on it, and that has been registered with your EFI firmware.
1. A zpool with a dataset on it to hold your Linux installation.
1. A Linux installation, located on the dataset in your zpool.

If you have those three things, then on startup your computer's EFI firmware will launch ZFSBootMenu. ZFSBootMenu will then scan your disks looking for zpools and for datasets that have Linux installations on them, and will present you with a menu so you can choose which one to start. ZFSBootMenu also gives you a menu of options for inspecting the various zpools and Linux installations on your computer, and you can even enter a full Linux shell with ZFS installed to make any changes or repairs that you need to before booting.

The steps for setting up those three things on your computer will depend on where you are starting from. You might want to do a clean install, wiping your disk(s) and starting from scratch, or you might have an existing Root-on-ZFS installation using GRUB that you want to convert to ZFSBootMenu. We'll cover these two scenarios in the sections below, and hopefully if you have a different scenario you will be able to figure out what you need to do from these examples as well. 

The exact commands shown in this document are for installing Debian 11, however it should be easy to figure out what the equivalent commands are for other Linux distributions by referring to other guides in this wiki or to the Root-on-ZFS guides for your distribution from the OpenZFS project at https://openzfs.github.io/openzfs-docs/Getting%20Started/Ubuntu/index.html#root-on-zfs

### How do I know if my computer supports EFI booting?

The easiest way to check if your computer supports EFI booting is by looking in your BIOS/firmware system settings when your computer first starts and before it has booted. You do this by pressing a key as soon as your computer starts - often the screen will have a startup message telling you what key to press, but sometimes it just displays a logo and you will need to do a web search to find out which key it is - often it's the [delete] key on Dell or Asus machines, the [Enter] and then [F1] key on Lenovo, or the [F10] on HP, and so on.
Different computers have different system setting menus, so you will have to figure out where the boot settings are and then look for "EFI boot" options, and confirm they are enabled. If there are no EFI settings at all and you have a very old machine, then your computer may not support EFI boot and you will have to consult a different guide on how to set up MBR boot with ZFSBootMenu instead. You can also do a web search for your computer's user manual if you are unsure about the BIOS setting menus or whether EFI boot is supported.

## 1. Installation environment

In order to set up ZFSBootMenu you need a Linux command line with ZFS installed. If you already have a Root-on-ZFS environment and just want to replace GRUB with ZFSBootMenu, then you can just use a root shell on your existing system. If however you want to do a clean install of a Root-on-ZFS Linux system with ZFSBootMenu, then you will need to boot your computer using a "Live USB stick" or "Live CD", so that you don't boot off any of the disks in your machine and are therefore free to erase and configure them without interfering with the Linux command line you are using.

### 1.1 Creating a live USB stick

To create a live USB or live CD, see the instructions on your distribution's website. For this example, we are using Debian 11, so we would follow the instructions at https://www.debian.org/CD/live/ , which involve downloading the live usb .iso file via bittorrent.

Once you have a live USB .iso image for your distribution of choice, you can use another Linux machine to copy it to a USB stick using the dd command. First, you need to identify the device node for your USB stick. Connect it to your machine, and then look in the /dev/disk/by-id directory:

    ls -l /dev/disk/by-id

This will display various id-based aliases for the disks on your system. Find the one that looks like it's your USB stick, and note the "sd*" name that it points to.

    lrwxrwxrwx 1 root root  9 Jul  5 17:03 ata-Samsung_SSD_870_EVO_1TB_S5Y2NF0R12***** -> ../../sda
    lrwxrwxrwx 1 root root 10 Jul  5 17:03 ata-Samsung_SSD_870_EVO_1TB_S5Y2NF0R12*****-part1 -> ../../sda1
    lrwxrwxrwx 1 root root 10 Jul  5 17:03 ata-Samsung_SSD_870_EVO_1TB_S5Y2NF0R12*****-part2 -> ../../sda2
    lrwxrwxrwx 1 root root 10 Jul  5 17:03 ata-Samsung_SSD_870_EVO_1TB_S5Y2NF0R12*****-part3 -> ../../sda3
    lrwxrwxrwx 1 root root 10 Jul  5 17:03 ata-Samsung_SSD_870_EVO_1TB_S5Y2NF0R12*****-part4 -> ../../sda4
    lrwxrwxrwx 1 root root 10 Jul  5 17:03 ata-Samsung_SSD_870_EVO_1TB_S5Y2NF0R12*****-part5 -> ../../sda5
    lrwxrwxrwx 1 root root 10 Jul  5 17:03 ata-Samsung_SSD_870_EVO_1TB_S5Y2NF0R12*****-part6 -> ../../sda6
    lrwxrwxrwx 1 root root 10 Jul  5 17:03 dm-name-swap -> ../../dm-0
    lrwxrwxrwx 1 root root  9 Jul  5 17:03 usb-Generic-_Multi-Card_201209265***00000-0:0 -> ../../sdb
    lrwxrwxrwx 1 root root  9 Jul  5 17:05 usb-SanDisk_Cruzer_Blade_000009031011200*****-0:0 -> ../../sdc
    lrwxrwxrwx 1 root root 10 Jul  5 17:05 usb-SanDisk_Cruzer_Blade_000009031011200*****-0:0-part1 -> ../../sdc1
    lrwxrwxrwx 1 root root 10 Jul  5 17:05 usb-SanDisk_Cruzer_Blade_000009031011200*****-0:0-part2 -> ../../sdc2
    lrwxrwxrwx 1 root root  9 Jul  5 17:03 wwn-0x5002538f411***** -> ../../sda
    lrwxrwxrwx 1 root root 10 Jul  5 17:03 wwn-0x5002538f411*****-part1 -> ../../sda1
    lrwxrwxrwx 1 root root 10 Jul  5 17:03 wwn-0x5002538f411*****-part2 -> ../../sda2
    lrwxrwxrwx 1 root root 10 Jul  5 17:03 wwn-0x5002538f411*****-part3 -> ../../sda3
    lrwxrwxrwx 1 root root 10 Jul  5 17:03 wwn-0x5002538f411*****-part4 -> ../../sda4
    lrwxrwxrwx 1 root root 10 Jul  5 17:03 wwn-0x5002538f411*****-part5 -> ../../sda5
    lrwxrwxrwx 1 root root 10 Jul  5 17:03 wwn-0x5002538f411*****-part6 -> ../../sda6

For example, here I can see that the links to my my USB drive are labeled "usb-SanDisk_Cruzer_Blade...", and point to ../../sdc, or in other words, /dev/sdc.

In this case, assuming that the .iso file has been downloaded to ~/Downloads/debian-live-11.3.0-amd64-kde.iso , the command to copy the .iso onto the USB stick would be:

    sudo dd bs=4M if=~/Downloads/debian-live-11.3.0-amd64-kde.iso of=/dev/sdc conv=fdatasync status=progress

The parts of this command are:
part             | meaning
-----------------|---------------------------------
sudo             | run the command as the root user
dd               | the copy command itself
bs=4M            | copy 4 megabytes at a time
if=...           | name of .iso file to copy from
of=...:          | name of USB device to write to
conf=fdatasync   | make sure data is fully written to the USB stick before finishing
status=progress  | show progress status while the write is running

Once the command completes, you can remove the USB stick and use it to boot the machine that you want to install ZFSBootMenu on. If the computer doesn't boot from the USB stick, check your BIOS system settings as described above, and make sure that USB booting is enabled and is the first boot option in your BIOS boot menu.

### 1.2 Configuring the live USB stick

Once we have used the live USB stick to boot the machine that you want to install ZFSBootMenu on, we need to add a few things to the live environment so that it can work with ZFS. 


## 2. Installing ZFSBootMenu on an EFI System Partition and registering it with your computer's firmware