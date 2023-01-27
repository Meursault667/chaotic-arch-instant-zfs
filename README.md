NOT READY

# chaotic-arch-instant-zfs

```bash
#conf________________________________________________________________________|
export DISK="/dev/disk/by-id/ata-SAMSUNG_MZ7LN256HCHP-000L7_S20HNXAGA36539"
export EFI="$DISK"-part1
export POOL="$DISK"-part2

#----------------------------------------------------------------------------|

#Partition___________________________________________________________________|
wipefs -a "$DISK"
sgdisk --zap-all "$DISK"

sgdisk -n 1:1m:+512m -t 1:ef00 "$DISK"
sgdisk -n 2:0:-10m -t 2:bf00 "$DISK"


#ZFS-Filesystem______________________________________________________________|

zpool create -f -o ashift=12         \
             -O acltype=posixacl       \
             -O relatime=on            \
             -O xattr=sa               \
             -O dnodesize=legacy       \
             -O normalization=formD    \
             -O mountpoint=none        \
             -O canmount=off           \
             -O devices=off            \
             -R /mnt                   \
             -O compression=lz4        \
             -O encryption=aes-256-gcm \
             -O keyformat=passphrase   \
             -O keylocation=prompt     \
             zroot "$POOL"

zfs create -o mountpoint=none zroot/ROOT
zfs create -o mountpoint=/ -o canmount=noauto zroot/ROOT/default
zfs create -o mountpoint=none zroot/data
zfs create -o mountpoint=/home zroot/data/home

zpool set cachefile=/etc/zfs/zpool.cache zroot
zpool set bootfs=zroot/ROOT/default zroot

zpool export zroot

#mount Filesystem____________________________________________________________|
zpool import -d /dev/disk/by-id -R /mnt zroot -N
# zpool import 9876543212345678910 -R /mnt zroot
zfs load-key -L prompt zroot
zfs mount zroot/ROOT/default
zfs mount -a

mount | grep mnt

#efi_________________________________________________________________________|
mkfs.vfat $EFI
mount --mkdir $EFI /mnt/efi


#base-install________________________________________________________________|
pacstrap -K -c /mnt    \
    base            \
    git             \
    neovim          \
    nnn             \
    efibootmgr


#base-conf___________________________________________________________________|
genfstab -U -p /mnt >> /mnt/etc/fstab
cp /etc/hostid /mnt/etc/hostid
mkdir /mnt/etc/zfs
cp /etc/zfs/zpool.cache /mnt/etc/zfs/zpool.cache


#chroot______________________________________________________________________|
systemd-nspawn -D /mnt
passwd
rm -rf /etc/securetty /usr/share/factory/etc/securetty
nvim /usr/lib/tmpfiles.d/arch.conf
# delete entry /etc/securetty"
logout
systemd-nspawn -b -D /mnt


#chroot-conf_________________________________________________________________|
systemd-firstboot

cat << EOF >> /etc/pacman.conf

[archzfs]
# Origin Server - France
Server = http://archzfs.com/$repo/x86_64
# Mirror - Germany
Server = http://mirror.sum7.eu/archlinux/archzfs/$repo/x86_64
# Mirror - Germany
Server = https://mirror.biocrafting.net/archlinux/archzfs/$repo/x86_64

EOF

zfsKey="DDF7DB817396A49B2A2723F7403BD972F75D9D76"

pacman-key --recv-keys "$zfsKey"
pacman-key --finger "$zfsKey"
pacman-key --lsign-key "$zfsKey"

#chaotic 4 downgrade

cat > /etc/mkinitcpio.conf <<EOF
MODULES=(i915 intel_agp)
BINARIES=()
FILES=()
HOOKS=(base udev autodetect modconf block keyboard keymap zfs filesystems)
COMPRESSION="zstd"
EOF

cat > /etc/mkinitcpio.d/linux.preset <<"EOF"
ALL_config="/etc/mkinitcpio.conf"
ALL_kver="/boot/vmlinuz-linux"
PRESETS=('default')
default_image="/boot/initramfs-linux.img"
EOF



#instant-install_____________________________________________________________|
git clone https://github.com/instantOS/instantARCH
cd instantARCH
bash topinstall.sh
pacman -R linux-lts

#zfs-install_________________________________________________________________|
pacman -S zfs-linux zfs-utils linux-headers linux-firmware


#ZFSBootMenu_________________________________________________________________|
yay -S zfsbootmenu-efi-bin
zfs set org.zfsbootmenu:commandline="quiet loglevel=4 rw" zroot/ROOT
zfs set org.zfsbootmenu:keysource="zroot/ROOT/default" zroot
efibootmgr -c -d "$DISK" -p 1 -L "ZFSBootMenu" -l '\EFI\zbm\zfsbootmenu-release-vmlinuz-x86_64.EFI'


#Enable Services_____________________________________________________________|
systemctl enable zfs.target
systemctl enable zfs-import-cache.service
systemctl enable zfs-mount.service
systemctl enable zfs-import.target

mkdir /etc/zfs/zfs-list.cache
touch /etc/zfs/zfs-list.cache/zroot
zed -F &
cat /etc/zfs/zfs-list.cache/zroot

# -> if empty
# zfs set canmount=noauto   zroot/ROOT/default

zgenhostid $(hostid)


#regen initramfs_____________________________________________________________|
mkinitcpio -P 

exit
umount -n -R /mnt
zpool export zroot
reboot


#swap____________________________________________________________optional

zfs create -V 9G -b $(getconf PAGESIZE) -o compression=zle \
    -o logbias=throughput -o sync=always \
    -o primarycache=metadata -o secondarycache=none \
    -o com.sun:auto-snapshot=false zroot/swap

mkswap -f /dev/zvol/zroot/swap
echo /dev/zvol/zroot/swap none swap discard 0 0 >> /etc/fstab
echo RESUME=none > /etc/initramfs-tools/conf.d/RESUME

swapon -av



#Trouble____________________________________________________________optional

## boot
pacman -Si zfs-linux-lts
pacman -Qi linux-lts

# wenn linux-lts [version] > zfs-linux-lts [depends on] 
# add chaotic aur 
pacman -S downgrade
downgrade linux-lts
pacman -S zfs-linux-lts

zpool set cachefile=none zroot

rm -rf /etc/hostid

mkinitcpio -P 

## umount/mount
DISK="/dev/disk/by-id/ata-SAMSUNG_MZ7LN256HCHP-000L7_S20HNXAGA36539"

zpool export -a
zpool import -N -R /mnt zroot
zfs load-key -a
zfs mount rpool/ROOT/default
zfs mount -a
mount "$DISK"-part1 /mnt/efi

## chaotic aur
pacman-key --recv-key FBA220DFC880C036 --keyserver keyserver.ubuntu.com
pacman-key --lsign-key FBA220DFC880C036
pacman -U 'https://cdn-mirror.chaotic.cx/chaotic-aur/chaotic-keyring.pkg.tar.zst' 'https://cdn-mirror.chaotic.cx/chaotic-aur/chaotic-mirrorlist.pkg.tar.zst'

Append (adding to the end of the file) to /etc/pacman.conf:
[chaotic-aur]
Include = /etc/pacman.d/chaotic-mirrorlist 

```
