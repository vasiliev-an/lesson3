# **Файловые системы и LVM**

### **Редактирование vagrant файла и создание виртуалки**

Редактируем пути до дисков:

```
:dfile => home + '/VirtualBox VMs/disks/sataDiskNumber.vdi',
```

Создаём виртуальную машину и коннектимся к ней:

```
vagrant up
vagrant ssh
```

### **Уменьшить том под / до 8G**

Ставим пакет xfsdump:

```
yum install xfsdump
```

Подготовим временный том для / раздела:

```
pvcreate /dev/sdb
vgcreate vg_root /dev/sdb
lvcreate -n lv_root -l +100%FREE /dev/vg_root
```

Создадим на нем файловую систему и смонтируем его:

```
mkfs.xfs /dev/vg_root/lv_root
mount /dev/vg_root/lv_root /mnt
```

Скопируем все данные с / раздела в /mnt:

```
xfsdump -J - /dev/VolGroup00/LogVol00 | xfsrestore -J - /mnt
```
Переконфигурируем grub для того, чтобы при старте перейти в новый /
Сымитируем текущий root -> сделаем в него chroot и обновим grub:

```
for i in /proc/ /sys/ /dev/ /run/ /boot/; do mount --bind $i /mnt/$i; done
chroot /mnt/
grub2-mkconfig -o /boot/grub2/grub.cfg
```

Обновим образ initrd:

```
cd /boot ; for i in `ls initramfs-*img`; do dracut -v $i `echo $i|sed "s/initramfs-//g; s/.img//g"` --force; done
```

В файле /boot/grub2/grub.cfg заменяем rd.lvm.lv=VolGroup00/LogVol00 на rd.lvm.lv=vg_root/lv_root

```
reboot
```
Изменяем размер старой VG и возвращаем на него рут. Для этого удаляем старый LV размером в 40G и создаем новый на 8G:

```
lvremove /dev/VolGroup00/LogVol00
lvcreate -n VolGroup00/LogVol00 -L 8G /dev/VolGroup00
mkfs.xfs /dev/VolGroup00/LogVol00
mount /dev/VolGroup00/LogVol00 /mnt
xfsdump -J - /dev/vg_root/lv_root | xfsrestore -J - /mnt
```
Как в первый раз переконфигурируем grub, за исключением исправления в файле /etc/grub2/grub.cfg

```
for i in /proc/ /sys/ /dev/ /run/ /boot/; do mount --bind $i /mnt/$i; done
chroot /mnt/
grub2-mkconfig -o /boot/grub2/grub.cfg
cd /boot ; for i in `ls initramfs-*img`; do dracut -v $i `echo $i|sed "s/initramfs-//g; s/.img//g"` --force; done
```


### **Выделить том под /var -  сделать в mirror**

Переносим /var. На свободных дисках создаем зеркало:


```
pvcreate /dev/sdc /dev/sdd
vgcreate vg_var /dev/sdc /dev/sdd
lvcreate -L 950M -m1 -n lv_var vg_var
```

Создаем на нем ФС и перемещаем туда /var:

```
mkfs.ext4 /dev/vg_var/lv_var
mount /dev/vg_var/lv_var /mnt
cp -aR /var/* /mnt/
rsync -avHPSAX /var/ /mnt/
```

Cохраняем на всякий случай содержимое старого var:

```
mkdir /tmp/oldvar && mv /var/* /tmp/oldvar
```

Монтируем новый var в каталог /var:

```
umount /mnt
mount /dev/vg_var/lv_var /var
```
Правим fstab для автоматического монтирования /var и перезагружаемся:

```
echo "`blkid | grep var: | awk '{print $2}'` /var ext4 defaults 0 0" >> /etc/fstab
reboot
```
Удаляем временную Volume Group:

```
lvremove /dev/vg_root/lv_root
vgremove /dev/vg_root
pvremove /dev/sdb
```
### **Выделить том под /home**

Выделяем том под /home по тому же принципу что делали для /var:

```
lvcreate -n LogVol_Home -L 2G /dev/VolGroup00
mkfs.xfs /dev/VolGroup00/LogVol_Home
mount /dev/VolGroup00/LogVol_Home /mnt/
cp -aR /home/* /mnt/ 
rm -rf /home/*
umount /mnt
mount /dev/VolGroup00/LogVol_Home /home/
```

Правим fstab для автоматического монтирования /home:

```
echo "`blkid | grep Home | awk '{print $2}'` /home xfs defaults 0 0" >> /etc/fstab
```

### **/home - сделать том для снапшотов**

Генерируем файлы в /home/:

```
touch /home/file{1..20}
```
Снимаем снапшот:

```
lvcreate -L 100MB -s -n home_snap /dev/VolGroup00/LogVol_Home
```

Удаляем часть файлов:

```
rm -f /home/file{11..20}
```

Процесс восстановления со снапшота:


```
umount /home
lvconvert --merge /dev/VolGroup00/home_snap
mount /home
```