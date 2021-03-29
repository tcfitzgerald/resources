# Arch Linux install notes

using SeaBIOS in qemu on proxmox

### List disks

```lsblk```

### using the data returned from lsblk, use fdisk to add partitions
```
fdisk /dev/sda
```

### new partition 1 - swap
```
n
p
1
<default>
+2G

t
82
```

### new partion 2 - /
```
n
p
2
<default>
<default>
<default>
```

### write
```
w
```

### format /
```
mkfs.ext4 /dev/sda2
```

### initialize swap
```
mkswap /dev/sda1
```

### mount /
```
mount /dev/sda2 /mnt
```

### enable swap
```
swapon /dev/sda1
```

### install essential packages
```
pacstrap /mnt base linux linux-firmware nano dhcpcd
```

### fstab
```
genfstab -U /mnt >> /mnt/etc/fstab
```

### chroot into /
```
arch-chroot /mnt
```

### timezone
```
ln -sf /usr/share/zoneinfo/America/Chicago /etc/localtime
hwclock --systohc
```

### localization
```
locale-gen
nano /etc/locale.gen (uncomment en_US locales)
nano /etc/locale.conf (add LANG=en_US.UTF-8)
```

### hostname
```
nano /etc/hostname

```

### rebuild initramfs
```
mkinitcpio -P
```

### set root password
```
passwd
```

### install and setup grub
```
pacman -S grub
grub-install --target=i386-pc /dev/sda
grub-mkconfig -o /boot/grub/grub.cfg
```

### reboot
```
exit
unmount -R /mnt
reboot
```
### post installation

?
