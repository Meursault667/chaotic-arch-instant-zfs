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
             -O compression=zstd       \
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
# zpool import yourstupidid -R /mnt zroot
# get your stupid id with "zpool import"
zfs load-key -L prompt zroot
zfs mount zroot/ROOT/default
zfs mount -a

# o k ?______________________________________________________________________|
zpool status -v
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
    efibootmgr      \
    downgrade       \


#base-conf___________________________________________________________________|
genfstab -U -p /mnt >> /mnt/etc/fstab
cp /etc/hostid /mnt/etc/hostid
mkdir /mnt/etc/zfs
cp /etc/zfs/zpool.cache /mnt/etc/zfs/zpool.cache

hwclock --systohc
systemctl enable systemd-timesyncd --root=/mnt

rm -f /mnt/etc/localtime
systemd-firstboot --root=/mnt --prompt --force
zgenhostid -f -o /mnt/etc/hostid


  
#chroot______________________________________________________________________|
systemd-nspawn -D /mnt
passwd
rm -rf /etc/securetty /usr/share/factory/etc/securetty
nvim /usr/lib/tmpfiles.d/arch.conf
# delete entry /etc/securetty"
logout
systemd-nspawn -b -D /mnt

#instant-install_____________________________________________________________|
git clone https://github.com/instantOS/instantARCH
cd instantARCH
bash topinstall.sh
pacman -R linux-lts

#chroot-conf_________________________________________________________________|
cat < EOF >> /etc/pacman.conf

[archzfs]
# Origin Server - France
Server = http://archzfs.com/$repo/x86_64
# Mirror - Germany
Server = http://mirror.sum7.eu/archlinux/archzfs/$repo/x86_64
# Mirror - Germany
Server = https://mirror.biocrafting.net/archlinux/archzfs/$repo/x86_64

[endeavouros]
SigLevel = PackageRequired
Include = /etc/pacman.d/endeavouros-mirrorlist

EOF

zfsKey="DDF7DB817396A49B2A2723F7403BD972F75D9D76"


pacman-key --recv-keys "$zfsKey"
pacman-key --lsign-key "$zfsKey"


enosKey="003DB8B0CB23504F"

cat < EOF >> /etc/pacman.d/endeavouros-mirrorlist
## Germany
Server = https://mirror.alpix.eu/endeavouros/repo/$repo/$arch
Server = https://de.freedif.org/EndeavourOS/repo/$repo/$arch
Server = https://mirror.moson.org/endeavouros/repo/$repo/$arch

## Netherlands
Server = https://mirror.erickochen.nl/endeavouros/repo/$repo/$arch

## Sweden
Server = https://ftp.acc.umu.se/mirror/endeavouros/repo/$repo/$arch
Server = https://mirror.linux.pizza/endeavouros/repo/$repo/$arch

## Canada
Server = https://ca.gate.endeavouros.com/endeavouros/repo/$repo/$arch

## China
Server = https://mirrors.tuna.tsinghua.edu.cn/endeavouros/repo/$repo/$arch

## Vietnam
Server = https://mirror.freedif.org/EndeavourOS/repo/$repo/$arch

## Github
Server = https://raw.githubusercontent.com/endeavouros-team/repo/master/$repo/$arch

EOF

pacman-key --keyserver keyserver.ubuntu.com -r "$enosKey"
sudo pacman-key --lsign "$enosKey"



#zfs-install_________________________________________________________________|
pacman -S zfs-linux zfs-utils linux-headers linux-firmware 

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


#ZFSBootMenu_________________________________________________________________|
pacman -S cpanminus kexec-tools fzf util-linux --noconfirm

git clone --depth=1 https://github.com/zbm-dev/zfsbootmenu/
cd zfsbootmenu
make
make install
cpanm --notest --installdeps .
mkdir -p /efi/EFI/ZBM

cat > /etc/zfsbootmenu/mkinitcpio.conf <<"EOF"
MODULES=()
BINARIES=()
FILES=()
HOOKS=(base udev autodetect modconf block keyboard keymap)
COMPRESSION="zstd"
EOF

cat > /etc/zfsbootmenu/config.yaml <<EOF
Global:
  ManageImages: true
  BootMountPoint: /efi
  InitCPIO: true
Components:
  Enabled: false
EFI:
  ImageDir: /efi/EFI/ZBM
  Versions: false
  Enabled: true
Kernel:
  CommandLine: ro quiet loglevel=0 zbm.import_policy=hostid
  Prefix: vmlinuz
EOF

zfs set org.zfsbootmenu:commandline="rw quiet nowatchdog rd.vconsole.keymap=$keymap" $zpoolname/ROOT/"$root_dataset"

generate-zbm


efibootmgr --disk "$DISK" \
      --part 1 \
      --create \
      --label "ZFSBootMenu Backup" \
      --loader "\EFI\ZBM\vmlinuz-backup.efi" \
      --verbose
efibootmgr --disk "$DISK" \
      --part 1 \
      --create \
      --label "ZFSBootMenu" \
      --loader "\EFI\ZBM\vmlinuz.efi" \
      --verbose


#Enable Services_____________________________________________________________|
zpool set cachefile=/etc/zfs/zpool.cache zroot
systemctl enable zfs-import-cache.service
systemctl enable zfs-import.target

mkdir /etc/zfs/zfs-list.cache
ln -s /usr/lib/zfs/zed.d/history_event-zfs-list-cacher.sh /etc/zfs/zed.d
systemctl enable zfs.target
systemctl enable zfs-zed.service
touch /etc/zfs/zfs-list.cache/zroot


# -> if empty
# zfs set canmount=noauto   zroot/ROOT/[...]

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

#Network_________________________________________________________________|



#Trouble____________________________________________________________optional

## boot
pacman -Si zfs-linux-lts
pacman -Qi linux-lts

# wenn linux [version] > zfs-linux [depends on] 
# add chaotic aur 
pacman -S downgrade
downgrade linux
pacman -S zfs-linux

# change /etc/pacman.conf
IgnorePkg

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

## endeavor repo


## chaotic aur
pacman-key --recv-key FBA220DFC880C036 --keyserver keyserver.ubuntu.com
pacman-key --lsign-key FBA220DFC880C036
pacman -U 'https://cdn-mirror.chaotic.cx/chaotic-aur/chaotic-keyring.pkg.tar.zst' 'https://cdn-mirror.chaotic.cx/chaotic-aur/chaotic-mirrorlist.pkg.tar.zst'

Append (adding to the end of the file) to /etc/pacman.conf:
[chaotic-aur]
Include = /etc/pacman.d/chaotic-mirrorlist 

# DRACUT

cat > /etc/dracut.conf.d/zfs.conf <<"EOF"
hostonly="yes"
nofsck="yes"
compress="zstd"
add_drivers+=" i915 "
add_dracutmodules+=" zfs "
omit_dracutmodules+=" btrfs "

dracut
# - [ ] dracut-hook
# - [ ]
```
