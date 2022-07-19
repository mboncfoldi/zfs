# Steps to install Kubuntu 22.04 desktop on zfs

no bpool or boot partition - only rpool +compression,encryption
no grub - systemd-boot

Start live iso --> terminal

*sudo -i

*add-apt-repository multiverse
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

adduser user0
usermod -aG adm,cdrom,dip,lpadmin,plugdev,sudo user0

mkdir -p /boot/efi/loader/entries
nano /boot/efi/loader/loader.conf

default ubuntu.conf
timeout 1
console-mode auto

blkid

nano /etc/kernel/postinst.d/zz-update-systemd-boot-zfs

#!/bin/bash
*# This is a simple kernel hook to populate the systemd-boot entries
*# whenever kernels are added or removed.

#ZFSROOT="CHANGEME E.G. data/dpool/buster"
#UUID="CHANGEME"
#OS="CHANGEME E.G. Debian"
#OSV="CHANGEME E.G. Buster"
#ROOTTYPE="CHANGEME; ONLY AFFECTS OS DISPLAYNAME IN MENU E.G. zfs"
#KERNELPARAMETERS="CHANGEME; Add extra options here e.g. quiet splash; leave empty if not using any options"

ZFSROOT="rpool/ROOT"
UUID="3720602209614833149"
OS="Ubuntu"
OSV="Jammy"
ROOTTYPE="zfs"
KERNELPARAMETERS="quiet splash"

# Our kernels.
KERNELS=()
FIND="find /boot -maxdepth 1 -name 'vmlinuz-*' -type f -not -name '*.dpkg-tmp' -print0 | sort -Vrz"
while IFS= read -r -u3 -d $'\0' LINE; do
	KERNEL=$(basename "${LINE}")
	KERNELS+=("${KERNEL:8}")
done 3< <(eval "${FIND}")

*# There has to be at least one kernel.
if [ ${#KERNELS[@]} -lt 1 ]; then
	echo -e "\e[2msystemd-boot\e[0m \e[1;31mNo kernels found.\e[0m"
	exit 1
fi

# Perform a nuclear clean to ensure everything is always in
# perfect sync.
rm /boot/efi/loader/entries/${OS}-${OSV}-${ROOTTYPE}.conf
rm -rf /boot/efi/EFI/${OS}-${OSV}-${ROOTTYPE}
mkdir /boot/efi/EFI/${OS}-${OSV}-${ROOTTYPE}

# Copy the latest kernel files to a consistent place so we can
# keep using the same loader configuration.
LATEST="${KERNELS[@]:0:1}"
echo -e "\e[2msystemd-boot\e[0m \e[1;32m${LATEST}\e[0m"
for FILE in config initrd.img System.map vmlinuz; do
    cp "/boot/${FILE}-${LATEST}" "/boot/efi/EFI/${OS}-${OSV}-${ROOTTYPE}/${FILE}"
    cat << EOF > /boot/efi/loader/entries/${OS}-${OSV}-${ROOTTYPE}.conf
title   ${OS} ${OSV} (${ROOTTYPE})
linux   /EFI/${OS}-${OSV}-${ROOTTYPE}/vmlinuz
initrd  /EFI/${OS}-${OSV}-${ROOTTYPE}/initrd.img
options root=zfs=${ZFSROOT} boot=zfs ${KERNELPARAMETERS}
EOF
done

# Copy any legacy kernels over too, but maintain their version-
# based names to avoid collisions.
if [ ${#KERNELS[@]} -gt 1 ]; then
	LEGACY=("${KERNELS[@]:1}")
	for VERSION in "${LEGACY[@]}"; do
	    echo -e "\e[2msystemd-boot\e[0m \e[1;32m${VERSION}\e[0m"
	    for FILE in config initrd.img System.map vmlinuz; do
	        cp "/boot/${FILE}-${VERSION}" "/boot/efi/EFI/${OS}-${OSV}-${ROOTTYPE}/${FILE}-${VERSION}"
	        cat << EOF > /boot/efi/loader/entries/${OS}-${OSV}-${ROOTTYPE}-${VERSION}.conf
title   ${OS} ${OSV} (${ROOTTYPE}) ${VERSION}
linux   /EFI/${OS}-${OSV}-${ROOTTYPE}/vmlinuz-${VERSION}
initrd  /EFI/${OS}-${OSV}-${ROOTTYPE}/initrd.img-${VERSION}
options root=zfs=${ZFSROOT} boot=zfs ${KERNELPARAMETERS}
EOF
	    done
	done
fi

# Success!
echo -e "\e[2m---\e[0m"
exit 0

chown root: /etc/kernel/postinst.d/zz-update-systemd-boot-zfs
chmod 0755 /etc/kernel/postinst.d/zz-update-systemd-boot-zfs

cd /etc/kernel/postrm.d/ && ln -s ../postinst.d/zz-update-systemd-boot-zfs zz-update-systemd-boot-zfs

[ -d "/etc/initramfs/post-update.d" ] || mkdir -p /etc/initramfs/post-update.d

cd /etc/initramfs/post-update.d/ && ln -s ../../kernel/postinst.d/zz-update-systemd-boot-zfs zz-update-systemd-boot-zfs

bootctl --path=/boot/efi install
efibootmgr -v

update-initramfs -c -k all

nano /etc/apt/preferences.d/no-boot-loaders

Package: grub*
Pin: release *
Pin-Priority: -1

Package: grub*:i386
Pin: release *
Pin-Priority: -1

Package: lilo
Pin: release *
Pin-Priority: -1

Package: lilo:i386
Pin: release *
Pin-Priority: -1

Package: os-prober
Pin: release *
Pin-Priority: -1

Package: os-prober:i386
Pin: release *
Pin-Priority: -1

exit

umount /mnt/install/boot/efi
mount | grep -v zfs | tac | awk '/\/mnt/ {print $3}' | xargs -i{} umount -lf {}
umount /mnt/install
zpool export -a

reboot
