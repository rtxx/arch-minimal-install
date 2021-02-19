# Minimal Arch Install
## Script 
The script basically does all this automatically. Needs the partitions already done. After that installs base and asks for grub, X and i3 installation.

## General install steps
1. Connect to the internet

    Connecting to a network with ```iwcl``` or wired connecting
   
    Update system clock 
2. Create partitions and mount them

   Creating partitions using ```cfdisk```, format and mount
  
3. Install arch

   Installing arch using ```pacstrap```

   Config ```fstab timezone locale-gen keyboard-layout hostname hosts networkmanager initramfs grub```

4. Config it

## Pre-Install and connect to the internet
Load pt keyboard
```
loadkeys pt-latin1
```

Wireless configuration
```iwctl```
* List devices
```device list```
* Scan networks
```station [device] scan```
* Get avaiable networks
```station [device] get-networks```
* Connect to a networks
```station [device] connect [SSID]```
* Check connection
```ping www.google.com```

Update system clock
```
timedatectl set-ntp true
```

## Disk partition
My default config with 3 partitions
Open ```cfdisk``` and create
* boot with at least 512MB, should be /dev/sdx1
* swap with atleast 1GB, should be /dev/sdx2
* root with the remaining, should be /dev/sdx3

Format the partitions 
```
mkfs.ext4 /dev/sdx1
mkswap /dev/sdx2
mkfs.ext4 /dev/sdx3
``` 
and mount them
```
mount /dev/sdx3 /mnt
mkdir /mnt/{boot,home}
mount /dev/sdx1 /mnt/boot
swapon /dev/sdx2 
```

## Install Arch
#### Install base system and extras 
```
pacstrap /mnt base linux linux-firmware nano networkmanager grub efibootmgr os-prober
```

#### Install Microcode - https://wiki.archlinux.org/index.php/Microcode

>Processor manufacturers release stability and security updates to the processor microcode. These updates provide bug fixes that can be critical to the stability of your system. Without them, you may experience spurious crashes or unexpected system halts that can be difficult to track down.

Amd: amd-ucode or Intel: intel-ucode
```
pacstrap /mnt amd-ucode
```

#### Generate fstab - https://wiki.archlinux.org/index.php/Fstab

>The fstab file can be used to define how disk partitions, various other block devices, or remote filesystems should be mounted into the filesystem.

```
genfstab -U /mnt >> /mnt/etc/fstab
```

#### ```chroot``` into the new system - https://wiki.archlinux.org/index.php/Chroot

>A chroot is an operation that changes the apparent root directory for the current running process and their children.

```
arch-chroot /mnt
```

#### Set the time zone - https://wiki.archlinux.org/index.php/System_time#Time_zone
```
timedatectl set-timezone Europe/Lisbon
```

#### Generate /etc/adjtime - https://wiki.archlinux.org/index.php/System_time#Set_hardware_clock_from_system_clock - https://man.archlinux.org/man/hwclock.8#The_Adjtime_File

>The following sets the hardware clock from the system clock. Additionally it updates /etc/adjtime or creates it if not present.

```
hwclock --systohc
```

#### Select locale

>Locales are used by glibc and other locale-aware programs or libraries for rendering text, correctly displaying regional monetary values, time and date formats, alphabetic idiosyncrasies, and other locale-specific standards.

Uncomment the locale needed

```
nano /etc/locale.gen
```
Generate it
```
locale-gen
```
#### Create locale.conf and set the language - https://wiki.archlinux.org/index.php/Locale#Setting_the_system_locale


>To set the system locale, write the LANG variable to /etc/locale.conf, where en_US.UTF-8 belongs to the first column of an uncommented entry in /etc/locale.gen
```
echo "LANG=en_US.UTF-8" >> /etc/locale.conf
```

#### Set keyboard layout
```
echo "KEYMAP=pt-latin1" >> /etc/vconsole.conf
```

#### Set hostname - https://wiki.archlinux.org/index.php/Network_configuration#Set_the_hostname

>A hostname is a unique name created to identify a machine on a network, configured in /etc/hostname

```
echo "archx64" >> /etc/hostname
```

#### Config /etc/hosts - https://wiki.archlinux.org/index.php/Network_configuration#Local_hostname_resolution
```
127.0.0.1    localhost
::1          localhost
127.0.1.1    archx64.localdomain     archx64
```
```
echo -e "127.0.0.1	localhost\n::1		localhost\n127.0.1.1	archx64.localdomain	archx64" >> /etc/hosts
```

#### Enable networkmanager service - https://wiki.archlinux.org/index.php/networkmanager

>NetworkManager is a program for providing detection and configuration for systems to automatically connect to networks.
```
systemctl enable NetworkManager.service
```

#### Recreate initramfs - https://wiki.archlinux.org/index.php/Arch_boot_process#initramfs
```
mkinitcpio -P
```

#### Config bootloader - https://wiki.archlinux.org/index.php/Arch_boot_process#Boot_loader

>A boot loader is a piece of software started by the firmware (BIOS or UEFI). It is responsible for loading the kernel with the wanted kernel parameters, and initial RAM disk based on configuration files. In the case of UEFI, the kernel itself can be directly launched by the UEFI using the EFI boot stub. A separate boot loader or boot manager can still be used for the purpose of editing kernel parameters before booting.

GRUB - https://wiki.archlinux.org/index.php/GRUB
* MBR ```grub-install --target=i386-pc /dev/sdx```
* UEFI ```grub-install --target=x86_64-efi --efi-directory=/mnt/boot --bootloader-id=archlinux```
* BIOS/GPT - https://wiki.archlinux.org/index.php/GRUB#GUID_Partition_Table_.28GPT.29_specific_instructions

Generate main config file for grub
```
grub-mkconfig -o /boot/grub/grub.cfg
```

#### Set root password
```
passwd
```

#### Exit chroot and reboot!
```
exit && reboot
```

#### Arch should now be installed, login with root

## Config the new install
#### Install sudo and bash completion
sudo - https://wiki.archlinux.org/index.php/Sudo
>Sudo allows a system administrator to delegate authority to give certain users—or groups of users—the ability to run commands as root or another user while providing an audit trail of the commands and their arguments.

bash-completion - https://wiki.archlinux.org/index.php/Bash#Tab_completion
>Tab completion is the option to auto-complete typed commands by pressing Tab (enabled by default).
```
pacman -S sudo bash-completion
```

#### Edit sudoers and remove the comment on %wheel - https://wiki.archlinux.org/index.php/Sudo#Using_visudo
>The configuration file for sudo is /etc/sudoers. It should **always** be edited with the visudo command
```
EDITOR=nano visudo
```
```
## Uncomment to allow members of group wheel to execute any command
%wheel ALL=(ALL) ALL
```
#### Create user and add them to wheel group - https://wiki.archlinux.org/index.php/sudo#Example_entries
>When creating new administrators, it is often desirable to enable sudo access for the wheel group and add the user to it, since by default Polkit treats the members of the wheel group as administrators. If the user is not a member of wheel, software using Polkit may ask to authenticate using the root password instead of the user password.
```
useradd -m -G wheel [user]
```

#### Add a password to the new created user
```
passwd [user]
```

#### Exit root and login with new user
```
exit
```

#### Test permissions of the new user
```
sudo pacman -Syu
```

#### At this point, arch is installed and ready to use, although it is very *very* bare bones. Highly recommended installing atleast a firewall and a WM, like i3 or a DE like xfce.

### Extras

#### Enabling multilib - https://wiki.archlinux.org/index.php/Official_repositories#multilib
> multilib contains 32-bit software and libraries that can be used to run and build 32-bit applications on 64-bit installs (e.g. wine, steam, etc).
```
sudo nano /etc/pacman.conf
``` 
and uncomment
```
[multilib]
Include = /etc/pacman.d/mirrorlist
```
#### Installing X - https://wiki.archlinux.org/index.php/Xorg
> Xorg (commonly referred as simply X) is the most popular display server among Linux users.
```
sudo pacman -S xorg-server xorg-xinit xorg-setxkbmap
```
or install Xorg group with a couple of extras
```
sudo pacman -S xorg xorg-xinit
```
#### Setting the keyboard layout - https://wiki.archlinux.org/index.php/Xorg/Keyboard_configuration#Setting_keyboard_layout
```
localectl set-x11-keymap pt
```

#### Installing graphic drivers - https://wiki.archlinux.org/index.php/Xorg#Driver_installation. 
Example: Intel
```
sudo pacman -S xf86-video-intel mesa lib32-mesa
```

#### Window Manager - i3 - https://wiki.archlinux.org/index.php/i3
>i3 is a dynamic tiling window manager inspired by wmii that is primarily targeted at developers and advanced users.

Config ```.xinitrc```, from https://wiki.manjaro.org/index.php/Proper_~/.xinitrc_File
```
nano ~/.xinitrc
```
```
#!/bin/sh
#
# ~/.xinitrc
#
# Executed by startx (run your window manager from here)

if [ -d /etc/X11/xinit/xinitrc.d ]; then
  for f in /etc/X11/xinit/xinitrc.d/*; do
    [ -x "$f" ] && . "$f"
  done
  unset f
fi
exec i3
```
#### Install i3 and some extras
```
sudo pacman -S i3-wm i3status i3lock dmenu ttf-opensans noto-fonts ttf-fantasque-sans-mono xterm htop gvfs
```
Optional: i3 layout based on manjaro i3 - https://github.com/rtxx/linux-config/blob/master/.config/i3/config . Copy to ~/.config/i3.
Warning! If this layout is used, there's a bunch of programs not installed yet.

#### Desktop Environment - xfce - https://wiki.archlinux.org/index.php/Xfce
>Xfce is a lightweight and modular desktop environment currently based on GTK 3.

Config ```.xinitrc```, from https://wiki.manjaro.org/index.php/Proper_~/.xinitrc_File
```
nano ~/.xinitrc
```
```
#!/bin/sh
#
# ~/.xinitrc
#
# Executed by startx (run your window manager from here)

if [ -d /etc/X11/xinit/xinitrc.d ]; then
  for f in /etc/X11/xinit/xinitrc.d/*; do
    [ -x "$f" ] && . "$f"
  done
  unset f
fi
exec startxfce4
```
#### Install xfce and some extras
```
sudo pacman -S xfce4 xfce4-goodies noto-fonts ttf-fantasque-sans-mono htop gvfs
```
> Gvfs - https://wiki.archlinux.org/index.php/File_manager_functionality

#### Start X server - https://wiki.archlinux.org/index.php/Xinit
>The xinit program allows a user to manually start an Xorg display server. The startx script is a front-end for xinit.
```
startx
```

#### Right now you should have a bare minimium graphic environment to use. Highly recommended install a DM, like lightdm.

#### User Directories - https://wiki.archlinux.org/index.php/XDG_user_directories
>xdg-user-dirs is a tool to help manage "well known" user directories like the desktop folder and the music folder. It also handles localization (i.e. translation) of the filenames.
```
sudo pacman -S xdg-user-dirs
```
Run ```xdg-user-dirs-update``` to create the default directories in /home (Documents, Downloads, etc)

#### Firewall - https://wiki.archlinux.org/index.php/Uncomplicated_Firewall
>Ufw stands for Uncomplicated Firewall, and is a program for managing a netfilter firewall. It provides a command line interface and aims to be uncomplicated and easy to use.
```
sudo pacman -S ufw
```
Basic config
```
ufw default deny && ufw allow from 192.168.0.0/24 && ufw limit ssh
```
Enable it and start it
```
ufw enable
systemctl enable ufw
systemctl start ufw
ufw status
```
Install gufw, GUI frontend for ufw - https://wiki.archlinux.org/index.php/Uncomplicated_Firewall#Gufw
>gufw is a GTK front-end for Ufw that aims to make managing a Linux firewall as accessible and easy as possible.
```
sudo pacman -S gufw
```
#### Sound - https://wiki.archlinux.org/index.php/PulseAudio - https://wiki.archlinux.org/index.php/Advanced_Linux_Sound_Architecture

>PulseAudio is a general purpose sound server intended to run as a middleware between your applications and your hardware devices, either using ALSA or OSS.

>The Advanced Linux Sound Architecture (ALSA) provides kernel driven sound card drivers. It replaces the original Open Sound System (OSS).
```
sudo pacman -S pulseaudio pulseaudio-alsa alsa-utils pavucontrol
```
Check sound levels with ```alsamixer``` or ```pavucontrol```.

Install ```volumeicon``` for a tray icon.

If ```paucontrol``` wont work, reboot and check again, if still the same, open as user (not root) and check it.

#### Display Manager - https://wiki.archlinux.org/index.php/LightDM
>LightDM is a cross-desktop display manager.
```
sudo pacman -S lightdm lightdm-gtk-greeter
```
Enabling it
```
systemctl enable lightdm
```
Install ``` lightdm-gtk-greeter-settings``` to change appearance of it

#### Keyring - https://wiki.archlinux.org/index.php/GNOME/Keyring
>GNOME Keyring is "a collection of components in GNOME that store secrets, passwords, keys, certificates and make them available to applications."
```
sudo pacman -S gnome-keyring
```
Install ```seahorse```, GUI frontend for ```gnome-keyring```, - https://wiki.archlinux.org/index.php/GNOME/Keyring#Manage_using_GUI
```
sudo pacman -S seahorse
```
##### Auto unlock keyring - https://wiki.archlinux.org/index.php/GNOME/Keyring#Automatic_login

Without a DM, using ```.xinitrc```
```
eval $(/usr/bin/gnome-keyring-daemon --start --components=pkcs11,secrets,ssh)
export SSH_AUTH_SOCK
```
With a DM, like LightDM

It should be automatic. However if it is not unlocked on login, open ```seahorse```, create a new *keyring* with the name ```login``` and the password should be the *same* as the current user.

To enable the keyring for terminal applications update ```.bash_profile``` with
```
if [ -n "$DESKTOP_SESSION" ];then
    eval $(gnome-keyring-daemon --start)
    export SSH_AUTH_SOCK
fi
```
### Appearance
#### QT and GTK 
>Qt and GTK based programs both use a different widget toolkit to render the graphical user interface. Each come with different themes, styles and icon sets by default, among other things, so the "look and feel" differ significantly. 
#### Install Breeze theme - https://wiki.archlinux.org/index.php/Uniform_look_for_Qt_and_GTK_applications#Breeze
```
sudo pacman -S breeze breeze-gtk breeze-icons qt5ct lxappearance
```
* Config QT
   ```
   echo "QT_QPA_PLATFORMTHEME=qt5ct" >> /etc/environment
   ```
   Open ```qt5ct``` and change theme to "breeze" and icons.

* Config GTK

   Open ```lxappearance``` and change theme and icons.

Xresources - https://github.com/rtxx/linux-config/blob/master/.Xresources . Copy to ```~/.Xresources```


#### Install Materia theme - https://github.com/nana-4/materia-theme
```
sudo pacman -S kvantum kvantum-theme-materia materia-gtk-theme capitaine-cursors arc-icon-theme lxappearance
```
* Config QT
   ```
   echo "QT_QPA_PLATFORMTHEME=qt5ct" >> /etc/environment
   ```
   Open ```qt5ct``` and change theme to "kvantum" and icons.

* Config GTK

   Open ```lxappearance``` and change theme and icons.

* Config Kvantum

   Open ```kvantummanager``` and change theme.
   >Kvantum (kvantum-qt5) is customizable SVG-based theme engine for Qt5 that comes with a variety of built-in styles, including versions of some of popular GTK themes such as Adapta, Arc, Ambiance, Materia.
   
Xresources - Monokai colors - https://github.com/logico-dev/Xresources-themes/blob/master/base16-monokai.Xresources . Copy to ```~/.Xresources```

   
#### Wallpapers
```
sudo pacman -S archlinux-wallpaper
```
Wallpapers are located in ```/usr/share/backgrounds/archlinux/```

#### Fonts

Open ```lxappearance``` and ```qt5ct``` change to ```Noto Sans Regular``` ```Fantasque Sans Mono```

