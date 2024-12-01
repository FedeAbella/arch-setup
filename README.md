# Arch Linux Setup from scratch

This is a guide on how to install Arch Linux from scratch, following the steps taken by me on my own install. YMMV. Read carefully and refer to the [Arch Linux Wiki](wiki.archlinux.org) if in doubt.

Most of these steps can be found in some form or another following the [Arch Linux Installation Guide](wiki.archlinux.org/title/Installation_guide) and related articles when relevant. I'll try to link all relevant information when reasonable (and when I've remembered to do so).

## Pre-installation

Follow the steps in the Installation Guide for acquiring the installation image, booting into the live usb and other config steps up to but not including disk management. This is the first deviation from the strict install guide. We'll set up the disks for **full-disk encryption**, including an encrypted `/boot` using GRUB, LVM on multiple drives, as well as a separate `home` directory for multiple users. Feel free to ignore any steps you don't need.

### Writing random data into disks

One way to securely erase disks prior to encryption is to write them full of random data. This may take a **long time** depending on the type of drive and storage capacity.

Find your drives using `lsblk`, and for each, write random data into them using

    $ dd if=/dev/urandom of=/dev/<drive_name_from_lsblk> status=progress bs=4096

### Partition the drives

Since we will be using an encrypted `root` as well as encrypted `/boot` with GRUB, this follows roughly the instructions on [Encrypted boot partition (GRUB)](wiki.archlinux.org/title/Dm-crypt/Encrypting_an_entire_system#Encrypted_boot_partition_(GRUB))

In my case, I'm working off of 4 drives: 

  - 1 nvme where I'll store: `/efi`, `root`, `swap` and an LVM to be used as extra storage.
  - 3 ssd drives where I'll create three different LVMs to be used for:
    - my personal `home` directory
    - the `home` directory of a separate user for work purposes
    - extra storage

All drives are partitioned using `gdisk`. The nvme drive is partitioned as:

  - One 1MB partition for BIOS boot (code `EF02`)
  - One 512MB partition for UEFI boot (code `EF00`)
  - The remaining space as a Linux LUKS partition where the remaining logical groups are created later

The three ssd drives all contain a single Linux LUKS partition.

### Encrypt the partitions

For all LUKS partitions, we create the encrypted containers using

    $ cryptsetup luksFormat --pbkdf pbkdf2 <path/to/partition>

when prompted, I just used the same passphrase for all partitions. We then then open the containers using

    $ cryptsetup open <path/to/partition> <mapper_name>

where for `<mapper_name>` I used something along the lines of `<partition_name>_crypt`.

### Creating the LVM groups

For all three ssds, I created a single `volume group`, and a single `logical group` inside each. In the nvme drive, I created a single volume group with 3 logical groups:

  - A 75GB group for `swap` (at time of writing I've got 32GB of ram, but this would allow me to expand to 64GB and have enough swap as recommended to allow for hibernate)
  - A 100GB group for `root`
  - A group with the remaining storage capacity

### Format the partitions

Follow the installation guide to format the EFI partition, and create the swap partition with `mkswap`. The BIOS partition is formatted as ext4. All other partitions are formated as BTRFS. Remember for LVM groups and LUKS container, the path to the partition is `/dev/mapper/<name>`.

### Mount points

The `root` lvm group is mounted to `/mnt`. One of the sdd groups is mounted to `/mnt/home/fede` which will be my personal home directory. Another ssd goes to `/mnt/home/<work>` for the work profile. The third ssd and the nvme storage group are mounted to dirs under `/mnt/home/fede` respectively.

The BIOS boot partition is not mounted, as we will be using EFI. The EFI partition is mounted to `/mnt/efi`. Finally, the swap partition is mounted using

    $ swapon /dev/mapper/<swap_name>
    $ swapon -a

**Note:** Mounting the partitions at this point makes it so that when the users are created, the `skel` files must be copied manually, since their home directories will already exist. A very minor inconvenience for the upside of handling all drive related activities at once.

## Installation

At this point, follow the installation guide and add the following packages to `pacstrap`: `amd-ucode` (in my case, I've got an AMD Ryzen CPU), `lvm2` (we'll need it to manage the LVM groups on boot), `btrfs-progs` (for handling multiple BTRFS drives), `neovim`, `man-db` (for access to `man`). Other needed packages will be installed later. Continue with the installation guide steps up intil setting the initramfs.

### Initramfs

Edit the mkinitcpio config file:

    $ neovim /etc/mkinitcpio.conf

and add to the `HOOKS` array so it looks like

    HOOKS=(base udev btrfs autodetect microcode modconf kms keyboard keymap consolefont block encrypt lvm2 filesystems fsck)

This adds the needed hooks to the kernel boot for decrypting drives and properly mounting the lvm groups. Then regenerate the initramfs:

    $ mkinitcpio -P

### Bootloader

Using GRUB as our bootloader allows for booting from an encrypted LUKS partition, and decrypting all other drives while only entering the passphrase once. First install grub:

    $ pacman -S grub efibootmgr
    $ grub-install --target=x86_64-efi --efi-directory=/efi --bootloader-id=GRUB --recheck

Then set-up GRUB to be able to boot from an encrypted drive. Edit `/etc/default/grub` and set

    GRUB_ENABLE_CRYPTODISK=y
    
then add to the `GRUB_CMDLINE_DEFAULT_LINUX` the kernel option `cryptdevice=UUID=<LUKS_UUID>:cryptlvm` where the UUID is that of the entire encrypted luks container.

### Decrypting all drives

So GRUB can unlock the initramfs without having to enter the passphrase again, and so we can decrypt all other drives on boot, we first create a keyfile with which to decrypt the drives. As with the passphrase, I'm using the same keyfile for all drives:

    $ dd bs=512 count=8 if=/dev/urandom iflag=fullblock | install -m 0600 /dev/stdin /etc/cryptsetup-keys.d/<key_name>.key

then for each luks container, add the key to the container with

    $ cryptsetup -v luksAddKey <path/to/container> /etc/cryptsetup-keys.d/<key_name>.key

Then edit the mkinitcpio config file again and add the path to the key in the `FILES` array:

    FILES=(/etc/cryptsetup-keys.d/<key_name>.key)

and add the key to `/etc/default/grub` as kernel parameter

    cryptkey=rootfs:/etc/cryptsetup-keys.d/<key_name>.key

Then regenerate the initramfs:

    $ mkinitcpio -P 

and generate the GRUB config file:

    $ grub-mkconfig -o /boot/grub/grub.cfg

Finally, in order to decrypt the rest of the drives, we need to add them to `/etc/crypttab`, which works like `fstab`, but for decrypting drives. For each luks container that's not the one decrypted by GRUB with the above config, add an entry to `/etc/crypttab` like

    <mapper_name>   <partition_UUID>    /etc/cryptsetup-keys.d/<key_name>.key

where `<mapper_name>` is the name used in `/dev/mapper/<mapper_name>`, and the partition UUID can be found like

    $ blkid /path/to/partition

Finally, continue with the **Reboot** step of the Installation guide, then on with post-install steps

## Post-Installation

### Network

Install Network Manager, so we can manage networks either with the cli or with a GUI after installing a display manager:

    $ pacman -S networkmanager
    $ systemctl enable NetworkManager.service
    $ systemctl start NetworkManager.service

Install `ufw` (Uncomplicated Firewall) for easy manipulation of iptables and nftables:

    $ pacman -S ufw
    $ systemctl enable ufw.service
    $ systemctl start ufw.service

Set the default `ufw` policy to deny all incoming connections (you can set up your own incoming rules as needed later):

    $ ufw default deny

Then enable `ufw` (this is only needed once at this point):

    $ ufw enable

### Timezone and NTP

Set the timezone:

    $ timedatectl set-timezone America/Montevideo

and check the system clock is correct with 

    $ timedatectl

If everything is correct, sync the hardware clock to the system clock

    $ hwclock --systohc

To ensure the time is synced via internet (NTP protocol), edit `/etc/systemd/timesyncd.conf` and uncomment the `NTP` and `FallbackNTP` lines. Then move all values in `FallbackNTP` to `NTP`, and add the following values

    FallbackNTP=0.pool.ntp.org 1.pool.ntp.org 0.fr.pool.ntp.org

Finally, enable syncing:

    $ timedatectl set-ntp true
    $ systemctl enable systemd-timesyncd.service
    $ systemctl start systemd-timesyncd.service

### Users and Groups

In my case, I created two users, both with `sudo` privileges: one for personal use, and one to use for work. Users can be created with:

    $ useradd -m -G wheel -s /bin/zsh <name>

This adds the user to the `wheel` group (we'll set up sudo access to this group next) aside from their own group, sets `zsh` as their default shell, and creates the home directory. Because the two user's home directories were already created as mounted partitions, this will throw a warning that `skel` files were not copied. We can fix that by:

    $ cp /etc/skel/.* /path/to/user/home
    $ chown -R <user>:<user> /path/to/user/home

to copy all skel files directly to their home, then transfering ownership of the home and all files in it to the user (since the home will be owned by root originally). If wanted, you can also `chmod` the home and/or files to restrict access from other users. Also, set the password for users with

    $ passwd <user>

Finally, to allow `wheel` members to use `sudo`:

    $ visudo

and uncomment the line for `%wheel`.

### Other security configs

Make failed logins have a delay of 4s: edit `/etc/pam.d/system-login` and add

    auth optional pam_faildelay.so delay=4000000

### Pacman

Set up pacman to clear the package cache weekly, keeping only the latest three package versions:

    $ pacman -S paccache
    $ systemctl enable paccache.timer.service
    $ systemctl start paccache.timer.service

Enable parallel downloads with pacman by editing `/etc/pacman.conf` and uncommenting the `ParallelDownloads` line

Allow colors in pacman by editing `/etc/pacman.conf` and uncommenting the `Color` line

Enable the `multilib` repository (needed for packages like Steam) by uncommenting the `[multilib]` section in `/etc/pacman.conf`

#### Hooks

We can set up some hooks in pacman for some quality of life improvements. First edit the `/etc/pacman.conf` file and uncomment the `HooksDir` line. Then

    $ mkdir /etc/pacman.d/hooks
    $ nvim /etc/pacman.d/hooks/20-list-orphans.hook
    $ nvim /etc/pacman.d/hooks/30-list-packages.hook

and follow the instructions in [the Arch Wiki](wiki.archlinux.org/title/Pacman/Tips_and_tricks) sections about listing packages and removing orphans to create the hook contents. The `list-packages` hook should export all installed packages to a file (in my case, in my `dotfiles` directory, so I can keep a synced copy).

### Mirrors

`pacman` mirrors may become outdated. The `reflector` package can be installed to update mirrors automatically. Edit the countries where to get mirrors from by editing the `/etc/xdg/reflector/reflector.conf` file. Available countries can be listed using

    $ reflector --list-countries

Also run

    $ systemctl enable reflector.service
    $ systemctl start reflector.service

to enable reflector on boot, and to run it at that time.

#### AUR

We will need to download some packages from the AUR eventually, so we might as well configure stuff for that now. We start by editing some configs in `/etc/makepkg.conf` which is used to build the AUR packages.

    $ nvim /etc/makepkg.conf

We can start by allowing `make` to use multiple cores by uncommenting the `MAKEFLAGS` line and setting the flag to `-j4` to use 4 cores.

We'll be using an AUR helper to install AUR packages, and it looks like `paru` is recommended as it forces `PKGBUILD` review at time of installation, so we'll just install that one

    $ git clone https://aur.archlinux.org/paru.git
    $ cd paru
    $ makepkg -si

### Graphical user interface

This assumes we're using Wayland, as it is the recommended display server protocol. Still, install `xorg-xwayland` to provide XWayland compatibility to X11 apps.

To enable Wayland support for Qt apps, install `qt5-wayland` and `qt6-wayland`.

Install drivers for AMD GPUs (my gpu at time of writing supports [AMDGPU](wiki.archlinux.org/title/AMDGPU):

    $ pacman -S mesa vulkan-radeon xf86-video-amdgpu

#### KDE Plasma

Install the `plasma` group. At time of writing, a single package in the group, `plasma-sdk` was not needed. All dependency options were chosen by default, except for `pipewire-jack` over `jack2`.

Plasma does not come with its own display manager, but SDDM is recommended. Install it with

    $ pacman -S sddm sddm-kcm qt5-declarative layer-shell-qt layer-shell-qt5

Also enable SDDM as a service to run the display manager on boot

    $ systemctl enable sddm.service

Follow the instructions on [SDDM for Wayland](wiki.archlinux.org/title/SDDM#Wayland) for specific extra configuration for Wayland on Plasma.

Install KDE applications by using the `kde-applications` group

    $ pacman -S kde-applications

and choose the ones you need.

### Vulkan

In order to use [Vulkan](wiki.archlinux.org/title/Vulkan) to run games, we need to install vulkan drivers:

    $ pacman -S vulkan-icd-loader

and when using AMD gpus (as at time of writing):

    $ pacman -S vulkan-radeon amdvlk vulkan-tools

Then check that vulkan is working by running

    $ vulkaninfo

and getting gpu information.

