**Домашнее задание №4**

**Тема** ***"Дисковая подсистема"***

**Задача**
Подготовить научиться использовать утилиту для управления программными RAID-массивами в Linux:

- уменьшить том под / до 8G;
- выделить том под /home;
- выделить том под /var - сделать в mirror;
- прописать монтирование в fstab;
- сгенерить файлы в /home/;
- снять снапшот;
- удалить часть файлов;
- восстановится со снапшота.

---
**Результат выполнения задания**
1. Создан `Vagrantfile` и поднята ВМ:
```
vagrant status
Current machine states:

lvm                       running (virtualbox)

vagrant ssh 
Last login: Sun Apr 21 19:34:14 2024 from 10.0.2.2
[vagrant@lvm ~]$ lsblk 
NAME                    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                       8:0    0   40G  0 disk 
├─sda1                    8:1    0    1M  0 part 
├─sda2                    8:2    0    1G  0 part /boot
└─sda3                    8:3    0   39G  0 part 
  ├─VolGroup00-LogVol00 253:0    0 37.5G  0 lvm  /
  └─VolGroup00-LogVol01 253:1    0  1.5G  0 lvm  [SWAP]
sdb                       8:16   0   10G  0 disk 
sdc                       8:32   0    2G  0 disk 
sdd                       8:48   0    1G  0 disk 
sde                       8:64   0    1G  0 disk
```

2. Созданы роли ansible для автоматизации задачи:
    - installing - установка необходимых пакетов
    - create_tmp_lvm - создание временного логического раздела
    - resize_lvm - изменение размера основного раздела
    - var_mirror - создание зеркала var
    - home_part - создание тома под home

3. Выполены роли:
    - installing
    - create_tmp_lvm
```
ansible-playbook playbooks/lvm.yaml --tags "installing, create_tmp_lvm"

PLAY [lvm] *********************************************************************

TASK [Gathering Facts] *********************************************************
ok: [lvm]

TASK [installing : installing xfsdump] *****************************************
ok: [lvm] => (item=xfsdump)
ok: [lvm] => (item=vim)

TASK [create_tmp_lvm : check vg] ***********************************************
ok: [lvm]

TASK [create_tmp_lvm : create volume group] ************************************
changed: [lvm]

TASK [create_tmp_lvm : LVM | check lvc] ****************************************
ok: [lvm]

TASK [create_tmp_lvm : LVM | create logical volume] ****************************
changed: [lvm]

TASK [create_tmp_lvm : LVM | format the xfs filesystem] ************************
changed: [lvm]

TASK [create_tmp_lvm : mounting logical volume on /mnt] ************************
changed: [lvm]

TASK [create_tmp_lvm : /mnt check] *********************************************
ok: [lvm]

TASK [create_tmp_lvm : copy / on /mnt] *****************************************
changed: [lvm]

TASK [create_tmp_lvm : LVM | Mounting partitions] ******************************
changed: [lvm]

PLAY RECAP *********************************************************************
lvm                        : ok=11   changed=6    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0  
```

В ходе выполнения роли:
- установлены библитеки и **создан временный логический раздел** для переноса системы
 ```
[root@lvm ~]# lsblk 
NAME                    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                       8:0    0   40G  0 disk 
├─sda1                    8:1    0    1M  0 part 
├─sda2                    8:2    0    1G  0 part /mnt/boot
└─sda3                    8:3    0   39G  0 part 
  ├─VolGroup00-LogVol00 253:0    0 37.5G  0 lvm  /
  └─VolGroup00-LogVol01 253:1    0  1.5G  0 lvm  [SWAP]
sdb                       8:16   0   10G  0 disk 
└─vg_root-lv_root       253:2    0   10G  0 lvm  /mnt
sdc                       8:32   0    2G  0 disk 
sdd                       8:48   0    1G  0 disk 
sde                       8:64   0    1G  0 disk 
 ```

4. Под chroot сконфигурирован grub и initrd:
```
[root@lvm ~]# chroot /mnt/
[root@lvm /]# grub2-mkconfig -o /boot/grub2/grub.cfg
Generating grub configuration file ...

cd /boot ; for i in `ls initramfs-*img`; \
> do dracut -v $i `echo $i|sed "s/initramfs-//g; \
> s/.img//g"` --force; done
Executing: /sbin/dracut -v initramfs-3.10.0-862.2.3.el7.x86_64.img 3.10.0-862.2.3.el7.x86_64 --force

*** Creating image file done ***
*** Creating initramfs image file '/boot/initramfs-3.10.0-862.2.3.el7.x86_64.img' done ***
```

5. Для загрузки во временный логический раздел, в grub config выбран только что созданный логический раздел
```
[root@lvm ~]# umount /mnt/
[root@lvm ~]# lsblk 
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
```

6. Выполнена роль resize:
```
ansible-playbook playbooks/lvm.yaml --tags "resize"         ─╯

PLAY [lvm] *********************************************************************

TASK [Gathering Facts] *********************************************************
ok: [lvm]

TASK [resize_lvm : Remove logical volume] **************************************
changed: [lvm]

TASK [resize_lvm : check vg] ***************************************************
ok: [lvm]

TASK [resize_lvm : create volume group] ****************************************
skipping: [lvm]

TASK [resize_lvm : check lv] ***************************************************
ok: [lvm]

TASK [resize_lvm : create logical volume] **************************************
changed: [lvm]

TASK [resize_lvm : format the xfs filesystem] **********************************
changed: [lvm]

TASK [resize_lvm : mounting logical volume on /mnt] ****************************
changed: [lvm]

TASK [resize_lvm : /mnt check] *************************************************
ok: [lvm]

TASK [resize_lvm : copy / on /mnt] *********************************************
changed: [lvm]

TASK [resize_lvm : mounting partitions] ****************************************
changed: [lvm]

PLAY RECAP *********************************************************************
lvm                        : ok=10   changed=6    unreachable=0    failed=0    skipped=1    rescued=0    ignored=0
```

6. Аналогично сконфигурирован grub и initrd. После конфигурирования осуществлена перезагрузка и, в итоге, **раздел / уменьшин до 8 GB**
```
[root@lvm ~]# umount /mnt/
[root@lvm ~]# lsblk 
NAME                    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                       8:0    0   40G  0 disk 
├─sda1                    8:1    0    1M  0 part 
├─sda2                    8:2    0    1G  0 part /boot
└─sda3                    8:3    0   39G  0 part 
  ├─VolGroup00-LogVol00 253:0    0    8G  0 lvm  /
  └─VolGroup00-LogVol01 253:1    0  1.5G  0 lvm  [SWAP]
sdb                       8:16   0   10G  0 disk 
└─vg_root-lv_root       253:2    0   10G  0 lvm  
sdc                       8:32   0    2G  0 disk 
sdd                       8:48   0    1G  0 disk 
sde                       8:64   0    1G  0 disk 
```

7. Выполнена роль var_mirrot, для **выделения тома под var в зеркало:**
```
ansible-playbook playbooks/lvm.yaml --tags "var_mirror"

PLAY [lvm] *********************************************************************

TASK [Gathering Facts] *********************************************************
ok: [lvm]

TASK [var_mirror : Create volume group] ****************************************
changed: [lvm] => (item=/dev/sdc)
changed: [lvm] => (item=/dev/sdd)

TASK [var_mirror : Create logical volume] **************************************
changed: [lvm]

TASK [var_mirror : Format the xfs filesystem] **********************************
changed: [lvm]

TASK [var_mirror : mounting logical volume on /mnt] ****************************
changed: [lvm]

TASK [var_mirror : copy /var/ on /mnt] *****************************************
changed: [lvm]

TASK [var_mirror : umount a /mnt] **********************************************
changed: [lvm]

TASK [var_mirror : mounting logical volume on /mnt] ****************************
changed: [lvm]

TASK [var_mirror : remove logical volume] **************************************
changed: [lvm]

PLAY RECAP *********************************************************************
lvm                        : ok=9    changed=8    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```
После выполнения роли сконфигурирован fstab, для автоматического монтирования раздела:
```
[root@lvm ~]# echo "`blkid | grep var: | awk '{print $2}'` \
>  /var ext4 defaults 0 0" >> /etc/fstab
[root@lvm ~]# lsblk 
NAME                    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                       8:0    0   40G  0 disk 
├─sda1                    8:1    0    1M  0 part 
├─sda2                    8:2    0    1G  0 part /boot
└─sda3                    8:3    0   39G  0 part 
  ├─VolGroup00-LogVol00 253:0    0    8G  0 lvm  /
  └─VolGroup00-LogVol01 253:1    0  1.5G  0 lvm  [SWAP]
sdb                       8:16   0   10G  0 disk 
sdc                       8:32   0    2G  0 disk 
sdd                       8:48   0    1G  0 disk 
└─vg_var-lv_var         253:3    0  952M  0 lvm  /var
sde                       8:64   0    1G  0 disk 
```

8. Выполнена роль для выделения тома под /home/:
```
ansible-playbook playbooks/lvm.yaml --tags "create_home_part"

PLAY [lvm] *********************************************************************

TASK [Gathering Facts] *********************************************************
ok: [lvm]

TASK [home_part : Create logical volume] ***************************************
changed: [lvm]

TASK [home_part : Format the xfs filesystem] ***********************************
changed: [lvm]

TASK [home_part : mounting logical volume on /mnt] *****************************
changed: [lvm]

TASK [home_part : copy /var/ on /mnt] ******************************************
changed: [lvm]

TASK [home_part : umount a /mnt] ***********************************************
changed: [lvm]

TASK [home_part : mounting logical volume on /mnt] *****************************
changed: [lvm]

PLAY RECAP *********************************************************************
lvm                        : ok=7    changed=6    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   


╭─    ~/S/LvmCreate    main !9 ?7 
╰─ ansible-playbook playbooks/lvm.yaml --tags "create_home_part"

PLAY [lvm] *********************************************************************

TASK [Gathering Facts] *********************************************************
ok: [lvm]

TASK [home_part : Create logical volume] ***************************************
ok: [lvm]

TASK [home_part : Format the xfs filesystem] ***********************************
ok: [lvm]

TASK [home_part : mounting logical volume on /mnt] *****************************
changed: [lvm]

TASK [home_part : copy /home/ on /mnt] *****************************************
changed: [lvm]

TASK [home_part : umount a /mnt] ***********************************************
changed: [lvm]

TASK [home_part : mounting logical volume on /mnt] *****************************
changed: [lvm]

PLAY RECAP *********************************************************************
lvm                        : ok=7    changed=4    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0 
```
Также, для автоматического монтирования сконфигурирован fstab. **В  результате выделен раздел под /home**
```
[root@lvm ~]# lsblk 
NAME                       MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                          8:0    0   40G  0 disk 
├─sda1                       8:1    0    1M  0 part 
├─sda2                       8:2    0    1G  0 part /boot
└─sda3                       8:3    0   39G  0 part 
  ├─VolGroup00-LogVol00    253:0    0    8G  0 lvm  /
  ├─VolGroup00-LogVol01    253:1    0  1.5G  0 lvm  [SWAP]
  └─VolGroup00-LogVol_Home 253:2    0    2G  0 lvm  /home
sdb                          8:16   0   10G  0 disk 
sdc                          8:32   0    2G  0 disk 
sdd                          8:48   0    1G  0 disk 
└─vg_var-lv_var            253:3    0  952M  0 lvm  /var
sde                          8:64   0    1G  0 disk 
```
