# **Дисковая подсистема**

## **Homework3**


    уменьшить том под / до 8G
    выделить том под /home
    выделить том под /var (/var - сделать в mirror)
    для /home - сделать том для снэпшотов
    прописать монтирование в fstab (попробовать с разными опциями и разными файловыми системами на выбор)
    Работа со снапшотами:

    сгенерировать файлы в /home/
    снять снэпшот
    удалить часть файлов
    восстановиться со снэпшота
    (залоггировать работу можно утилитой script, скриншотами и т.п.)

    на нашей куче дисков попробовать поставить btrfs/zfs:

    с кешем и снэпшотами
    разметить здесь каталог /opt
___

### проверяем наличие дисков в системе:

```
# lvmdiskscan 
  /dev/VolGroup00/LogVol00 [     <37.47 GiB] 
  /dev/VolGroup00/LogVol01 [       1.50 GiB] 
  /dev/sda2                [       1.00 GiB] 
  /dev/sda3                [     <39.00 GiB] LVM physical volume
  /dev/sdb                 [      10.00 GiB] 
  /dev/sdc                 [       2.00 GiB] 
  /dev/sdd                 [       1.00 GiB] 
  /dev/sde                 [       1.00 GiB] 
  4 disks
  3 partitions
  0 LVM physical volume whole disks
  1 LVM physical volume```

###1. Уменьшаем том / до 8GB

```ставим пакет xfsdump:

yum install xfsdump

### подготавливаем место для временного перемещения

[root@lvm ~]# yum install xfsdump
[root@lvm ~]# pvcreate /dev/sdb
[root@lvm ~]# vgcreate vg_root /dev/sdb
[root@lvm ~]# lvcreate -n lv_root -l+100%FREE /dev/vg_root
[root@lvm ~]# mkfs.xfs /dev/vg_root/lv_root
[root@lvm ~]# mount /dev/vg_root/lv_root /mnt

###Переносим данные в /mnt

[root@lvm ~]# xfsdump -J - /dev/VolGroup00/LogVol00 | xfsrestore -J - /mnt

###Перенастраиваем grub для загрузки с нового раздела

[root@lvm ~]# for i in /proc/ /sys/ /dev/ /run/ /boot/; do mount --bind $i /mnt/$i; done
[root@lvm ~]# chroot /mnt/
[root@lvm ~]#  cd /boot ; for i in `ls initramfs-*img`; do dracut -v $i `echo $i|sed "s/initramfs-//g; s/.img//g"` --force; done
[root@lvm ~]# sed -i 's/VolGroup00\/LogVol00/vg_root\/lv_root/g' /boot/grub2/grub.cfg ```

Выходим из chroot и reboot
Проверяем:

$ vagrant ssh
Last login: Wed Aug 10 08:11:55 2022 from 10.0.2.2
[vagrant@lvm ~]$ lsblk 
NAME                    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                       8:0    0   40G  0 disk 
├─sda1                    8:1    0    1M  0 part 
├─sda2                    8:2    0    1G  0 part /boot
└─sda3                    8:3    0   39G  0 part 
  ├─VolGroup00-LogVol01 253:1    0  1.5G  0 lvm  [SWAP]
  └─VolGroup00-LogVol00 253:2    0 37.5G  0 lvm  
sdb                       8:16   0   10G  0 disk 
└─vg_root-lv_root       253:0    0   10G  0 lvm  /
sdc                       8:32   0    2G  0 disk 
sdd                       8:48   0    1G  0 disk 
sde                       8:64   0    1G  0 disk 

*Удаляем старый раздел и создаем заново с размером 8GB*
[vagrant@lvm ~]$ sudo -i
[root@lvm ~]#  lvremove /dev/VolGroup00/LogVol00 
Do you really want to remove active logical volume VolGroup00/LogVol00? [y/n]: y
  Logical volume "LogVol00" successfully removed
[root@lvm ~]# lvcreate -n VolGroup00/LogVol00 -L 8G /dev/VolGroup00
WARNING: xfs signature detected on /dev/VolGroup00/LogVol00 at offset 0. Wipe it? [y/n]: y
  Wiping xfs signature on /dev/VolGroup00/LogVol00.
  Logical volume "LogVol00" created.

[root@lvm ~]#  mkfs.xfs /dev/VolGroup00/LogVol00
[root@lvm ~]#  mount /dev/VolGroup00/LogVol00 /mnt
[root@lvm ~]#  xfsdump -J - /dev/vg_root/lv_root | xfsrestore -J - /mnt
[root@lvm ~]#  for i in /proc/ /sys/ /dev/ /run/ /boot/; do mount --bind $i /mnt/$i; done
[root@lvm ~]# chroot /mnt/
[root@lvm /]#  grub2-mkconfig -o /boot/grub2/grub.cfg
[root@lvm /]#  cd /boot ; for i in `ls initramfs-*img`; do dracut -v $i `echo $i|sed "s/initramfs-//g;

*Перенос /var*

[root@lvm boot]# pvcreate /dev/sdc /dev/sdd
  Physical volume "/dev/sdc" successfully created.
  Physical volume "/dev/sdd" successfully created.
[root@lvm boot]# vgcreate vg_var /dev/sdc /dev/sdd
  Volume group "vg_var" successfully created
[root@lvm boot]#  lvcreate -L 950M -m1 -n lv_var vg_var
  Rounding up size to full physical extent 952.00 MiB
  Logical volume "lv_var" created.
[root@lvm boot]#  mkfs.ext4 /dev/vg_var/lv_var
Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (4096 blocks): done
Writing superblocks and filesystem accounting information: done

[root@lvm boot]#  mount /dev/vg_var/lv_var /mnt
[root@lvm boot]#  cp -aR /var/* /mnt/ 
[root@lvm mnt]#  umount /mnt
[root@lvm mnt]# mount /dev/vg_var/lv_var /var
[root@lvm mnt]# echo "`blkid | grep var: | awk '{print $2}'` /var ext4 defaults 0 0" >> /etc/fstab

*Перезагружаемся проверяем*

[vagrant@lvm ~]$ lsblk 
NAME                     MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                        8:0    0   40G  0 disk 
├─sda1                     8:1    0    1M  0 part 
├─sda2                     8:2    0    1G  0 part /boot
└─sda3                     8:3    0   39G  0 part 
  ├─VolGroup00-LogVol00  253:0    0    8G  0 lvm  /
  └─VolGroup00-LogVol01  253:1    0  1.5G  0 lvm  [SWAP]
sdb                        8:16   0   10G  0 disk 
└─vg_root-lv_root        253:4    0   10G  0 lvm  
sdc                        8:32   0    2G  0 disk 
├─vg_var-lv_var_rmeta_0  253:2    0    4M  0 lvm  
│ └─vg_var-lv_var        253:7    0  952M  0 lvm  /var
└─vg_var-lv_var_rimage_0 253:3    0  952M  0 lvm  
  └─vg_var-lv_var        253:7    0  952M  0 lvm  /var
sdd                        8:48   0    1G  0 disk 
├─vg_var-lv_var_rmeta_1  253:5    0    4M  0 lvm  
│ └─vg_var-lv_var        253:7    0  952M  0 lvm  /var
└─vg_var-lv_var_rimage_1 253:6    0  952M  0 lvm  
  └─vg_var-lv_var        253:7    0  952M  0 lvm  /var
sde                        8:64   0    1G  0 disk 

*Выделяем том под /home*

[root@lvm ~]#  mount /dev/VolGroup00/LogVol_Home /mnt/
[root@lvm ~]#  cp -aR /home/* /mnt/
[root@lvm ~]#  rm -rf /home/*
[root@lvm ~]# umount /mnt
[root@lvm ~]#  mount /dev/VolGroup00/LogVol_Home /home/
[root@lvm ~]# echo "`blkid | grep Home | awk '{print $2}'` /home xfs defaults 0 0" >> /etc/fstab

*Создаем файлы, удаляем и восстанавливаем*

[root@lvm ~]# touch /home/file{1..20}
[root@lvm ~]#  lvcreate -L 100MB -s -n home_snap /dev/VolGroup00/LogVol_Home
  Rounding up size to full physical extent 128.00 MiB
  Logical volume "home_snap" created.
[root@lvm ~]# rm -f /home/file{11..20}
[root@lvm ~]# umount /home
[root@lvm ~]#  lvconvert --merge /dev/VolGroup00/home_snap
  Merging of volume VolGroup00/home_snap started.
  VolGroup00/LogVol_Home: Merged: 100.00%
[root@lvm ~]# mount /home
[root@lvm ~]# ls /home/
file1  file10  file11  file12  file13  file14  file15  file16  file17  file18  file19  file2  file20  file3  file4  file5  file6  file7  file8  file9


