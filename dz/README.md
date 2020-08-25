Развернем vagrant, выключаем VM.

Для решение задач, нужно сделать по умолчанию, логинеться под root.

Добавляем в наш Vagrantfile следующее:

[root@lvm ~]$ vim ./Vagrantfile

config.ssh.username = 'root'
config.ssh.password = 'vagrant'
config.ssh.insert_key = 'true'

1. Уменьшить том под / до 8G

1.1.Проверим наличие дисков:

[root@lvm ~]$ lsblk


1.2. Свободное место на диске sdb

Для решение задачи установим утилиту xfsdump

[root@lvm ~]# yum update

[root@lvm ~]# yum install xfsdump -y


1.3. Подготовим временный раздел для корневого тома:

[root@lvm ~]# pvcreate /dev/sdb

[root@lvm ~]# vgcreate vg_root /dev/sdb

[root@lvm ~]# lvcreate -n lv_root -l +100%FREE /dev/vg_root


1.4. Создадим файловую систему XFS и смонтируем ее в каталог /mnt

[root@lvm ~]# mkfs.xfs /dev/vg_root/lv_root

[root@lvm ~]# mount /dev/vg_root/lv_root /mnt

[vagrant@lvm ~]$ lsblk


1.5. Сдампим содержимое текущего корневого раздела в наш временный:

[root@lvm ~]# xfsdump -J - /dev/VolGroup00/LogVol00 | xfsrestore -J - /mnt

Проверим содержимое каталога /mnt

[root@lvm ~]# ls /mnt

1.6. Заходим в окружение chroot нашего временного корня:

[root@lvm ~]# for i in /proc/ /sys/ /dev/ /run/ /boot/; do mount --bind $i /mnt/$i; done

[root@lvm ~]# chroot /mnt/

1.7. Запишем новый загрузчик:

[root@lvm ~]# grub2-mkconfig -o /boot/grub2/grub.cfg

1.8. Обновляем образы загрузки:

[root@lvm ~]# cd /boot ; for i in `ls initramfs-*img`; do dracut -v $i `echo $i|sed "s/initramfs-//g; s/.img//g"` --force; done

1.9. Открываем конфигурационный файл grub:

[root@lvm ~]# vi /boot/grub2/grub.cfg

Меняем все значения
/LogVol00 на rd.lvm.lv=vg_root/lv_root

1.10. Выходим из окружения chroot и перезагружаем компьютер

Проверяем что загрузился временный раздел:

[root@lvm ~]# lsblk

И так, у нас есть раздел, который нужно уменьшить и который теперь не примонтирован в качестве корня.

1.11. Удаляем его логический том:

[root@lvm ~]# lvremove /dev/VolGroup00/LogVol00

1.12. Создаем новый логический том меньшего размера:

[root@lvm ~]# lvcreate -n VolGroup00/LogVol00 -L 8G /dev/VolGroup00

1.13. Создаем на нем файловую систему и монтируем его:

[root@lvm ~]# mkfs.xfs /dev/VolGroup00/LogVol00

[root@lvm ~]# mount /dev/VolGroup00/LogVol00  /mnt

1.14. Возвращаем обратно содержимое корня:

[root@lvm ~]# xfsdump -J - /dev/vg_root/lv_root | xfsrestore -J - /mnt

root@lvm ~]# ls /mnt

1.15.Заходим в окружение chroot нашего временного корня:

[root@lvm ~]# for i in /proc/ /sys/ /dev/ /run/ /boot/; do mount --bind $i /mnt/$i; done

[root@lvm ~]# chroot /mnt

1.16. Запишем новый загрузчик:

[root@lvm ~]# grub2-mkconfig -o /boot/grub2/grub.cfg

1.17. Обновляем образы загрузки:

[root@lvm ~]# cd /boot ; for i in `ls initramfs-*img`; do dracut -v $i `echo $i|sed "s/initramfs-//g; s/.img//g"` --force; done

1.18. Открываем конфигурационный файл grub:

[root@lvm ~]# vi /boot/grub2/grub.cfg

задействуют том lvm

Перезагружаем и проверяем.

Удаляем том, группу и снимаем lvm-метку с диска, который нами использовался как временный (sdb):

[root@lvm ~]# lvremove /dev/vg_root/lv_root

[root@lvm ~]# vgremove vg_root

[root@lvm ~]# pvremove /dev/sdb


2. Выделить том под /var в зеркало.

[root@lvm ~]# lsblk

На свободных дисках создаем зеркало:

[root@lvm ~]# pvcreate /dev/sdd /dev/sde

[root@lvm ~]# vgcreate vg_var /dev/sdd /dev/sde

[root@lvm ~]# lvcreate -l +100%FREE -m1 -n lv_var vg_var

2.1. Создаем на нем ФС и перемещаем туда /var:

[root@lvm ~]# mkfs.ext4 /dev/vg_var/lv_var

[root@lvm ~]# mount /dev/vg_var/lv_var /mnt

[root@lvm ~]# rsync -avHPSAX /var/ /mnt/

На всякий случай сохраняем содержимое старого var (или же можно его просто удалить):

[root@lvm ~]# mkdir /tmp/oldvar && mv /var/* /tmp/oldvar

монтируем новый var в каталог /var:

[root@lvm ~]# umount /mnt
[root@lvm ~]# mount /dev/vg_var/lv_var /var

Правим fstab для автоматического монтирования /var:
[root@lvm ~]# echo "`blkid | grep var: | awk '{print $2}'` /var ext4 defaults 0 0" >> /etc/fstab

Перезагружаем систему

[root@lvm ~]# lsblk


3. Выделить том под /home

[root@lvm ~]# lvcreate -n LogVol02 -L 2G /dev/VolGroup00

[root@lvm ~]# mkfs.xfs /dev/VolGroup00/LogVol02

[root@lvm ~]# mount /dev/VolGroup00/LogVol02 /mnt/

[root@lvm ~]# cp -aR /home/* /mnt/

[root@lvm ~]# rm -rf /home/*

[root@lvm ~]# umount /mnt

[root@lvm ~]# mount /dev/VolGroup00/LogVol02 /home/

[root@lvm ~]# echo "`blkid | grep LogVol02 | awk '{print $2}'` /home xfs defaults 0 0" >> /etc/fstab

4. /home - сделать том для снапшотов

Сгенерируем файлы в /home/:

[root@lvm ~]# touch /home/file{1..20}

Снять снапшот:

[root@lvm ~]# lvcreate -L 200MB -s -n home_snap /dev/VolGroup00/LogVol02

Удалить снапшот:

[root@lvm ~]# lvremove /dev/VolGroup00/home_snap

Удалить часть файлов:

[root@lvm ~]# rm -f /home/file{11..20}

Процесс восстановления со снапшота:

[root@lvm ~]# umount -l /home

[root@lvm ~]# lvconvert --merge /dev/VolGroup00/home_snap

[root@lvm ~]# mount /home
