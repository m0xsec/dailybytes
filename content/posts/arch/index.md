---
title: "I use Arch, btw"
date: 2022-06-03T22:30:00-07:00
draft: false
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

I've been wanting to install Arch on my primary workstation for a while though I was pretty overwhelmed with figuring out what sort of configuration I wanted. I now have something that I like and wanted to outline what I did. This is mostly for my own reference and documentation, though please feel free to use this and adapt it to your own needs!

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

Before you get started, make sure that you have UEFI Mode enabled and Secure Boot disabled on your system. We will enable [Secure Boot](#secure-boot) later in this guide.

## WiFi and Syncing System Clock
You will probably want to connect to WiFi, unless you are directly plugged in or are setting up an air-gapped system.

[`iwctl`](https://wiki.archlinux.org/title/Iwd#iwctl) is probably the easiest way to do this.

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

Now to create the system partitions, let's start with the EFI boot partition:
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

With our new partitions made, it is time to write the changes!
```
Command (? for help): w
```

## LUKS
{{< notice tip >}}
Consider your threat model when creating your passphrase.
{{< /notice >}}
Our root file system will be encrypted, so we will need a strong passphrase. Depending on your threat model, you may want to be careful about selecting a secure passphrase; also be mindful when it comes to storage of this passphrase. No sticky notes!

When creating the LUKS volume, make sure you are selecting your root partition, and not the EFI boot partition. In my case, the second partition (the root partition) is `nvme1n1p2`.
```
$ cryptsetup luksFormat /dev/nvme1n1p2
```

Once that finishes, you are going to want to open the newly created LUKS volume. You will need to name the volume when you open it, I typically just call it "luks", though you can pick whatever you want. Just make sure you stay consistent!
```
$ cryptsetup open /dev/nvme1n1p2 luks
```

## Format EFI and Root Partitions
Simple and easy, format the EFI boot partition with FAT32:

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

Now for the exciting part, let's `chroot` into our new linux install so we can do some additional configuration!

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


We will also want to create our user account. I am adding the user to the `wheel` group, as in a moment we will be granting this user group access to use the `sudo` command for privilege escalation. Also, when setting the password, again I say, consider your threat model :3
```
$ useradd -mG wheel <username>
$ passed <username>
```

To add the `wheel` group to the suderos file we will want uncomment the following line from the default config (or add it if it isn't there):
```
%wheel ALL=(ALL) ALL
```

This will allow members of the `wheel` group to execute any command as root.

### System Hostname and Hosts Configuration
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

{{< notice note>}}
You may need to also select your locale in GNOME's settings later.
{{< /notice >}}
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
The initramfs configuration is super important (not to mention super neat). I would recommend that you take some time to familiarize yourself with [`mkinitcpio`](https://wiki.archlinux.org/title/mkinitcpio), which is the tool used to create the initial ramdisk (initramfs).

To start, we will need to modify `/etc/mkinitcpio.conf` to make changes to the `MODULES` and `HOOKS` sections. Go ahead and open that file in your editor of choice and set those lines to the following:
```
MODULES=(btrfs)
HOOKS=(base udev systemd autodetect keyboard modconf block sd-encrypt filesystems)
```

Once that is done, it is time to generate our new initramfs:
```
$ mkinitcpio -P
```

If you notice some warning messages about missing firmware, check the [Additional Firmware](#additional-firmware) section towards the end of this guide. It is okay to ignore for the moment and likely won't cause any problems. Though if you are concerned, go ahead and check out that section then regenerate your initramfs after you install any additional firmware packages.

# Bootloader 
If you don't want to use systemd-boot, follow instructions for your bootloader of choice instead. Though if you are cool with systemd-boot, go ahead and install it with the following command:
```
$ bootctl --path=/boot install
```

Now to make our Arch boot entry. You are going to need the UUID of your root partition for this, so to make things a bit easier, I would recommend adding the UUID to the boot entry file, twice:
{{< notice warning >}}
Make sure you are using the correct drive and are specifying your root partition.
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
rd.luks.options=<UUID>=discard rw quiet lsm=lockdown,yama,apparmor,bpf

```

The `nvidia-drm.modeset=1` kernel option will be needed for our NVIDIA driver, which we will install [later](#nvidia). If you won't be using the NVIDIA driver, go ahead and remove this.

# GNOME 42
Installing the latest version of GNOME is pretty straight forward. At the time of writing, GNOME 42 is the latest and greatest! You can install it like this:
```
$ pacman -S gnome gdm gnome-tweaks firefox
```

You may want to include `xorg` if you are not planning on using Wayland and prefer to stick with X11.

Once that is done installing, you will want to enable to GNOME Desktop Manager service:
```
$ systemctl enable gdm
```

If you are looking to install a different Window Manager / Desktop Environment, check the [Arch Wiki](https://wiki.archlinux.org/title/desktop_environment) for guidance on how to do that.

# NVIDIA
To install the NVIDIA proprietary drivers, and to get them to play nice with Wayland and GDM, we need to do a few things.

First, install the following:
```
$ pacman -S nvidia-dkms nvidia-utils
```

You may just want `nvidia` instead of `nvidia-dkms`, depending on your setup.

The next thing you'll want to do is prevent the Nouveau video driver from loading. Open `/etc/modprobe.d/prevent-nouveau.conf` in your editor of choice, and add:
```
blacklist nouveau
```

As of writing this, GDM will force X11 and not offer Wayland as an option if it detects that an NVIDIA graphics card is present. To get around this, we just need to disable this rule by doing the following:
```
$ ln -s /dev/null /etc/udev/rules.d/61-gdm.conf
```

Now we will need to set some custom NVIDIA kernel flags and enable NVIDIA's power management services in order for the latest drivers to work with Wayland. This is a recent issue, as I haven't had to do this in the past.

Go ahead and open up `/etc/modprobe.d/nvidia-power-management.conf` in your editor of choice, and add the following:
```
options nvidia NVreg_EnableMSI=0 NVreg_PreserveVideoMemoryAllocations=1 NVreg_TemporaryFilePath=/tmp-nvidia
```

You can remove the `NVreg_EnableMSI=0` option if you'd like. I needed to do that on my end to fix an issue where my display wouldn't wake up after my computer went to sleep.

Now you just need to enable the following NVIDIA services:
```
$ systemctl enable nvidia-suspend
$ systemctl enable nvidia-hibernate
```

Since we added some new configuration files to our `/etc/modprobe.d/` folder, we will need to regenerate our initramfs. Earlier we included the `modconf` hook, which will tell `mkinitcpio` to include all configuration files in that directory.
```
$ mkinitcpio -P
```

Once the initframfs generates, it is time to boot into our new install for real! Go ahead and exit chroot, unmount, and reboot:
```
$ exit
$ umount -a
$ reboot
```

# Apparmor Profiles
Assuming everything went well, you should now be booted into your fresh Arch install and logged into your local user account :3

Now that we're here, let's verify that Apparmor is working.
```
$ sudo aa-enabled
Yes
```

The output of `Yes` will indicate that Apparmor is enabled. To get the current status and list of profiles loaded, run the following:
```
$ sudo aa-status
apparmor module is loaded.
50 profiles are loaded.
50 profiles are in enforce mode.
   ...
	...
0 profiles are in complain mode.
0 profiles are in kill mode.
0 profiles are in unconfined mode.
0 processes have profiles defined.
0 processes are in enforce mode.
0 processes are in complain mode.
0 processes are unconfined but have a profile defined.
0 processes are in mixed mode.
0 processes are in kill mode.

```

If you aren't getting output similar to what is outlined above, head on over to the [Arch Wiki](https://wiki.archlinux.org/title/AppArmor) for guidance.

# Timeshift
Timeshift will be the application we use to create and manage BTRFS snapshots of our root subvolume. This application isn't yet available in Arch's official repos, though like anything else, we can find it in the Arch User Repository (AUR).

An easy and convenient way to install and manage AUR packages on your system is to use an AUR helper (or write your own ;3). There are several helpers to pick from, though today I will be using [`paru`](https://github.com/morganamilo/paru). One of the main reasons I like `paru` is that it allows you to review and edit the `PKGBUILD`, in-line, before it runs. If you don't care about that, or don't enjoy using `vim` to edit files, I'd suggest checking out `yay`.

```
$ git clone https://aur.archlinux.org/packages/paru/
$ cd paru
$ makepkg -si
```

Now that you have `paru`, you can install Timeshift like this:
```
$ paru -S timeshift
```

{{< notice tip >}}
Think about your disaster and data recovery plan and test it often!
{{< /notice >}}

After you install Timeshift, go ahead and open it up and go through the settings / setup process. Make sure you select the `BTRFS` snapshot type. The snapshot location is up to you, though the most convenient is probably to have it stored on the root subvolume. 

I am only using snapshots on my root subvolume, though you can enable snapshots on your home subvolume as well. I am using Timeshift for my system level stuff and `duplicity` for my user data.

# TPM 2.0 Enrollment
{{< notice warning >}}
Depending on your situation, doing this might be a bad idea.
{{< /notice >}}
I liked the idea of having my TPM unlock the LUKS volume on boot to speed up the boot process, creating a more seamless experience. Depending on our situation, enrolling your TPM as one of your LUKS key slots might be a bad idea. As always, consider your threat model.

Verify that your TPM is detected, and that it is version 2.0:
```
$ cat /sys/class/tpm/tpm0/tpm_version_major 
2
```

If that path doesn't exist, you might be able to check `/sys/class/tpm/tpm0/device/description`.

At this point, `systemd-cryptenroll` should already be able to see our TPM, since we included the `systemd` and `sd-encrypt` hooks when generating our initramfs. To verify this you can run:
```
$ systemd-cryptenroll --tpm2-device=list

PATH        DEVICE DRIVER 
/dev/tpmrm0 00:00  tpm_tis

```

You should see your TPM listed in the output, similar to what is shown above. Assuming that looks good, you will now want to enroll your TPM to one of your key slots on your LUKS volume, in my case that is partition 2:
```
$ systemd-cryptenroll --tpm2-device=auto --tpm2-pcrs=0,7 /dev/nvme1n1p2
```

After than, you will need to add a new `rd.luks.options` option to your boot entry for Arch. Open `/boot/loader/entries/arch.conf` in your editor of choice, and add `tpm2-device=auto`. Your boot entry should now look something like this:
```
title Arch Linux
linux /vmlinuz-linux-zen
initrd /intel-ucode.img
initrd /initramfs-linux-zen.img
options nvidia-drm.modeset=1 apparmor=1 security=apparmor rd.luks.name=<UUID>=luks root=/dev/mapper/luks rootflags=subvol=@
rd.luks.options=<UUID>=tpm2-device=auto,discard rw quiet lsm=lockdown,yama,apparmor,bpf
```

If you reboot, you should see your system go straight to the login screen. If the TPM is tampered with or removed, you will be prompted to enter your passphrase like before.

# Secure Boot
Before you enable Secure Boot, you will need to go into your BIOS and enable "Setup Mode". This will allow key enrollment. Depending on your BIOS, you may have to first delete all of your pre-existing keys before Setup Mode can be toggled. Refer to your motherboard's documentation if you run into issues. 

You may also wish to keep vendor keys enrolled so you can continue to boot into Windows and use other boot utilities. If you don't know what you are doing, chances are you will want to keep these vendor keys.

Once your BIOS is in Secure Boot Setup Mode, go ahead and boot back into Arch, and run the following command to verify:
```
$ sbctl status
Installed:	✓ sbctl is installed
Setup Mode:	✗ Enabled
Secure Boot:	✗ Disabled
```

You should see output similar to what is shown above. The important part is to make sure that "Setup Mode" says enabled and that "Secure Boot" says disabled.

Now, with that out of the way, you can create secure boot keys and enroll them, then sign your kernel files. To do this, I used `sbctl`. They have some excellent documentation on their [Github repo](https://github.com/Foxboron/sbctl#usage).

To create keys and enroll them, do the following (note, this will ensure that Microsoft's keys are also enrolled. Remove `--microsoft` if you don't want that):
```
$ sbctl create-keys
$ sbctl enroll-keys --microsoft
```

Now we need to sign our EFI boot files. To figure out which files need signing, we can run `sbctl verify`:
```
$ sbctl verify
Verifying file database and EFI images in /efi...
✘ /boot/EFI/BOOT/BOOTX64.EFI is not signed
✘ /boot/EFI/systemd/systemd-bootx64.efi  is not signed
✘ /boot/vmlinuz-linux is not signed
✘ /boot/vmlinuz-linux-zen is not signed
```

To sign these files, simply run:
```
$ sbctl sign -s /boot/EFI/BOOT/BOOTX64.EFI
$ sbctl sign -s /boot/EFI/systemd/systemd-bootx64.efi 
$ sbctl sign -s /boot/vmlinuz-linux
$ sbctl sign -s /boot/vmlinuz-linux-zen
```

To verify that those files were signed, run `sbctl verify` again:
```
$ sbctl verify
Verifying file database and EFI images in /efi...
✔ /boot/EFI/BOOT/BOOTX64.EFI is signed
✔ /boot/EFI/systemd/systemd-bootx64.efi  is signed
✔ /boot/vmlinuz-linux is signed
✔ /boot/vmlinuz-linux-zen is signed
```

You may also need to generate an EFI Stub, depending on your configuration. I didn't need to do this on my end in order for this to work. To learn more, check out the `sbctl` [usage guide](https://github.com/Foxboron/sbctl#generate-efi-stub).

Now you can reboot and enable Secure Boot! Once you log back in, you can verify that everything is working by running `sbctl status`:
```
$ sbctl status
Installed:	✓ sbctl is installed
Owner GUID:	<UUID>
Setup Mode:	✓ Disabled
Secure Boot:	✓ Enabled
Vendor Keys:	microsoft

```

Congrats, you now have Secure Boot working!

# Additional Firmware
The `linux-firmware` package covers most firmware for common hardware devices, though I had to install some additional packages to ensure that my system was stable. When you [generate your initframfs](#initramfs-configuration-and-generation), you may see some warning messages about firmware not being installed. Search around for the firmware that is mentioned and you will likely find a package for it in the [AUR](https://aur.archlinux.org/packages?O=0&SeB=nd&K=firmware&outdated=&SB=p&SO=d&PP=50&submit=Go).

```
$ pacman -S linux-firmware-qlogic linux-firmware-iwlwifi-git
$ paru -S upd72020x-fw aic94xx-firmware wd719x-firmware
```

At this point you're done! You can probably start enjoying your new Arch Linux install, so go have fun :3

# References
I used so, so many references during this process. The list below are the ones I found to be the most helpful.

- https://wiki.archlinux.org/title/Installation_guide
- https://wiki.archlinux.org/title/Iwd#iwctl
- https://wiki.archlinux.org/title/AppArmor
- https://wiki.archlinux.org/title/GNOME
- https://wiki.archlinux.org/title/NVIDIA
- https://wiki.archlinux.org/title/systemd-boot
- https://wiki.archlinux.org/title/Trusted_Platform_Module#Data-at-rest_encryption_with_LUKS
- https://forum.artixlinux.org/index.php/topic,1506.0.html
- https://unix.stackexchange.com/questions/610779/pop-os-systemd-boot-cant-detect-windows
- https://www.reddit.com/r/archlinux/comments/u3ydcf/after_updating_to_5173arch_gnome_wayland/
- https://www.reddit.com/r/archlinux/comments/u31p61/option_to_run_gnome_on_wayland_missing_from_gdm/
- https://www.reddit.com/r/archlinux/comments/u36x68/my_arch_fallback_to_x11_session_after_today/
- https://lemmy.eus/post/2898
- https://github.com/Foxboron/sbctl#usage
- https://odysee.com/@EF-TechMadeSimple:3/arch-linux-install-january-2021-iso-2:a
- https://wiki.tnonline.net/w/Btrfs/Space_Cache





`<3 m0x`