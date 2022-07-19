# Steps to install Kubuntu 22.04 destop on zfs
#
# no bpool or boot partition - only rpool +compression,encryption 
# no grub - systemd-boot
#
# Start live iso --> terminal
#
sudo -i

add-apt-repository multiverse
apt update
apt install debootstrap arch-install-scripts zfs-initramfs gdisk
systemctl stop zed

blkdiscard -f /dev/vda
sgdisk --zap-all /dev/vda
sgdisk -n1:1M:-1 -t1:EF00 -c1:EFI /dev/vda
sgdisk --print /dev/vda
sgdisk -n2:8388574:8388574 -t2:8300 -c2:key1 /dev/vda
sgdisk --print /dev/vda

blkdiscard -f /dev/vdb
sgdisk --zap-all /dev/vdb
sgdisk -n1:1M:0 -t1:BF00 -c1:rpool /dev/vdb
sgdisk --print /dev/vdb

tr -d '\n' < /dev/urandom | dd of=/dev/disk/by-partlabel/key1

zpool create -o ashift=12 -o autotrim=on -O acltype=posixacl -O dnodesize=auto -O normalization=formD -O atime=off -O xattr=sa -O compression=lz4 -O encryption=aes-256-gcm -O keyformat=passphrase -O keylocation=file:///dev/disk/by-partlabel/key1 -O canmount=off -O mountpoint=none -R /mnt/install rpool /dev/disk/by-partlabel/rpool

zfs create -o canmount=noauto -o mountpoint=/ rpool/ROOT
zfs mount rpool/ROOT
zfs set devices=off rpool

mkfs.vfat -F 32 -s 1 /dev/disk/by-partlabel/EFI
mkdir -p /mnt/install/boot/efi
mount /dev/disk/by-partlabel/EFI /mnt/install/boot/efi/

debootstrap jammy /mnt/install/
sed '/cdrom/d' /etc/apt/sources.list > /mnt/install/etc/apt/sources.list
cp /etc/netplan/*.yaml /mnt/install/etc/netplan/

genfstab -U /mnt/install/ >> /mnt/install/etc/fstab
nano /mnt/install/etc/fstab

arch-chroot /mnt/install/

apt update
apt install language-pack-en-base
dpkg-reconfigure locales
dpkg-reconfigure tzdata
apt install --no-install-recommends linux-image-generic linux-headers-generic
apt install efibootmgr zfs-initramfs nano bash-completion
apt install kubuntu-desktop
