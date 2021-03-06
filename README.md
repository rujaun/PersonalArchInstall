# Personal Arch Linux install guide
This guide was written for my own personal use.

## Disclaimer
No liability for the contents of this documents can be accepted. Use the concepts, examples and other content at your own risk. There may be errors and inaccuracies, that may of course be damaging to your system. Although this is highly unlikely, you should proceed with caution. The author does not accept any responsibility for any damage incurred.

---
## Update the system clock
```
timedatectl set-ntp true
timedatectl set-local-rtc 1 --adjust-system-clock
```
---

## Partition disks

List all disks:
```
fdisk -l
```

Write new GPT parition label:
```
parted --script /dev/sda mklabel gpt
```

Partition Layout:
```
EFI = +512MiB Hex Code = ef00
BOOT = +1024MiB Hex Code = 8300
LVM = Rest of the space available. Hex Code = 8e00
```

Partition Disks:
```
gdisk /dev/sda
```

Create partition: `n`

Review partitions: `p`

Write partitions: `w`

Reboot so that the kernel reads new partition structure.

Zero out the partitions:
```
cat /dev/zero > /dev/sda1
cat /dev/zero > /dev/sda2
cat /dev/zero > /dev/sda3
```

---
## Create filesystems for EFI and boot parititons

Create UEFI filesystem:
```
mkfs.fat -F 32 /dev/sda1
```

Create BOOT filesystem:
```
mkfs.ext2 /dev/sda2
```

Encrypt and open LVM partiton:
```
cryptsetup -c aes-xts-plain64 -h sha512 -s 512 --use-random luksFormat /dev/sda3

cryptsetup luksOpen /dev/sda3 encrypted
```

Create encrypted LVM partitions:
```
pvcreate /dev/mapper/encrypted
vgcreate Arch /dev/mapper/encrypted

lvcreate -L +32768M Arch -n swap
lvcreate -l +100%FREE Arch -n root
```

Create filesystems on encrypted partitons:
```
mkswap /dev/mapper/Arch-swap
mkfs.ext4 /dev/mapper/Arch-root
```

Mount the new partitions:
```
mount /dev/mapper/Arch-root /mnt
swapon /dev/mapper/Arch-swap
mkdir /mnt/boot
mount /dev/sda2 /mnt/boot
mkdir /mnt/boot/efi
mount /dev/sda1 /mnt/boot/efi
```

---
## Mirror selection
By this point you should have established an internet connection with either ethernet or wifi

```
ping 8.8.8.8
```

Sync pacman:
```
pacman -Syy
```

Install reflector:
```
pacman -S reflector
```

Get best performant mirrors and update mirrorlist:
```
reflector -c "US" -f 12 -l 10 -n 12 --save /etc/pacman.d/mirrorlist
```
---
### Arch Linux installation

Install base system:
```
pacstrap /mnt base base-devel grub efibootmgr linux linux-headers linux-firmware util-linux lvm2 intel-ucode vim htop git
```
---
### Configure Arch Linux installation

Generate Fstab file:
```
genfstab -U /mnt >> /mnt/etc/fstab
```

Chroot into newly installed system
```
arch-chroot /mnt
```

Set the timezone:
```
ln -sf /usr/share/zoneinfo/Africa/Johannesburg /etc/localtime
hwclock --systohc
```

Set the locale:
```
vim /etc/locale.gen

### Uncomment
en_ZA.UTF-8
```

Generate the locale config:
```
echo LANG=en_ZA.UTF-8 > /etc/locale.conf
export LANG=en_ZA.UTF-8
locale-gen
```

Set the hostname:
```
echo archbox > /etc/hostname
```

Configure hosts file:
```
vim /etc/hosts

### Add the following:
127.0.0.1       localhost
::1             localhost
127.0.1.1       archbox
```

Set the root password:
```
passwd
```

---
### Install Plymouth
```
git clone https://aur.archlinux.org/plymouth.git
cd plymouth
makepkg -si
```

---
### Configure mkinitcpio with correct hooks for decryption

Edit `/etc/mkinitcpio.conf` and change the HOOKS statement to:

```
HOOKS=(base udev plymouth autodetect modconf block keymap plymouth-encrypt lvm2 resume filesystems keyboard fsck)
```

Generate your initrd image:
```
mkinitcpio -p linux
```

---
### Install Bootloader

Install GRUB:
```
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=ArchLinux
```

Edit `/etc/default/grub` and change:
```
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash loglevel=3 rd.udev.log_priority=3 vt.global_cursor_default=0"
GRUB_CMDLINE_LINUX="cryptdevice=/dev/sda3:encrypted resume=/dev/mapper/Arch-swap"
``` 

Create GRUB config:
```
grub-mkconfig -o /boot/grub/grub.cfg
```

Select Plymouth theme:
```
plymouth-set-default-theme -R spinfinity
```

Unmount all partitions and reboot:
```
umount -R /mnt
swapoff -a
reboot
```

---
### Install and enable NetworkManager
```
pacman -S networkmanager
systemctl enable NetworkManager.service
```
Reboot:
```
reboot
```
---
### Add local user account and install sudo
```
pacman -S sudo
```

Add new user and set password
```
useradd --create-home new_user
passwd new_user
```

Add the new user to wheel group:
```
usermod --append --groups wheel new_user
```

Edit sudoers file and give wheel group sudo privileges:
```
visudo

### Uncomment
%wheel ALL=(ALL) ALL
```
---
### Enable TRIM scheduling for SSDs
```
sudo systemctl enable fstrim.timer
```
---
### Install AUR (Arch User Repository) helper

Install git and golang:
```
sudo pacman -S git go
```

In home directory clone yay:
```
cd ~/
git clone https://aur.archlinux.org/yay-git.git
cd yay-git
```

Build and install yay:
```
makepkg -si
```
---
### Install Xorg and GPU drivers

Enable multilib for 32bit support:
```
sudo vim /etc/pacman.conf

### Uncomment
[multilib]
Include = /etc/pacman.d/mirrorlist
```

Upgrade system after multilib enable:
```
sudo pacman -Syu
```

Install Xorg and dkms:
```
sudo pacman -S xorg dkms 
```

For Nvidia:
```
sudo pacman -S nvidia-dkms nvidia-utils lib32-nvidia-utils nvidia-settings vulkan-icd-loader lib32-vulkan-icd-loader
```

For AMD:
```
sudo pacman -S lib32-mesa vulkan-radeon lib32-vulkan-radeon vulkan-icd-loader lib32-vulkan-icd-loader
```

For Intel:
```
sudo pacman -S lib32-mesa vulkan-intel lib32-vulkan-intel vulkan-icd-loader lib32-vulkan-icd-loader
```
---
### Install Plasma desktop environment
For a minimalistic Arch Linux installation, I don't install `kde-applications`.

Install Plasma:
```
sudo pacman -S plasma plasma-wayland-session
```

Install KDE thumbnail render dependencies:
```
sudo pacman -S kdegraphics-thumbnailers kio-extras
```

Install de/compression tools:
```
sudo pacman -S bzip2 gzip lzip xz p7zip unrar zip unzip
```

Install a few applications:
```
sudo pacman -S konsole notepadqq dolphin partitionmanager kcolorchooser krita okular vlc ark persepolis transmission-qt firefox chromium foliate alacritty
```

Enable SDDM with Plymouth:
```
sudo systemctl enable sddm-plymouth.service
```

Enable SDDM without Plymouth:
```
sudo systemctl enable sddm.service
```

---
### Auto mount second disk with Fstab
Create mount point directory:
```
sudo mkdir /mnt/sdb1
```

Retrieve UUID for the second disk:
```
lsblk -f
```

Edit Fstab file:
```
sudo vim /etc/fstab
```

Append the following with previously retrieved UUID:
```
UUID=<your-uuid-here>        /mnt/sdb1      ext4        defaults        0 2
```

Reboot.
```
reboot
```

Change ownership of mount point to enable writing to disk:
```
sudo chown new_user:wheel /mnt/sdb1
```

---
### Fix screen tearing on Plasma (Nvidia)

**NB: This now resides in config repo**

Under `System Settings` -> `Display and Monitor` -> `Compositor`:
`Tearing prevention("vsync")` set to `Never`

Install `force-composition-pipeline.sh` to:
```
~/.config/autostart-scripts/force-composition-pipeline.sh
```

Make it executable:
```
chmod +x ~/.config/autostart-scripts/force-composition-pipeline.sh
```

*Credit to: [vitkin](https://unix.stackexchange.com/users/316930/vitkin) - [Source](https://unix.stackexchange.com/questions/510757/how-to-automatically-force-full-composition-pipeline-for-nvidia-gpu-driver#answer-550695)*

---
### Install support for NTFS and exfat drives / partitions
```
sudo pacman -S ntfs-3g exfat-utils
```
---
### Install OpenSSH and add SSH key to ssh-agent
```
sudo pacman -S openssh
```
Start ssh-agent:
```
eval "$(ssh-agent -s)"
```
Close down permissions on existing SSH key:
```
chmod 400 ~/.ssh/id_rsa
```
Add SSH key to ssh-agent:
```
ssh-add ~/.ssh/id_rsa
```
---
### Install development tools from the AUR (Arch User Repository)
Install Visual Studio Code (vscodium)
```
yay -S vscodium-bin
```

Install powerline patched programming fonts:
```
yay -S awesome-terminal-fonts-git
yay -S powerline-fonts-git
yay -S nerd-fonts-source-code-pro
yay -S nerd-fonts-fira-code
```
---
### Install powerline-go
```
go get -u github.com/justjanne/powerline-go
```

Add this to `.bashrc`:
```
GOPATH=$HOME/go

function _update_ps1() {
    PS1="$($GOPATH/bin/powerline-go -error $?)"
}
if [ "$TERM" != "linux" ] && [ -f "$GOPATH/bin/powerline-go" ]; then
    PROMPT_COMMAND="_update_ps1; $PROMPT_COMMAND"
fi
```

Source `.bashrc` to update with new PS1:
```
source ~/.bashrc
```
---
### Install development tools from the Arch repo
Install Geany and Dbeaver:
```
sudo pacman -S geany dbeaver
```

Install Python 3:
```
sudo pacman -S python python-pip
```

Install Go:
```
sudo pacman -S go
```

---
### Vim-plug installation
```
curl -fLo ~/.vim/autoload/plug.vim --create-dirs \
    https://raw.githubusercontent.com/junegunn/vim-plug/master/plug.vim
```

---
### Docker installation
```
sudo pacman -S docker docker-compose
```

Add user to the `docker` group (requires restart):
```
sudo usermod --append --groups docker new_user
```

Start Docker service:
```
sudo systemctl start docker.service
```

Stop Docker service:
```
sudo systemctl stop docker.service
```

Get status of Docker service:
```
systemctl status docker
```

Verify Docker operation:
```
docker info
```

Test Docker installation with Arch Linux image and return `hello world`:
```
docker run -it --rm archlinux bash -c "echo hello world"
```

Install Kitematic:
```
yay -S kitematic
```

---
### Install credential manager for SSH key passphrases

Install required packages:
```
sudo pacman -S kwallet ksshaskpass kwalletmanager kwallet-pam signon-kwallet-extension
```
Install `ssh-agent.sh` to:
```
~/.config/plasma-workspace/env/ssh-agent.sh
```
Make it executable:
```
chmod u+x ~/.config/plasma-workspace/env/ssh-agent.sh
```

Set `SSH_ASKPASS` environment variable - install `askpass.sh` to:
```
~/.config/plasma-workspace/env/askpass.sh
```
Make it executable:
```
chmod u+x ~/.config/plasma-workspace/env/askpass.sh
```

Create an ssh-add startup script - install `ssh-add.sh` to:
```
~/.config/autostart-scripts/ssh-add.sh
```
Make it executable:
```
chmod u+x ~/.config/autostart-scripts/ssh-add.sh
```

Logout or reboot

Add your SSH key passphrases to kwallet:
```
ssh-add /path/to/key < /dev/null
```

**NB: Remember to set password for kwallet!**

**Configuration for more than one key:**

In `ssh-add.sh` replace:
```
ssh-add -q < /dev/null
```
with:
```
ssh-add -q ~/.ssh/key1 ~/.ssh/key2 ~/.ssh/key3 < /dev/null
```
Reboot.

Run this for each of your SSH private keys to store their passphrases in `kwallet`:
```
ssh-add /path/to/key < /dev/null
```

Reboot.

*Credit to: [Feakster](https://forum.manjaro.org/u/Feakster) - [Source](https://archived.forum.manjaro.org/t/howto-use-kwallet-as-a-login-keychain-for-storing-ssh-key-passphrases-on-manjaro-arm-kde/115719)*

---
### Install fonts

**NB: This now resides in config repo**

Make new fonts directory:
```
sudo mkdir /usr/share/fonts/WinFonts
```

Copy fonts to new directory:
```
sudo cp ~/WinFonts/* /usr/share/fonts/WinFonts/
```

Change permissions on new fonts and directory:
```
sudo chmod 644 /usr/share/fonts/WinFonts/*
```

Regenerate the fontconfig cache:
```
fc-cache --force
```

Create font config - install `local.conf` to:
```
/etc/fonts/local.conf
```

---
### Install LibreOffice
```
sudo pacman -S libreoffice-fresh
```

---
### Fix Plasma Discover repository loading
Install / reinstall the following and restart:
```
sudo pacman -S packagekit packagekit-qt5 appstream appstream-qt
```

---
### Install a few mathematics applications
```
sudo pacman -S geogebra kmplot cantor kalgebra labplot
```

---
### Install xanmod linux kernel

Install build dependencies:
```
sudo pacman -S xmlto inetutils bc cpio python-pytz python-babel python-docutils python-imagesize python-markupsafe python-jinja python-pygments python-snowballstemmer python-sphinx-alabaster-theme python-sphinxcontrib-applehelp python-sphinxcontrib-devhelp python-sphinxcontrib-htmlhelp python-sphinxcontrib-jsmath python-sphinxcontrib-qthelp python-sphinxcontrib-serializinghtml python-sphinx python-sphinx_rtd_theme gd netpbm gts gsfonts graphviz liblqr libraqm imagemagick
```

Import GPG keys:
```
gpg --keyserver pool.sks-keyservers.net --recv-keys ABAF11C65A2970B130ABE3C479BE3E4300411886
gpg --keyserver pool.sks-keyservers.net --recv-keys 647F28654894E3BD457199BE38DBBDC86092693E
```

Install xanmod kernel and headers from AUR (Arch User Repositories):

**NB: This will compile the latest upstream Linux kernel and apply xanmod patches. Might take a couple of hours.**

```
yay -S linux-xanmod linux-xanmod-headers
```

Update GRUB:
```
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

Reboot:
```
reboot
```

---
### Install research paper managers
```
yay -S jabref
yay -S zotero
```

<!--
Global Theme: Breeze
Plasma Style: Breeze
Application Style: Breeze
Window Decorations: Breezemite
Colors: Breeze Light
Fonts General: Noto Sans
Icons: Tela
Login Screen (SDDM): Nordic
Splash Screen: Dots ArchLinux Splashscreen
Konsole Color Profile: Dark One Nuanced
-->

