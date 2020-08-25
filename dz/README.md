### Стенд для занятия по LVM
Попробуйте настроить файловую систему. Развернем vagrant с OS Centos 7, после чего выключим VM. Для решение задач, нужно сделать по умолчанию ssh root. Добавляем в наш Vagrantfile следующее:
```
[root@lvm ~]$ vim ./Vagrantfile
```
```
config.ssh.username = 'root'
config.ssh.password = 'vagrant'
config.ssh.insert_key = 'true'
```
#### Уменьшить том под / до 8G
Проверим наличие дисков:
```
[root@lvm ~]$ lsblk
```
Для решение задачи установим утилиту xfsdump
```
[root@lvm ~]# yum update
[root@lvm ~]# yum install xfsdump -y
```
Подготовим временный раздел для корневого тома:
```
[root@lvm ~]# pvcreate /dev/sdb
[root@lvm ~]# vgcreate vg_root /dev/sdb
[root@lvm ~]# lvcreate -n lv_root -l +100%FREE /dev/vg_root
```
Создадим файловую систему XFS и смонтируем ее в каталог /mnt
```
[root@lvm ~]# mkfs.xfs /dev/vg_root/lv_root
[root@lvm ~]# mount /dev/vg_root/lv_root /mnt
[vagrant@lvm ~]$ lsblk
```
Сдампим содержимое текущего корневого раздела в наш временный:
```
[root@lvm ~]# xfsdump -J - /dev/VolGroup00/LogVol00 | xfsrestore -J - /mnt
```
Проверим содержимое каталога /mnt
```
[root@lvm ~]# ls /mnt
```
Заходим в окружение chroot нашего временного корня:
```
[root@lvm ~]# for i in /proc/ /sys/ /dev/ /run/ /boot/; do mount --bind $i /mnt/$i; done
[root@lvm ~]# chroot /mnt/
```
Запишем новый загрузчик:
```
[root@lvm ~]# grub2-mkconfig -o /boot/grub2/grub.cfg
```
Обновляем образы загрузки:
```
[root@lvm ~]# cd /boot ; for i in `ls initramfs-*img`; do dracut -v $i `echo $i|sed "s/initramfs-//g; s/.img//g"` --force; done
```
Открываем конфигурационный файл grub:
```
[root@lvm ~]# vi /boot/grub2/grub.cfg
```
Меняем значение lv=VolGroup00/LogVol00 на lv=vg_root/lv_root. Выходим из окружения chroot и перезагружаем компьютер reboot. Проверяем что загрузился временный раздел:
```
[root@lvm ~]# lsblk
```
И так, у нас есть раздел, который нужно уменьшить и который теперь не примонтирован в качестве корня. Удаляем его логический том:
```
[root@lvm ~]# lvremove /dev/VolGroup00/LogVol00
```
Создаем новый логический том меньшего размера:
```
[root@lvm ~]# lvcreate -n VolGroup00/LogVol00 -L 8G /dev/VolGroup00
```
Создаем на нем файловую систему и монтируем его:
```
[root@lvm ~]# mkfs.xfs /dev/VolGroup00/LogVol00
[root@lvm ~]# mount /dev/VolGroup00/LogVol00  /mnt
```
Возвращаем обратно содержимое корня:
```
[root@lvm ~]# xfsdump -J - /dev/vg_root/lv_root | xfsrestore -J - /mnt
[root@lvm ~]# ls /mnt
```
Заходим в окружение chroot нашего временного корня:
```
[root@lvm ~]# for i in /proc/ /sys/ /dev/ /run/ /boot/; do mount --bind $i /mnt/$i; done
[root@lvm ~]# chroot /mnt
```
Запишем новый загрузчик:
```
[root@lvm ~]# grub2-mkconfig -o /boot/grub2/grub.cfg
```
Обновляем образы загрузки:
```
[root@lvm ~]# cd /boot ; for i in `ls initramfs-*img`; do dracut -v $i `echo $i|sed "s/initramfs-//g; s/.img//g"` --force; done
```
Открываем конфигурационный файл grub:
```
[root@lvm ~]# vi /boot/grub2/grub.cfg
```
Задействуют том lvm. Перезагружаем и проверяем.
```
[root@lvm ~]# lsblk
```
Удаляем том, группу и снимаем lvm-метку с диска, который нами использовался как временный (sdb):
```
[root@lvm ~]# lvremove /dev/vg_root/lv_root
[root@lvm ~]# vgremove vg_root
[root@lvm ~]# pvremove /dev/sdb
```

#### Выделить том под /var в зеркало.
```
[root@lvm ~]# lsblk
```
На свободных дисках создаем зеркало:
```
[root@lvm ~]# pvcreate /dev/sdd /dev/sde
[root@lvm ~]# vgcreate vg_var /dev/sdd /dev/sde
[root@lvm ~]# lvcreate -l +100%FREE -m1 -n lv_var vg_var
```
Создаем на нем ФС и перемещаем туда /var:
```
[root@lvm ~]# mkfs.ext4 /dev/vg_var/lv_var
[root@lvm ~]# mount /dev/vg_var/lv_var /mnt
[root@lvm ~]# rsync -avHPSAX /var/ /mnt/
```
На всякий случай сохраняем содержимое старого var (или же можно его просто удалить):
```
[root@lvm ~]# mkdir /tmp/oldvar && mv /var/* /tmp/oldvar
```
Монтируем новый var в каталог /var:
```
[root@lvm ~]# umount /mnt
[root@lvm ~]# mount /dev/vg_var/lv_var /var
```
Правим fstab для автоматического монтирования /var:
```
[root@lvm ~]# echo "`blkid | grep var: | awk '{print $2}'` /var ext4 defaults 0 0" >> /etc/fstab
```
Перезагружаем систему
```
[root@lvm ~]# lsblk
```
#### Выделить том под /home
```
[root@lvm ~]# lvcreate -n LogVol02 -L 2G /dev/VolGroup00
[root@lvm ~]# mkfs.xfs /dev/VolGroup00/LogVol02
[root@lvm ~]# mount /dev/VolGroup00/LogVol02 /mnt/
[root@lvm ~]# cp -aR /home/* /mnt/
[root@lvm ~]# rm -rf /home/*
[root@lvm ~]# umount /mnt
[root@lvm ~]# mount /dev/VolGroup00/LogVol02 /home/
[root@lvm ~]# echo "`blkid | grep LogVol02 | awk '{print $2}'` /home xfs defaults 0 0" >> /etc/fstab
```
#### /home - сделать том для снапшотов

Сгенерируем файлы в /home/:
```
[root@lvm ~]# touch /home/file{1..20}
```
Снять снапшот:
```
[root@lvm ~]# lvcreate -L 200MB -s -n home_snap /dev/VolGroup00/LogVol02
```
Удалить снапшот: (в том случии, когда нужно)
```
[root@lvm ~]# lvremove /dev/VolGroup00/home_snap
```
Удалить часть файлов:
```
[root@lvm ~]# rm -f /home/file{11..20}
```
Процесс восстановления со снапшота:
```
[root@lvm ~]# umount -l /home
[root@lvm ~]# lvconvert --merge /dev/VolGroup00/home_snap
[root@lvm ~]# mount /home
```
Детально посмотреть выполнение ДЗ. Утилитой asciinema оспроизвести файл dz_lvm.cast

Методички:
```
- [fcfg-eth1](provisioning/templates/ifcfg-eth1.j2)
1. [Теория_LVM](blob/master/Теория_LVM_FS-5373-72b708.pdf)
```
