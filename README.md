# Подробная инструкция по установке Gentoo(UEFI + OpenRC + dmcrypt + LUKS).

## 1.Подготовительные действия
**Создание загрузочной флешки:**  
**Linux**:  
`dd if=gentoo.iso of=/dev/sdX`  
**Windows**:  
Через rufus.  
  
## 2.Вход в систему, подготовка дисков и монтирование разделов

`date 032814552016` # Установить дату 03.28.2016 14:55  

`cfdisk /dev/sdX` # Размечаем диск 
Должно получиться что-то такое:  
/dev/sdX1      2048   1054719   1052672  514M EFI System  
/dev/sdX2   1054720  34256895  33202176 15.9G Linux swap  
/dev/sdX3  34256896 234441614 200184719 95.5G Linux filesystem  

**Scheme for mount**:  
/dev/sdX1 -> /boot #vfat  
/dev/sdX2 -> /boot/efi #vfat  
/dev/sdX3 -> swap
/dev/sdX4 -> /dev/mapper/crypto #ext4  

`mkfs.vfat /dev/sdX1`  
`mkfs.vfat /dev/sdX2`  
`mkswap /dev/sdX3`  
`swapon /dev/sdX3`  
`cryptsetup -s 512 luksFormat /dev/sdX4`  
`cryptsetup luksOpen /dev/sdX4 crypto`
`mkfs.ext4 /dev/mapper/crypto`  
 
`mount /dev/mapper/crypto /mnt/gentoo`  
`cd /mnt/gentoo`  

`cd /mnt/gentoo`  
`links www.gentoo.org/downloads` # Качаем stage3  
`tar xvpf stage3-*.xz --xattrs`  

`mirrorselect -i -o >> /mnt/gentoo/etc/portage/make.conf`  
  
`mkdir /mnt/gentoo/etc/portage/repos.conf`  
`cp /mnt/gentoo/usr/share/portage/config/repos.conf /mnt/gentoo/etc/`  
`cp -L /etc/resolv.conf /mnt/gentoo/etc/`  
  
`mount --types proc /proc /mnt/gentoo/proc`  
`mount --rbind /sys /mnt/gentoo/sys`  
`mount --make-rslave /mnt/gentoo/sys`  
`mount --rbind /dev /mnt/gentoo/dev`  
`mount --make-rslave /mnt/gentoo/dev`  
  
## 3.Вход в OS
`chroot /mnt/gentoo /bin/bash`  
`source /etc/profile`  
`export PS1="(chroot) ${PS1}"`  
  
## 4. Настройка Gentoo
`nano /etc/portage/make.conf`
Добавляем строки :  
*CFLAGS="-march=native -O2 -pipe"*  
*CXXFLAGS="${CFLAGS}"*  
*CHOST="x86_64-pc-linux-gnu"*  
*ACCEPT_KEYWORDS="~amd64"*  
*GRUB_PLATFORMS="efi-64"*  
  
`eselect profile list`  
`eselect profile set (X)`  
`echo "Europe/Moscow" > /etc/timezone`  
`emerge --config sys-libs/timezone-data`  
`nano /etc/locale.gen`  
Раскомментировать:  
*en_US ISO-8859-1*  
*en_US.UTF-8 UTF-8*  
`locale-gen`  
`eselect locale list`  
`eselect locale set (X)`  
`source /etc/profile`  
`export PS1="(chroot) ${PS1}"`
`nano /etc/fstab`  
/dev/sdX1 /boot vfat noauto,noatime 1 2  
/dev/sdX2 /boot/efi vfat noauto,noatime 1 2  
/dev/sdX3 none swap sw 0 0  
/dev/mapper/crypto / ext4 defaults 0 0  
crypto /dev/sdb3 none  
`nano /etc/default/grub`  
GRUB_TIMEOUT=15
GRUB_CMDLINE_LINUX="crypt_root=UUID="f2389a1f-de2c-4440-8747-05b26af21e39" real_root=UUID="0fc9de2b-fb0f-4892-b9a3-80d19a7e193c" rootfstype=ext4" #crypt-root=<UUID /dev/sdX4> real_root=<UUID /dev/mapper/root (NOT crypto!)>
`nano /etc/conf.d/dmcrypt`  
target=/dev/mapper/crypto  
source='/dev/sdb3'  
Чтобы узнать UUID:  
`blkid |grep sda2`  
`blkid |grep sda3`  
  
## 5. Установка Grub:2 и компиляция ядра
`emerge --sync`  
`echo -e "sys-boot/grub mount\nsys-apps/util-linux static-libs" > /etc/portage/package.use/myuse`  
`emerge -v sys-kernel/genkernel sys-kernel/linux-firmware sys-boot/os-prober dev-libs/libisoburn sys-fs/mdadm grub gentoo-   sources net-misc/dhcpcd app-admin/syslog-ng vixie-cron sys-fs/e2fsprogs`  
`rc-update add mdraid boot` 
`rc-update add dmcrypt`  
`genkernel all`  
  
`mkdir /boot/efi/`  
`mount /dev/sdX1 /boot/`  
`mount /dev/sdX2 /boot/efi`
`grub-install --efi-directory=/boot/efi/ --target=x86_64-efi /dev/sda`  
`mount -o remount,rw /sys/firmware/efi/efivars`  
`grub-install --efi-directory=/boot/efi/ --target=x86_64-efi /dev/sda`  
`grub-mkconfig -o /boot/grub/grub.cfg`  
  
## 6. Установка пароля root и перезагрузка системы, на этом этапе отключаем флешку от ПК
`passwd`  
`exit`  
`cd`  
`umount /mnt/gentoo/boot/efi/`  
`umount -l /mnt/gentoo/dev{/shm,/pts,}`  
`umount -R /mnt/gentoo`  
  
## 8. Установка дополнительного софта + XFCE + XORG
