# Set up the temporary tty keyboard layout
loadkeys hu

# Set up wifi if necessary (ethernet works by default)
wifi-menu
ping archlinux.org
 
# Update the system clock
timedatectl set-ntp true
timedatectl status

# Partition the disk(s)
cfdisk
# For UEFI systems, use a GPT label
# For Legacy systems, use a DOS label
# UEFI ONLY!!!: 512MB EFI System Partition
# The rest of the space can be used as Linux Filesystem
# 150% of total ram capacity can be used for swap (optional)

# Format the partitions
mkfs.vfat /dev/sdx1 # UEFI ONLY!!!
mkfs.ext4 /dev/sdx2
mkswap /dev/sdx3 # Only if you created a swap partition
swapon /dev/sdx3 # Only if you created a swap partition

# Mount the paritions
mount /dev/sdx2 /mnt
mkdir /mnt/boot # UEFI ONLY!!!
mount /dev/sdx1 /mnt/boot # UEFI ONLY!!!

# Update the pacman mirror list (technically optional, though it saves time)
reflector --verbose --latest 5 --sort rate --save /etc/pacman.d/mirrorlist

# Install the base system, nvim and networkmanager
# intel-ucode should be swapped for amd-ucode for amd cpus
pacstrap /mnt base base-devel linux linux-firmware neovim networkmanager intel-ucode

# Generate the fstab
genfstab -U /mnt >> /mnt/etc/fstab

# Change root into the installed system
arch-chroot /mnt

# Link a timezone to the system
# Europe, Bucharest in my case
ln -sf /usr/share/zoneinfo/Europe/Bucharest /etc/localtime
hwclock --systohc

# Set up the system locale
# Uncomment "en_US.UTF-8 UTF-8" for english
nvim /etc/locale.gen
locale-gen

# Set up the system langauge
echo "LANG=en_US.UTF-8" > /etc/locale.conf

# Set up the permanent virtual console (tty) keyboard layout
# Standard Hungarian in my case
echo "KEYMAP=hu" > /etc/vconsole.conf

# Change the virtual console (tty) font (requires terminus-font)
echo "FONT=ter-118n" >> /etc/vconsole.conf

# Set the system hostname
# Change *myhostname* to the actual desired hostname
echo "*myhostname*" > /etc/hostname
nvim /etc/hosts
127.0.0.1   localhost
::1     localhost
127.0.1.1	*myhostname*.localdomain	*myhostname*

# Enable networkmanager
systemctl enable NetworkManager

# Set the root account's passoword
passwd

# Add a normal user
# Change *myusername* to the actual desired username
useradd -mG wheel,users,audio,video,optical,storage,power,rfkill *myusername*

# Set the regular user's password
passwd *myusername*

# Enable sudo permissions for all the users in the %wheel group
# by uncommenting the desired %wheel privilige (sudo with/without a password)
EDITOR=nvim visudo

# Installing the GRUB bootloader
# LEGACY SYSTEMS:
pacman -S grub
grub-install /dev/sdx
grub-mkconfig -o /boot/grub/grub.cfg

# UEFI SYSTEMS:
pacman -S grub efibootmgr
mkdir /boot/efi
mount /dev/sdx1 /boot/efi
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=GRUB
grub-mkconfig -o /boot/grub/grub.cfg
mkdir /boot/efi/EFI/BOOT
cp /boot/efi/EFI/GRUB/grubx64.efi /boot/efi/EFI/BOOT/BOOTX64.EFI
# CREATE A STARTUP.NSH SCRIPT:
nvim /boot/efi/startup.nsh # Write: bcf boot add 1 fs0:\EFI\GRUB\grubx64.efi "GRUB" *NEWLINE* exit

# Exit the chroot environment and reboot into the freshly installed system
exit
shutdown now
