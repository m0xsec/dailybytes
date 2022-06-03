---
title: "I use Arch, btw"
date: 2022-06-02T23:25:59-07:00
draft: true
tags: ["arch", "linux", "guide", "tpm2.0", "btrfs", "apparmor", "secureboot"]
categories: ["linux"]
showToc: true
TocOpen: true
cover:
    image: "/posts/arch/images/cover.jpg" # image path/url
    alt: "(btw)" # alt text
    caption: "(btw)" # display caption under cover
    relative: false # when using page bundles set this to true
    hidden: false # only hide on current single page
---

I've been wanting to install Arch on my primary workstation for a while though was pretty overwhelmed with figuring out what sort of configuration I wanted. I now have something that I like and wanted to outline what I did. This is mostly for my own reference and documentation, though please feel free to use this and adapt it to your own needs!

This guide will outline how to setup a clean install of Arch linux with the following:
- Secure Boot.
- LUKS + BTRFS with subvolumes.
- AppArmor.
- systemd-boot.
- TPM 2.0 enrollment for LUKS.
- GNOME 42 w/ Wayland and NVIDIA proprietary driver.
<!--more-->

{{< notice note>}}
Please do your own research and check the latest [Arch Install Guide](https://wiki.archlinux.org/title/Installation_guide) for guidance. Sorry if this breaks your system uwu
{{< /notice >}}

# Base Install
The [Arch Install Guide](https://wiki.archlinux.org/title/Installation_guide) is honestly the best way to get started and is a great reference if you get stuck along the way.

## WiFi and Syncing System Clock
You will probably want to connect to WiFi, unless you are directly plugged in or are setting up an air-gapped system.

[`iwctl`](https://wiki.archlinux.org/title/Iwd#iwctl) is probably the easiet way to do this.

```
$ iwctl
[iwd]# device list
[iwd]# station <device> connect <SSID>
```

You can also do this as a one-liner, with the AP passphrase:
`iwctl --passphrase <passphrase> station <device> connect <SSID>`

Once you verify that you are connected to the internet, go ahead and sync your system clock using NTP:
`timedatectl set-ntp true`

## Partitions
{{< notice warning >}}
Make sure you are using the correct drive! Always verify so you don't accidently nuke your data.
{{< /notice >}}

We will want to verify which hard drive we are going to install Arch to. The easiest way to do this is to use `lsblk` or `fdisk -l`. Personally, `lsblk` is prettier and I much prefer it :) 

```
$ lsblk
NAME                                MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINTS
sda                                   8:0    0   3.6T  0 disk  
└─sda1                                8:1    0   3.6T  0 part  
sdb                                   8:16   0   4.5T  0 disk  
└─sdb1                                8:17   0   4.5T  0 part  
sdc                                   8:18   0   8.5T  0 disk  
└─sdc1                                8:19   0   8.5T  0 part  

nvme1n1                             259:0    0 931.5G  0 disk  
├─nvme1n1p1                         259:4    0   100M  0 part  
├─nvme1n1p2                         259:5    0    16M  0 part  
├─nvme1n1p3                         259:6    0 930.8G  0 part  
└─nvme1n1p4                         259:7    0   593M  0 part

nvme0n1                             259:1    0 931.5G  0 disk  
├─nvme0n1p1                         259:4    0   100M  0 part  
├─nvme0n1p2                         259:5    0    16M  0 part  
├─nvme0n1p3                         259:6    0 930.8G  0 part  
└─nvme0n1p4                         259:7    0   593M  0 part  
```

My system has multiple hard drives. In this case, I am going to install Arch on one of the 1TB NVME SSDs, `nvme1n1`. 

Now to create the system partitions, lets start with the EFI boot partition:
```
$ gdisk /dev/nvme1n1
Command (? for help): o
Command (? for help): n
Default partition number.
Default first sector.
Last sector: +550M
Partition type: EF00
```

For the root partition, accept all of the default values:
```
Command (? for help): n
Default partition number.
Default first sector.
Default last sector.
Default partition type.
```

With our new paritions made, it si time to write the changes!
```
Command (? for help): w
```

## LUKS
{{< notice tip >}}
Consider your threat model when creating your passphrase.
{{< /notice >}}
Our root file system will be encrypted, so we will need a strong passphrase. Depending on your threat model, you may want to be careful about selecting a secure passphrase; also be mindful when it comes to storage of this passphrase. No sticky notes!

When creating the LUKS volume, make sure you are selecting your root parition, and not the EFI boot partition. In my case, my second parition (the root parition) is `nvme1n1p2`.
```
$ cryptsetup luksFormat /dev/nvme1n1p2
```

Once that finishes, you are going to want to open the newly created LUKS volume. You will need to name the volume when you open it, I typically just call it "luks", though you can pick whatever you want. Just make sure you stay consistant!
```
$ cryptsetup open /dev/nvme1n1p2 luks
```

## Format EFI and Root Partitions
Simple and easy, format the EFI boot parition with FAT32:

`$ mkfs.vfat -F32 /dev/nvme1n1p1` 

And now the root partition, with BTRFS:

`$ mkfs.btrfs /dev/mapper/luks`

## Create and Mount BTRFS Subvolumes
In order to take advantage of BTRFS snapshots, we will need to make subvolumes for root and home. Root being the system files that live in `/` and home being our user data in `/home/`.
```
$ mount /dev/mapper/luks /mnt
$ btrfs sub create /mnt/@
$ btrfs sub create /mnt/@home
$ umount /mnt
```

Now we are going to mount the root subvolume, make mount directories for the home subvolume and the EFI boot partition, then mount the boot partition:
```
$ mount -o noatime,nodiratime,compress=zstd:1,space_cache=v2,ssd,subvol=@ /dev/mapper/luks /mnt
$ mkdir -p /mnt/{boot,home}
$ mount -o noatime,nodiratime,compress=zstd:1,space_cache=v2,ssd,subvol=@home /dev/mapper/luks /mnt/home
$ mount /dev/nvme1n1p1 /mnt/boot
```

If you are wanting to use space cache (`space_cache=v1`) instead of free space tree (`space_cache=v2`) , check [this out](https://wiki.tnonline.net/w/Btrfs/Space_Cache) for more info.

## Install System Packages
With the prep work out of the way, it is time to install our base system packages so we can `chroot` into the system :3
{{< notice info >}}
If you are using an AMD CPU, you will want amd-ucode and not intel-ucode. Also, you may want to use linux instead of linux-zen for your kernel.
{{< /notice >}}
```
$ pacstrap /mnt linux-zen linux-zen-headers linux-firmware base base-devel btrfs-progs intel-ucode efibootmgr networkmanager dialog wpa_supplicant bluez bluez-utils mtools dosfstools git xdg-utils xdg-user-dirs alsa-utils pipewire pipewire-alsa pipewire-pulse apparmor sbctl vim zsh 
```

Feel free to change this list to fit your needs. You can always install more packages later. Also, feel free to replace `vim` with your editor of choice :)

After the `pacstrap` command finishes, you will want to generate your `/etc/fstab` file. This handy command will generate this file based on your current mount points. Cool, right?
```
$ genfstab -U /mnt >> /mnt/etc/fstab
```

Now for the exciting part, lets `chroot` into our new linux install so we can do some additional configuration!

## Chroot and System Configuration
To `chroot` into the new linux install you will run the following command:
```
$ arch-chroot /mnt/
```

You will now be presented with a root user prompt. 

Go ahead and enable the following services, so we don't forget!
```
$ systemctl enable NetworkManager apparmor bluetooth
```

### User Accounts
First things first, we need to set a root password. Make sure you pick something secure. Again, consider your threat model!
```
$ passwd
```


We will also want to create our user account. I am adding the user to the `wheel` group, as in a moment we will be granting this user group access to use the `sudo` command for privledge escalation. Also, when setting the password, again I say, consider your threat model :3
```
$ useradd -mG wheel <username>
$ passed <username>
```

To add the `wheel` group to the suderos file we will want uncomment the following line from the default config (or add it if it isn't there):
```
%wheel ALL=(ALL) ALL
```

This will allow members of the `wheel` group to execute any command as root.

### Hostname and Locale Generation
Your system's hostname can be whatever you want, all you need to do is put it in the `/etc/hostname` file.
```
$ echo <hostname> > /etc/hostname
```

We will also want to define this hostname in our `/etc/hosts/` file, as follows:
```
127.0.0.1 localhost
::1 localhost
127.0.1.1 <hostname>.localdomain <hostname>
```

### Locale Generation and Timezone
Another important thing we need to do is define our language locale. You will want to open the `/etc/locale.gen` file in your editor of choice and uncomment the lines that are relevant to your language. For me, I went with U.S English, with UTF-8 encoding.
```
en_US.UTF-8 UTF-8
```

Now you will need to set the locale and generate:
```
$ echo en_US.UTF-8 UTF-8 > /etc/locale.conf
$ locale-gen
```

To set the timezone, you will need to figure out the correct timezone string based on your region. An easy way to do that is to use `grep`:
```
$ timedatectl list-timezones | grep <region>
```

For me, I picked `America/Los_Angeles`. Now to set the timezone you picked, and sync it to your system clock:
```
$ ln -sf /usr/share/zoneinfo/America/Los_Angeles /etc/localtime
$ hwclock --systohc
```

You can run `date` to verify the current system time.

### initramfs Configuration and Generation
The initramfs configuration is super important (not to mention super neat). I would reccomend that you take some time to familarize yourself with [`mkinitcpio`](https://wiki.archlinux.org/title/mkinitcpio), which is the tool used to create the initial ramdisk (initramfs).

To start, we will need to modify `/etc/mkinitcpio.conf` to make changes to the `MODULES` and `HOOKS` sections. Go ahead and open that file in your editor of choice and set those lines to the following:
```
MODULES=(btrfs)
HOOKS=(base udev systemd autodetect keyboard modconf block sd-encrypt filesystems)
```

Once that is done, it is time to generate our new initramfs:
```
$ mkinitcpio -P
```

If you notice some warning messages about missing firmware, check the [Additional Firmware](#additional-firmware) section towards the end of this guide. It is okay to ignore for the moment and likely won't cause and problems. Though if you are concerned, go ahead and check out that section then regenerate your initramfs after you install any additional firmware packages.

# Bootloader 
If you don't want to use systemd-boot, follow insturctions for your bootloader of choice instead. Though if you are cool with systemd-boot, go ahead and install it with the following command:
```
$ bootctl --path=/boot install
```

Now to make our Arch boot entry. You are going to see the UUID of your root partition for this, so to make things a bit easier, I would reccomend adding the UUID to the boot entry file, twice:
{{< notice warning >}}
Make sure you are using the correct drive and are specifing your root partition.
{{< /notice >}}
```
$ echo `blkid -S UUID -o value /dev/nvme1n1p2` >> /boot/loader/entries/arch.conf
$ echo `blkid -S UUID -o value /dev/nvme1n1p2` >> /boot/loader/entries/arch.conf
```

Go ahead and open `/boot/loader/entries/arch.conf` in your editor of choice and add the following:
{{< notice warning >}}
If you did not use linux-zen, make sure you specify that in this file. Also, if you used amd-ucode, use that for your initrd instead.
{{< /notice >}}
```
title Arch Linux
linux /vmlinuz-linux-zen
initrd /intel-ucode.img
initrd /initramfs-linux-zen.img
options nvidia-drm.modeset=1 apparmor=1 security=apparmor rd.luks.name=<UUID>=luks root=/dev/mapper/luks rootflags=subvol=@
rd.luks.options=<UUID>=tpm2-device=auto,discard rw quiet lsm=lockdown,yama,apparmor,bpf

```

The `nvidia-drm.modeset=1` kernel option will be needed for our NVIDIA driver, which we will install [later](#nvidia). If you won't be using the NVIDIA driver, go ahead and remove this.

Also, if you are not going to use TPM 2.0 for LUKS, go ahead and remove the `tpm2-device` LUKS option.

# GNOME 42 (Or whatever you want)
blah blah here

# Apparmor Profiles
blah blah here

# Timeshift
blah blah here

# TPM 2.0 Enrollment
blah blah here

# Secure Boot
blah blah here

# NVIDIA
blah blah here

# Additional Firmware
The `linux-firmware` package covers most firmware for common hardware devices, though I had to install some additional packages to ensure that my system was stable.
```
$ pacman -S linux-firmware-qlogic aic94xx-firmware wd719x-firmware
```


`<3 m0x`