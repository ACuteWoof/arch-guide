# Arch Linux Installation Guide
This guide assumes that you're already in the arch linux installation medium.

## set the keyboard layout
The default keyboard layout is **US**. Available layouts can be listed with:
```
ls /usr/share/kbd/keymaps/**/*.map.gz
```
you can set a new keymap with:
```
loadkeys <keymap>
```

## check if you're booted in BIOS or UEFI mode
```
ls /sys/firmware/efi/efivars
```
if your system is booted into BIOS mode then this directory doesn't exist.

## connect to the internet
- If you're using ethernet, make sure your network interface is connected:

```
ip link
```
- If you're using wi-fi, then use iwctl:
```
iwctl
#you'll get a interactive prompt with a prefix of [iwd]#

# list your wifi device name(s), most likely it's wlan0
device list

# scan for available networks
station <device name> scan

# list the scanned networks
station <device name> get-networks

#connect to a network
station <device name> connect <SSID>

#exit out of the interactive prompt
exit
```
- verify the connection:
```
ping -c 5 archlinux.org
```

## update the system clock
```
timedatectl set-ntp true
```

## partition your disk
Disks are assigned to a block device such as /dev/sda, /dev/nvme0n1 or /dev/mmcblk0. To list the disks recognized by the live system, run:
```
fdisk -l
```
use `fdisk` or `cfdisk` to modify the partition table.

`example layouts` ( assuming that your device name is /dev/sda )
- for BIOS with MBR:

|   partition   |   type                |   mount point |
|:-------------:|:---------------------:|:-------------:|
|   /dev/sda1   |   linux filesystem    |   /mnt        |
|   /dev/sda2   |linux swap             |   [SWAP]      |

- for UEFI with GPT

|   partition   |   type                |   mount point |
|:-------------:|:---------------------:|:-------------:|
|   /dev/sda1   |   efi system          | /mnt/boot/efi |
|   /dev/sda2   |   linux filesystem    |   /mnt        |
|   /dev/sda3   |   linux swap          |   [SWAP]      |

### suggested size for swap and efi system:
- **swap:**

| system RAM | swap size |
|:----------:|:---------:|
| < 2 GB | 2x the amount of RAM |
| 2-8 GB | equal to the amount of RAM |
| 8-64 GB | at least 4 GB |

- **efi system :** 300 MiB

# format your partitions

`replace X with respective partition number.`
- EFI system
```
mkfs.vfat /dev/sdaX
```
- swap
```
# initialize your swap partition with mkswap
mkswap /dev/sdaX
```
- root partition
```
mkfs.ext4 /dev/sdaX
# you can create a btrfs file system with mkfs.btrfs
```

# mount the filesystems
first mount your root partition to /mnt:
```
mount /dev/root_partition /mnt
```
enable your swap partition:
```
swapon /dev/swap_partition
```
create remaining mount points such as _/mnt/boot/efi_ with mkdir:
```
mkdir -p /mnt/boot/efi
```
mount your EFI system (for UEFI only)
```
mount /dev/efi_partition /mnt/boot/efi
```

# Installation
Packages are downloaded from mirror servers. After connecting to the internet, `reflector` updates the mirror list, sorting them by download rate. Mirrors are defined in `/etc/pacman.d/mirrorlist`. You can re-order the mirror list by editing the file.
## base installation
```
pacstrap /mnt base base-devel linux linux-firmware neovim nano
```
## configure the system
### fstab
create the file system table with genfstab:
```
genfstab -U /mnt >> /mnt/etc/fstab
```
### chroot
change root into the newly installed system to configure some stuff:
```
arch-chroot /mnt
```
### time-zone
you can use tab-completion to see the regions and city
```
ln -sf /usr/share/zoneinfo/REGION/CITY /etc/localtime

hwclock --systohc
```
### locale
use nvim or any other text editor and uncomment `en_US.UTF-8 UTF-8` and other needed languages in `/etc/locale.gen`. Then generate the locales:
```
locale-gen

# set the LANG variable
echo LANG=en_US.UTF-8 > /etc/locale.conf
```
set the console keymap:
```
echo KEYMAP=yourKeymap > /etc/vconsole.conf
```
### network configuration
set your host name:
```
echo yourHostName > /etc/hostname
```
edit your `/etc/hosts` file:
```
# add these lines
127.0.0.1   localhost
::1         localhost
127.0.1.1   myhostname.localdomain myhostname
```
# Install Bootloader
- UEFI
```
pacman -s grub efibootmgr os-prober

grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=GRUB

grub-mkconfig -o /boot/grub/grub.cfg
```
- BIOS
```
pacman -S grub os-prober

grub-install --target=i386-pc /dev/sda

grub-mkconfig -o /boot/grub/grub.cfg
```

# Setup users
## set root password
```
passwd
```
## add a user
```
useradd -m -G wheel,audio,video,input,kvm yourUserName
passwd yourUserName
```
## give administrative privileges to your user
```
EDITOR=nvim visudo
# you can use any other editor too, like vim, nano
```
uncomment the following line:
```
%wheel ALL=(ALL) ALL
```

# Useful packages
- networking
```
pacman -S networkmanager

systemctl enable NetworkManager
```
- bluetooth
```
pacman -S blueman

systemctl enable bluetooth
```
- sound
```
pacman -S pulseaudio-alsa pavucontrol
#pavucontrol-qt for Qt desktop
```

- graphics driver
```
pacman -S mesa vulkan-icd-loader
```
- power management
```
pacman -S tlp

systemctl enable tlp
```

# Desktop Environment
- KDE
```
pacman -S xorg sddm plasma plasma-wayland-session

systemctl enable sddm
```
- Other desktop environments:

check out [arch wiki](https://wiki.archlinux.org/title/Desktop_environment)

# Reboot
```
exit

umount -R /mnt

reboot
```