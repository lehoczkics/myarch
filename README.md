# myarch
Arch linux setup notes - WIP

# References
Arch Wiki first and foremost...
 - https://wiki.archlinux.org/index.php/Installation_guide
 - https://wiki.archlinux.org/index.php/User:Altercation/Bullet_Proof_Arch_Install
 - https://github.com/egara/arch-btrfs-installation
 - https://ix5.org/tech/2018/arch-install/

# Prerequisites, assumptions
 - Hardware: EFI capable laptop with empty SSD and enough RAM; Intel GPU
 - full-disk encryption
 - no swap therefore no hibernation
 - single btrfs volume with subvolumes
 - use buttermanager and grub-btrfs
 - Another machine on the same network to install over SSH

# Subvolume layout
The basic idea is to put most of the data on separate subvolumes to keep the root sublvolume's snapshots small.
Another goal is having selective backup from important data and not to bother with disposables.
- **rootfs**: /
- **home**: /home
- **tmp**: /tmp
- **var**: /var
- **dl**: /home/tsabi/Downloads - to exclude from backups
- **docker**: /var/lib/docker
- **kvm**: /home/tsabi/vms - to switch off CoW for VM images

# Prep
 - Boot up the target machine in UEFI mode with an Arch installer.
 - Connect network 
 - set root password 
 - `systemctl start sshd.service`
 - log in from another machine via SSH and continue installation comfortably.

# Partitioning
Set drive path and mount options as variables:
```
DRIVE=/dev/sda
o=defaults,x-mount.mkdir
o_btrfs=$o,compress=lzo,ssd,noatime
```

Do the partitioning in one shot (See below) or use `cgdisk` to partition interactively: 
```
sgdisk --zap-all $DRIVE
sgdisk --clear \
         --new=1:0:+550MiB --typecode=1:ef00 --change-name=1:EFI \
         --new=2:0:0       --typecode=2:8300 --change-name=2:cryptsystem \
           $DRIVE

mkfs.fat -F32 -n EFI /dev/disk/by-partlabel/EFI
```

# Encryption
Have your passphrase with you
```
cryptsetup luksFormat --type luks1 /dev/disk/by-partlabel/cryptsystem
cryptsetup open /dev/disk/by-partlabel/cryptsystem system
```
also create and add a key to luks key slot after filesystems are created

# Create btrfs and subvolumes on cryptsystem
```
mkfs.btrfs --force --label system /dev/mapper/system
mount -t btrfs LABEL=system /mnt
cd /mnt
btrfs subvolume create _active
btrfs subvolume create _active/rootfs
btrfs subvolume create _active/home
btrfs subvolume create _active/tmp
btrfs subvolume create _active/var
btrfs subvolume create _active/dl
btrfs subvolume create _active/docker
btrfs subvolume create _active/kvm
btrfs subvolume create _snapshots
```

# Mount the layout inside /mnt
```
cd
umount -R /mnt
mount -t btrfs -o subvol=_active/rootfs,$o_btrfs LABEL=system /mnt
mount -t btrfs -o subvol=_active/home,$o_btrfs LABEL=system /mnt/home
mount -t btrfs -o subvol=_active/tmp,$o_btrfs LABEL=system /mnt/tmp
mount -t btrfs -o subvol=_active/var,$o_btrfs LABEL=system /mnt/var
mount -t btrfs -o subvol=_snapshots,$o_btrfs LABEL=system /mnt/.snapshots
```
The remaining subvolumes will be mounted after user creation.
Finally place EFI:
```
mkdir -p /mnt/boot/efi
mount -o $o LABEL=EFI /mnt/boot/efi
```

# Arch install
Rearrange pacman mirrors:
```
pacman -Sy reflector
reflector --country Hungary  --age 12 --protocol https --sort rate --save /etc/pacman.d/mirrorlist
```
Install base system on top of /mnt (chroot) :
```
pacstrap /mnt base linux-lts linux-firmware
genfstab -L -p /mnt >> /mnt/etc/fstab
arch-chroot /mnt
pacman -Syu base-devel btrfs-progs iw gptfdisk terminus-font linux-lts zsh vim efibootmgr grub linux-firmware intel-ucode cryptsetup coreutils
```

Generate keyfile and add it to luks:
```
dd bs=512 count=4 if=/dev/random of=/crypto_keyfile.bin iflag=fullblock
chmod 0400 /crypto_keyfile.bin
cryptsetup luksAddKey /dev/disk/by-partlabel/cryptsystem /crypto_keyfile.bin
```
Add it to crypttab:
```
echo "system    /dev/disk/by-partlabel/cryptsystem  /crypto_keyfile.bin  luks" >> /etc/crypttab
```

## Other configuration based on Arch Installation Guide
```
passwd
ln -sf /usr/share/zoneinfo/Europe/Budapest /etc/localtime
hwclock --systohc
echo "en_US.UTF-8 UTF-8" >> /etc/locale.gen
locale-gen
echo "KEYMAP=us" >> /etc/vconsole.conf
echo "LANG=en_US.UTF-8" > /etc/locale.conf
echo "myhostname" > /etc/hostname
```
Edit **/etc/hosts**:
```
127.0.0.1	localhost
::1		localhost
127.0.1.1	myhostname.localdomain	myhostname
```

# Initramfs
edit /etc/mkinitcpio.conf
```
MODULES="crc32c-intel vfat i915"
HOOKS="base udev autodetect modconf block keyboard keymap encrypt filesystems btrfs systemd sd-vconsole sd-encrypt"
```
and generate intramfs:
```
mkinitcpio -p linux-lts
```

# Configure and install the boot loader

Edit **/etc/default/grub** First add<br>
`GRUB_ENABLE_CRYPTODISK=y`<br>
to the end of the file; then alter the **GRUB_CMDLINE** lines like this:
```
GRUB_CMDLINE_LINUX_DEFAULT="cryptdevice=/dev/mapper/system root=/dev/mapper/system net.ifnames=0 biosdevname=0 noresume"
GRUB_CMDLINE_LINUX="quiet"
```
(because I like good old interface names like eth0 and wlan0)

And install it into the EFI partition:
```
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=arch --recheck --debug
grub-mkconfig -o /boot/grub/grub.cfg
```
