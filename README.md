# GPD MicroPC Dual Boot - Arch / Windows 10 Install Guide
_NOTE: This guide is not yet complete and should only be used as a light reference right now._
_Pull requests are greatly appreciated!_

This guide overviews how to do a full wipe of the GPD MicroPC, followed by a dual boot of
Arch Linux and Windows 10.

# Table of Contents

- [Prerequisites](#prerequisites)
- [Step 1: Use Arch USB to reformat drive](#step-1-use-arch-usb-to-reformat-drive)
- [Step 2: Install Windows 10](#step-2-install-windows-10)
- [Step 3: Let's Install Arch](#step-3-lets-install-arch)
    - [Configure LUKS + LVM2 partitions on `sda2`](#configure-luks--lvm2-partitions-on-sda2)
    - [Wirelessly connect to the internet](#wirelessly-connect-to-the-internet)
    - [Mount, pacstrap and prepare for arch-chroot](#mount-pacstrap-and-prepare-for-arch-chroot)
    - [arch-chroot](#arch-chroot)
- [Step 4: Make "Linux Boot Manager" the default in the BIOS](#step-4-make-linux-boot-manager-the-default-in-the-bios)
- [Step 5: From Text to X](#step-4-from-text-to-x)
    - [Add some repos to pacman](#add-some-repos-to-pacman)
    - [Build and install yay package manager tool in order to install AUR packages](#build-and-install-yay-package-manager-tool-in-order-to-install-aur-packages)
    - [Additional timezone setup](#additional-timezone-setup)
    - [thermald](#thermald)
    - [NetworkManager](#networkmanager)
    - [Sound](#sound)
    - [Bluetooth](#bluetooth)
    - [TLP](#tlp)
    - [Xorg](#xorg)
        - [Install Xorg packages](#install-xorg-packages)
        - [Create MicroPC Xorg configs](#create-micropc-xorg-configs)
        - [Install Intel video drivers](#install-intel-video-drivers)
    - [XFCE4](#xfce4)
    - [Create ~/.xinitrc](#create-xinitrc)
    - [Start X](#start-x)



# Prerequisites

  - [GPD MicroPC](https://www.indiegogo.com/projects/gpd-micropc-6-inch-handheld-industry-laptop)
  - [Arch ISO](https://www.archlinux.org/download) written to USB stick
  - [Microsoft Windows 10 installer](https://www.microsoft.com/en-us/software-download/windows10ISO) on USB stick
  - an extra USB drive for the Arch 5.2 packages (or possibly add them to your Arch USB stick)
      - [linux-mainline-5.2rc5-1-x86_64.pkg.tar.gz](https://github.com/joshskidmore/gpd-micropc-arch-guide/raw/master/linux-mainline-packages/linux-mainline-5.2rc5-1-x86_64.pkg.tar.gz)
      - [linux-mainline-docs-5.2rc5-1-x86_64.pkg.tar.gz](https://github.com/joshskidmore/gpd-micropc-arch-guide/raw/master/linux-mainline-packages/linux-mainline-docs-5.2rc5-1-x86_64.pkg.tar.gz)
      - [linux-mainline-headers-5.2rc5-1-x86_64.pkg.tar.gz](https://github.com/joshskidmore/gpd-micropc-arch-guide/raw/master/linux-mainline-packages/linux-mainline-headers-5.2rc5-1-x86_64.pkg.tar.gz)
  - ability to understand basic linux commands
  - patience (and eyesight) to spend a good bit of time looking at small, sideways text


# Step 1:  Use Arch USB to reformat drive
  1. Download standard Arch linux ISO
  2. Write ISO to USB drive
  3. Boot MicroPC and tap `F7`
  4. _IMMEDIATELY_ hit `e` which edits the kernel options and add ` nomodeset=1` to the end of the options and hit enter.
  4. In the options, select the USB device containing Arch
  5. Reformat `sda` disk

    Essentially, this is what we're going for:

        NAME              FSTYPE        SIZE    NOTES                       MOUNTPOINT
        sda
        ├─sda1            vfat          512M    efi                         /boot
        ├─sda2            luks+lvm2     60G     linux                       /
        ├─sda[3-5]                      ~58G    windows

    It's critical that we make the first partition (sda1) the EFI partition. For now, we're going
    to placehold our linux partition because the Windows 10 installer will create a bunch of bullshit
    partitions. For this guide, by placing the linux partition directly after the EFI partition, we
    can let Windows create its dumpster fire of partitions, and it won't effect us.

        gdisk /dev/sda
        # o ↵ to create a new empty GUID partition table (GPT)
        # y ↵ to confirm

        # n ↵ add a new partition
        # ↵ to select default partition number of 1
        # ↵ to select default start at first sector
        # +512M ↵ make that size partition for booting
        # ef00 ↵ EFI partition type

        # n ↵ add a new partition
        # ↵ to select default partition number of 2
        # ↵ to select default start at first sector
        # +60G ↵ allocate whatever size wanted for linux

        # we're intentionally leaving space for Windows 10 partitions

        # p ↵ if you want to check the partition layout
        # w ↵ to write changes to disk
        # y ↵ to confirm

  5. Format the EFI partition to vfat

        `mkfs.vfat /dev/sda2`


# Step 2: Install Windows 10

  1. Create a Windows 10 install USB
  2. Boot MicroPC and tap `F7`
  3. In the options, select the USB device containing the Windows 10 installer
  4. Install Windows 10. The only important thing is that in the disk partitioner, let Windows use the unused space at the end of
  the drive. The Windows installer will automatically place its EFI files into the EFI partition (`sda1`). The installer will
  warn you that it has to create additional partitions which is normal.

Windows will take over the MicroPC and act as if it's rolling solo and that's completely fine. It will reboot a couple times. It's
also normal that Windows 10 is rotated right until Windows drivers are installed and rotation is configured.


# Step 3: Let's Install Arch
Remove the Windows 10 install USB stick and, just like step one, insert the Arch install USB and boot into Arch.

## Configure LUKS + LVM2 partitions on `sda2`

    # Encrypt /dev/sda2
    # This step will ask for a password which will be used to unlock the partition on boot. Make sure you know the password.
    cryptsetup luksFormat -v -s 512 -h sha512 /dev/sda2

    # Open/mount encrypted disk
    # Upon unlocking, this will mount the unlocked disk to /dev/mapper/luks.
    cryptsetup luksOpen /dev/sda2 luks

    # Create LVM2 Physical Volume (PV) on /dev/mapper/luks
    pvcreate /dev/mapper/luks

    # Create LVM2 Volume Group (VG) on /dev/mapper/luks
    vgcreate rootvg /dev/mapper/luks

    # Create LVM2 Logical Volume (LV) on rootvg
    # This will create an LVM logical volume at /dev/mapper/rootvg-root.
    lvcreate -n root -l 100%FREE rootvg

    # Finally, format the logical volume to ext4
    mkfs.ext4 /dev/mapper/rootvg-root

## Wirelessly connect to the internet
Internet is needed to download packages. `wifi-menu` is a basic curses command which will temporarily select and configure a
wifi network.

    wifi-menu

## Mount, pacstrap and prepare for arch-chroot

    # Mount the ext4-formatted root LV
    # Note: /mnt becomes your actual Arch install.
    mount /dev/mapper/rootvg-root /mnt

    # Make the boot directory to mount the EFI (sda1) partition
    mkdir /mnt/boot

    # Mount the EFI parition
    mount /dev/sda1 /mnt/boot

    # Pacstrap the /mnt directory with utilities needed for arch-chroot
    pacstrap /mnt base base-devel dialog openssl-1.0 bash-completion git intel-ucode wpa_supplicant

    # Generate your fstab
    genfstab -pU /mnt >> /mnt/etc/fstab


## arch-chroot
`arch-chroot` drops you into your Arch filesystem in order to handle additional install tasks.

    arch-chroot /mnt /bin/bash

Awesome! You're now in your to-be Arch filesystem. Let's install some basic shit.

First thing, we need to install mainline kernel packages since the standard packages don't yet support the MicroPC.

    # the next steps assume you copied the three packages to an external usb stick labeled sdb
    mount /dev/sdb1 /mnt
    cd /mnt; pacman -U *.gz

Set your hostname (yeah, this is the most difficult part .. at least for me)

    echo MYHOSTNAME > /etc/hostname

By now, you're probably sick of looking at tiny text, sideways. Let's fix the font size issue (on
reboot) by creating the file `/etc/vconsole.conf` and adding:

    FONT=latarcyrheb-sun32

Because we're using disk encryption and LVM2, we need to add and reorder mkinitcpio's hooks. Edit
`/etc/mkinitcpio.conf` and completely replace the line beginning with `HOOKS` with:

    HOOKS=(base systemd autodetect keyboard modconf block sd-encrypt sd-lvm2 fsck filesystems)

With new HOOKS in-tow, regenerate your ramdisk and kernel using mkinitcpio:

    mkinitcpio -P

We'll be using `systemd-boot` as a better grub replacement. First, install systemd-booti using the
following command. The command will create a new EFI entry named `Linux Boot Manager` which will sit
alongside Windows' `Windows Boot Manager`.

    bootctl install

Windows will be included as a choice on boot, and any additional entries must be created in the
directory `/boot/loader/entries/`. We're going to create a default entry for `arch`.

First, we need the block or UUID of the sda2 LUKS partition for our `arch` entry. Since
you're working sideways with little ass text, lets dump that ID right into the empty entry file:

    blkid | grep sda2 | cut -d \" -f 2 > /boot/loader/entries/arch.conf

Next, add the following to the `/boot/loader/entries/arch.conf` entry. Replace `[UUID]` with the
UUID you dumped into the file. Remove the brackets.

    title Arch Linux
    linux /vmlinuz-linux
    initrd /intel-ucode.img
    initrd /initramfs-linux.img
    options rd.luks.name=[UUID]=luks root=/dev/mapper/rootvg-root rw fbcon=rotate:1

The above systemd-boot entry will eventually be default, but for now, we need to duplicate that config
to a mainline config at `/boot/loader/entries/arch-mainline.conf`:

    title Arch Linux Mainline
    linux /vmlinuz-linux-mainline
    initrd /intel-ucode.img
    initrd /initramfs-linux-mainline.img
    options rd.luks.name=[UUID]=luks root=/dev/mapper/rootvg-root rw fbcon=rotate:1

Of note is the kernel command line option `fbcon=rotate:1`. This will rotate any non-X content to the
right (after reboot).

We also need to edit (or possibly create) the file `/boot/loader/loader.conf`. This file handles
systemd-boot's general config options. Most of the following options should be self explanitory, but
one that might not be is `console-mode 2`. This option will force larger text during the handoff from
the BIOS to the kernel. The file should contain:

    default arch-mainline
    auto-firmware no
    timeout 3

Specify a timezone:

    ln -sf /usr/share/zoneinfo/US/Eastern /etc/localtime

Create `/etc/locale.conf` with the following:

    LANG=en_US.UTF-8
    LANGUAGE=en_US
    LC_ALL=C

Edit the file `/etc/locale.gen` and uncomment the proper locale line:

    en_US.UTF-8 UTF-8

Then, generate the locale using the following command:

    locale-gen

Set a root password:

    passwd

Create your user and set a password:

    useradd -m -g users -G wheel,storage,power -s /bin/bash [USERNAME]
    passwd [USERNAME]

If you wish to run sudo'd commands without entering a password, run `export EDITOR=vi; visudo` and
uncomment the line:

    %wheel ALL=(ALL) NOPASSWD: ALL

Ok, we're done in `arch-chroot`! Let's unmount and restart.

    # Exit arch-chroot and back into the default shell
    exit

    # Recursively unmount /mnt
    umount -R /mnt

    # Remove USB stick and reboot
    reboot

# Step 4: Make "Linux Boot Manager" the default in the BIOS
This is a pretty obvious step, but the option is a bit hidden in the MicroPC's options. Restart the
MicroPC and hit `ESC` to jump into the BIOS. From there, tab over to `UEFI Hard Disk Drive BBS Priorities`
and select `Linux Boot Manager` as the first entry. Make sure to save and apply changes.

# Step 5: From Text to X
Upon reboot, immediately after the BIOS, you should see `systemd-boot` prompt you with two
options: `arch` and `Windows Boot Manager`. All text should be sized decently and rotated
correctly. If this is not the case, something has been missed.

You should also be prompted to enter the LUKS partition decryption password that you created
earlier. If entered correctly, the system will continue to load and provide a terminal login.

Login with the user you created, *not* `root`.

## Wirelessly connect to the internet (again)
You know, because reboot.

    wifi-menu

## Add some repos to pacman

Edit `/etc/pacman.conf` and uncomment these lines:

    [multilib]
    Include = /etc/pacman.d/mirrorlists

In `/etc/pacman.conf`, append these lines to the bottom

    [archlinuxfr]
    SigLevel = Never
    Server = http://repo.archlinux.fr/$arch

## Sync pacman to retrieve latest package information

    sudo pacman -Sy

## Build and install `yay` package manager tool in order to install AUR packages
If you prefer another AUR/pacman tool, feel free to replace this step and `yay` with that tool.

    # Build yay package and install
    git clone https://aur.archlinux.org/yay.git
    cd yay
    makepkg -si

    # After install, remove the yay build folder
    cd ..; rm -rf yay

    # Sync yay
    yay -Sy

## Additional timezone setup
`tzselect` is an interactive utility. Upon running the following command, choose your region and timezone.

    sudo tzselect

## thermald
`thermald` protects your MicroPC from overheating.

    # Install
    yay -S thermald

    # Enable + start
    sudo systemctl enable thermald.service
    sudo systemctl start thermald.service

## NetworkManager

    # Install
    yay -S networkmanager network-manager-applet nm-connection-editor

    # Enable + start
    sudo systemctl enable NetworkManager
    sudo systemctl start NetworkManager

`nmtui` is a command-line utility which allows selection of a network. Note: Because `wifi-menu` was used,
and a wifi network has already been selected, `nmtui` might fail or error. Upon next reboot,
`nmtui` should be available (or just use the applet in Xorg).

    nmtui

## Sound

    # Install pulseaudio packages
    yay -S pulseaudio pulseaudio-alsa pulseaudio-bluetooth pulseaudio-ctl

    # (optional) Install nice traybar utils to work with audio
    yay -S pasystray-gtk3-standalone pavucontrol

## Bluetooth

    # Install
    yay -S bluez bluez-utils bluez-tools

    # Enable + start
    sudo systemctl enable bluetooth
    sudo systemctl start bluetooth

    # (optional) Install nice traybar utils
    yay -S blueman blueberry

## TLP
TLP helps reduce power by setting some sane power defaults.

    # Install
    yay -S tlp

    # Enable + start
    sudo systemctl enable tlp
    sudo systemctl start tlp

## Xorg

### Install Xorg packages

    yay -S xorg-server xorg-xev xorg-xinit xorg-xkill xorg-xmodmap xorg-xprop xorg-xrandr xorg-xrdb xorg-xset xinit-xsession

### Create MicroPC Xorg configs

Create Intel xorg config at `/etc/X11/xorg.conf.d/20-intel.conf` with the following:

    Section "Device"
      Identifier    "Intel Graphics"
      Driver        "intel"
      Option        "AccelMethod"            "sna"
      Option        "TearFree"               "true"
      Option        "DRI"                    "3"
    EndSection

Create display config at `/etc/X11/xorg.conf.d/30-display.conf` with the following:

    Section "Monitor"
      Identifier    "DSI"
      Option        "Rotate"                 "right"
    EndSection

### Install Intel video drivers

    yay -S xf86-video-intel


## XFCE4

    yay -S xfce4 xfce4-goodies
    # hit enter to install all the goodies

## Create `~/.xinitrc`

    # Simple hack to allow mouse scrolling when right click button is held down
    xinput --set-prop pointer:"AMR-4630-XXX-0- 0-1023 USB KEYBOARD Mouse" "libinput Middle Emulation Enabled" 1
    xinput --set-prop pointer:"AMR-4630-XXX-0- 0-1023 USB KEYBOARD Mouse" "libinput Button Scrolling Button" 3
    xinput --set-prop pointer:"AMR-4630-XXX-0- 0-1023 USB KEYBOARD Mouse" "libinput Scroll Method Enabled" 0 0 1

    # Start XFCE4
    exec startxfce4

## Start X

    startx
