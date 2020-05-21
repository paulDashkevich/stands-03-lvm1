# stands-03-lvm

Стенд для домашнего занятия "Файловые системы и LVM"

1. Уменьшение раздела до 8Gb
        используем окружение chroot:
        поскольку фс xfs, потребовалась утилита xfsdump
        из дисков в ВМ собрался временный раздел для сохранения дампа
        pvcreate /dev/sdd  +  vgcreate vg00 /dev/sdd  +  lvcreate -L 10G -n lv00 vg00
        натягиваем фс на созданную lv    mkfs.xfs /dev/vg00/lv00
        монтируем временный раздел mount /dev/mapper/vg00-lv00 /mnt
        дампим и восстанавливаем дамп в /mnt наш /     xfsdump -J - /dev/VolGroup00/LogVol00 | xfsrestore -J - /mnt
        заходим в окружение chroot
        for i in /proc/ /sys/ /dev/ /run/ /boot/; do mount --bind $i /mnt/$i; done
        chroot /mnt/
        пишем новый загрузчик  grub2-mkconfig -o /boot/grub2/grub.cfg
        обновляем образы загрузки cd /boot ; for i in `ls initramfs-*img`; do dracut -v $i `echo $i|sed "s/initramfs-//g; s/.img//g"` --force; done
        выходим exit и ребутм ВМ reboot
        удаляем изначальную lv 39G lvremove /dev/VolGroup00-LogVol00    - подтверждаем y
        создаём новую lv 8G по заданию: lvcreate -n LogVol00 8G VolGroup00 и вайпим сигнатуры xfs 'y'
        натягиваем fs: mkfs.xfs /dev/VolGroup00/LogVol00
        монтируем mount /dev/VolGroup00/LogVol00 /mnt и задействуем утилиту дампа xfsdump -J - /dev/vg00/lv00 | xfsrestore -J - /mnt
        обратно к chroot
        for i in /proc/ /sys/ /dev/ /run/ /boot/; do mount --bind $i /mnt/$i; done
        chroot /mnt/
        загрузчик grub2-mkconfig -o /boot/grub2/grub.cfg
        Образы загрузки: cd /boot ; for i in `ls initramfs-*img`; do dracut -v $i `echo $i|sed "s/initramfs-//g; s/.img//g"` --force; done
        выходим и ребутим ВМ exit; reboot
        удаляем временный lv: lvremove /dev/vg00-lv00 и подтверждаем акт насилия 'y' + vgremove vg00 + pvremove /dev/sdd
2. Выделяем том под /home и /var
        создаём том: lvcreate -L 1G -n LVHome /dev/mapper/VolGroup00
        создаём том: lvcreate -L 2G -n VARLV /dev/mapper/VARLV
        монтируем: mount /dev/VolGroup00/LVHome /mnt
        монтируем: mount /dev/VolGroup00/VARLV /mnt
        копируем содержимое папки /home   cp -pr --preserve=context /home/. /mnt
        копируем содержимое папки /var   cp -pr --preserve=context /var/. /mnt
        удаляем содержимое папки /home  rm -r -f /home/*
        удаляем содержимое папки /var  rm -r -f /var/* (удаляется не всё, но ради эксперимента)
        монтируем свежий том: mount /dev/VolGroup/LVhome /home
        монтируем свежий том: mount /dev/VolGroup/VARLV /var
        смотрим UUID blkid и прописываем монтирование при загрузке: vi /etc/fstab

        /dev/mapper/VolGroup00-LogVol00 /                     xfs     defaults        0 0
        UUID=570897ca-e759-4c81-90cf-389da6eee4cc /boot       xfs     defaults        0 0
        UUID=557a968a-8d39-479e-b365-8acb5604fc97 /var        xfs     defaults        0 0
        UUID=3a98d5dd-f8a2-4ccc-aa90-8b4e2f10c645 /home       xfs     defaults        0 0
        UUID=fee24c94-daa5-44f5-a0b9-cb389efa14d4 swap        swap    defaults        0 0
3. Создание зеркального тома /var
        для выполнения этого пункта ДЗ достаточно просто конвертировать наш lv 'VARLV' в зеркало
         lvconvert -m1 /dev/VolGroup00/VARLV
4. Создаю том BTRFS
	mkfs.btrfs /dev/sdc
	монтирую mount /dev/sdc /mnt
	размечаю /opt: копирую /opt   cp -pr --preserve=context /opt/. /mnt
	добавляю запись в fstab по UUID тома с BTRFS
	#
# /etc/fstab
# Created by anaconda on Sat May 12 18:50:26 2018
#
# Accessible filesystems, by reference, are maintained under '/dev/disk'
# See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info
#
/dev/mapper/VolGroup00-LogVol00 /                     xfs     defaults        0 0
UUID=570897ca-e759-4c81-90cf-389da6eee4cc /boot       xfs     defaults        0 0
UUID=557a968a-8d39-479e-b365-8acb5604fc97 /var        xfs     defaults        0 0
UUID=3a98d5dd-f8a2-4ccc-aa90-8b4e2f10c645 /home       xfs     defaults        0 0
UUID=36f38353-28d1-4b13-ae7a-7c6aa2913059 /opt        btrfs   defaults        0 0
UUID=fee24c94-daa5-44f5-a0b9-cb389efa14d4 swap        swap    defaults        0 0

	после перезагрузки ВМ видим, что всё ОК
# lsblk
NAME                          MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                             8:0    0   40G  0 disk
├─sda1                          8:1    0    1M  0 part
├─sda2                          8:2    0    1G  0 part /boot
└─sda3                          8:3    0   39G  0 part
  ├─VolGroup00-LogVol00       253:0    0    8G  0 lvm  /
  ├─VolGroup00-LVhome         253:1    0    1G  0 lvm  /home
  ├─VolGroup00-VARLV_rmeta_0  253:2    0   32M  0 lvm
  │ └─VolGroup00-VARLV        253:6    0    2G  0 lvm  /var
  └─VolGroup00-VARLV_rimage_0 253:3    0    2G  0 lvm
    └─VolGroup00-VARLV        253:6    0    2G  0 lvm  /var
sdb                             8:16   0   10G  0 disk
├─VolGroup00-VARLV_rmeta_1    253:4    0   32M  0 lvm
│ └─VolGroup00-VARLV          253:6    0    2G  0 lvm  /var
└─VolGroup00-VARLV_rimage_1   253:5    0    2G  0 lvm
  └─VolGroup00-VARLV          253:6    0    2G  0 lvm  /var
sdc                             8:32   0    2G  0 disk /opt
sdd                             8:48   0    1G  0 disk
└─sdd1                          8:49   0 1023M  0 part [SWAP]
sde                             8:64   0    1G  0 disk

			
