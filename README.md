# Домашнее задание: работа с mdadm.
## 1. Добавить в Vagrantfile дополнительные диски.
В конфигурационный файл были добавлены 2 диска, а так же размер каждого инициализируемого диска был выставлен в размере 100 MB:
```
                :sata1 => {
                        :dfile => './sata1.vdi',
                        :size => 100, 
                        :port => 1
                },
...
                :sata6 => {
                        :dfile => './sata6.vdi',
                        :size => 100, 
                        :port => 6
                }
```
После запуска, проверка успешности создания дисков:
```
$ sudo lshw -short | grep disk
/0/100/1.1/0.0.0    /dev/sda   disk        42GB VBOX HARDDISK
/0/100/d/0          /dev/sdb   disk        104MB VBOX HARDDISK
/0/100/d/1          /dev/sdc   disk        104MB VBOX HARDDISK
/0/100/d/2          /dev/sdd   disk        104MB VBOX HARDDISK
/0/100/d/3          /dev/sde   disk        104MB VBOX HARDDISK
/0/100/d/4          /dev/sdf   disk        104MB VBOX HARDDISK
/0/100/d/5          /dev/sdg   disk        104MB VBOX HARDDISK
```

## 2. Собрать R0/R5/R10 на выбор.
Тип RAID был выбран 10.

Проверяем, использовались ли диски в других рейдах и, если да, обнуляем суперблоки:
```
$ sudo mdadm --zero-superblock --force /dev/sd{b,c,d,e,f,g}
mdadm: Unrecognised md component device - /dev/sdb
mdadm: Unrecognised md component device - /dev/sdc
mdadm: Unrecognised md component device - /dev/sdd
mdadm: Unrecognised md component device - /dev/sde
mdadm: Unrecognised md component device - /dev/sdf
mdadm: Unrecognised md component device - /dev/sdg
```

Создаём RAAID 10 из 4-х дисков, добавляем 1 диск в hot-spare и смотрим результат:
```
$ sudo mdadm --create --verbose /dev/md0 -l 10 -n 4 /dev/sd{b,c,d,e}
mdadm: layout defaults to n2
mdadm: layout defaults to n2
mdadm: chunk size defaults to 512K
mdadm: size set to 100352K
mdadm: Defaulting to version 1.2 metadata
mdadm: array /dev/md0 started.
```
```
$ sudo mdadm /dev/md0 --add /dev/sdf
mdadm: added /dev/sdf
```
```
$ sudo mdadm -D /dev/md0 
/dev/md0:
           Version : 1.2
     Creation Time : Tue Feb  7 14:35:33 2023
        Raid Level : raid10
        Array Size : 200704 (196.00 MiB 205.52 MB)
     Used Dev Size : 100352 (98.00 MiB 102.76 MB)
      Raid Devices : 4
     Total Devices : 5
       Persistence : Superblock is persistent

       Update Time : Tue Feb  7 14:37:31 2023
             State : clean 
    Active Devices : 4
   Working Devices : 5
    Failed Devices : 0
     Spare Devices : 1

            Layout : near=2
        Chunk Size : 512K

Consistency Policy : resync

              Name : otuslinux:0  (local to host otuslinux)
              UUID : 107c4cd9:f1627f99:47e5c262:e0e182ef
            Events : 18

    Number   Major   Minor   RaidDevice State
       0       8       16        0      active sync set-A   /dev/sdb
       1       8       32        1      active sync set-B   /dev/sdc
       2       8       48        2      active sync set-A   /dev/sdd
       3       8       64        3      active sync set-B   /dev/sde

       4       8       80        -      spare   /dev/sdf
```

## 3. Создание конфигурационного файла mdadm.conf
Создаём необходимые каталог и конфигурационный файл, заполняем его данными о массиве, проверяем:
```
$ sudo mkdir /etc/mdadm/
$ sudo touch /etc/mdadm/mdadm.conf
```
```
# echo "DEVICE partitions" > /etc/mdadm/mdadm.conf
# mdadm --detail --scan --verbose | awk '/ARRAY/ {print}' >> /etc/mdadm/mdadm.conf
```
```
# cat /etc/mdadm/mdadm.conf
DEVICE partitions
ARRAY /dev/md0 level=raid10 num-devices=4 metadata=1.2 spares=1 name=otuslinux:0 UUID=107c4cd9:f1627f99:47e5c262:e0e182ef
```
## 4. Сломать/починить RAID
Помечаем, как сбойный диск sdb и проверяем работу hot-spare:
```
$ sudo mdadm /dev/md0 --fail /dev/sdb
mdadm: set /dev/sdb faulty in /dev/md0
```
```
$ sudo mdadm -D /dev/md0
/dev/md0:
           Version : 1.2
     Creation Time : Tue Feb  7 14:35:33 2023
        Raid Level : raid10
        Array Size : 200704 (196.00 MiB 205.52 MB)
     Used Dev Size : 100352 (98.00 MiB 102.76 MB)
      Raid Devices : 4
     Total Devices : 5
       Persistence : Superblock is persistent

       Update Time : Tue Feb  7 14:52:10 2023
             State : clean 
    Active Devices : 4
   Working Devices : 4
    Failed Devices : 1
     Spare Devices : 0

            Layout : near=2
        Chunk Size : 512K

Consistency Policy : resync

              Name : otuslinux:0  (local to host otuslinux)
              UUID : 107c4cd9:f1627f99:47e5c262:e0e182ef
            Events : 37

    Number   Major   Minor   RaidDevice State
       4       8       80        0      active sync set-A   /dev/sdf
       1       8       32        1      active sync set-B   /dev/sdc
       2       8       48        2      active sync set-A   /dev/sdd
       3       8       64        3      active sync set-B   /dev/sde

       0       8       16        -      faulty   /dev/sdb
```
Помечаем, как сбойный диск sdb и проверяем статус массива:
```
$ sudo mdadm /dev/md0 --fail /dev/sde
mdadm: set /dev/sde faulty in /dev/md0
```
```
$ cat /proc/mdstat 
Personalities : [raid10] 
md0 : active raid10 sdc[1] sde[3](F) sdd[2] sdf[4] sdb[0](F)
      200704 blocks super 1.2 512K chunks 2 near-copies [4/3] [UUU_]
      
unused devices: <none>
```
Удаляем сбойные диски из массива:
```
$ sudo mdadm /dev/md0 --remove /dev/sdb
mdadm: hot removed /dev/sdb from /dev/md0
$ sudo mdadm /dev/md0 --remove /dev/sde
mdadm: hot removed /dev/sde from /dev/md0
```
Добавляем резервный диск sdg в массив, проверяем:
```
$ sudo mdadm /dev/md0 --add /dev/sdg
mdadm: added /dev/sdg
```
```
$ cat /proc/mdstat 
Personalities : [raid10] 
md0 : active raid10 sdg[5] sdc[1] sdd[2] sdf[4]
      200704 blocks super 1.2 512K chunks 2 near-copies [4/4] [UUUU]
      
unused devices: <none>
```

## 5. Создать GPT раздел, пять партиций и смонтировать их на диск.
Создание GPT раздела:
```
$ sudo parted -s /dev/md0 mklabel gpt
```
Создание пяти партиций:
```
$ sudo parted -s /dev/md0 mkpart primary ext4 0% 20%
$ sudo parted -s /dev/md0 mkpart primary ext4 20% 40%
$ sudo parted -s /dev/md0 mkpart primary ext4 40% 60%
$ sudo parted -s /dev/md0 mkpart primary ext4 60% 80%
$ sudo parted -s /dev/md0 mkpart primary ext4 80% 100%
```
Создание на них ФС:
```
$ for i in $(seq 1 5); do sudo mkfs.ext4 /dev/md0p$i; done
mke2fs 1.42.9 (28-Dec-2013)
/dev/md0p1 alignment is offset by 506880 bytes.
This may result in very poor performance, (re)-partitioning suggested.
Filesystem label=
OS type: Linux
Block size=1024 (log=0)
Fragment size=1024 (log=0)
Stride=512 blocks, Stripe width=1024 blocks
10040 inodes, 40124 blocks
2006 blocks (5.00%) reserved for the super user
First data block=1
Maximum filesystem blocks=33685504
5 block groups
8192 blocks per group, 8192 fragments per group
2008 inodes per group
Superblock backups stored on blocks: 
	8193, 24577

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (4096 blocks): done
Writing superblocks and filesystem accounting information: done

mke2fs 1.42.9 (28-Dec-2013)
Filesystem label=
OS type: Linux
Block size=1024 (log=0)
Fragment size=1024 (log=0)
Stride=512 blocks, Stripe width=1024 blocks
9760 inodes, 38912 blocks
1945 blocks (5.00%) reserved for the super user
First data block=1
Maximum filesystem blocks=33685504
5 block groups
8192 blocks per group, 8192 fragments per group
1952 inodes per group
Superblock backups stored on blocks: 
	8193, 24577

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (4096 blocks): done
Writing superblocks and filesystem accounting information: done

mke2fs 1.42.9 (28-Dec-2013)
Filesystem label=
OS type: Linux
Block size=1024 (log=0)
Fragment size=1024 (log=0)
Stride=512 blocks, Stripe width=1024 blocks
10240 inodes, 40960 blocks
2048 blocks (5.00%) reserved for the super user
First data block=1
Maximum filesystem blocks=33685504
5 block groups
8192 blocks per group, 8192 fragments per group
2048 inodes per group
Superblock backups stored on blocks: 
	8193, 24577

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (4096 blocks): done
Writing superblocks and filesystem accounting information: done

mke2fs 1.42.9 (28-Dec-2013)
Filesystem label=
OS type: Linux
Block size=1024 (log=0)
Fragment size=1024 (log=0)
Stride=512 blocks, Stripe width=1024 blocks
10000 inodes, 39936 blocks
1996 blocks (5.00%) reserved for the super user
First data block=1
Maximum filesystem blocks=33685504
5 block groups
8192 blocks per group, 8192 fragments per group
2000 inodes per group
Superblock backups stored on blocks: 
	8193, 24577

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (4096 blocks): done
Writing superblocks and filesystem accounting information: done

mke2fs 1.42.9 (28-Dec-2013)
Filesystem label=
OS type: Linux
Block size=1024 (log=0)
Fragment size=1024 (log=0)
Stride=512 blocks, Stripe width=1024 blocks
10000 inodes, 39916 blocks
1995 blocks (5.00%) reserved for the super user
First data block=1
Maximum filesystem blocks=33685504
5 block groups
8192 blocks per group, 8192 fragments per group
2000 inodes per group
Superblock backups stored on blocks: 
	8193, 24577

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (4096 blocks): done
Writing superblocks and filesystem accounting information: done
```
Монтирование новых разделов, проверка:
```
$ for i in $(seq 1 5); do sudo mkdir -p /raid_part/fs{1,2,3,4,5} && sudo mount /dev/md0p$i /raid_part/fs$i; done
```
```
$ lsblk 
NAME      MAJ:MIN RM  SIZE RO TYPE   MOUNTPOINT
sda         8:0    0   40G  0 disk   
`-sda1      8:1    0   40G  0 part   /
sdb         8:16   0  100M  0 disk   
sdc         8:32   0  100M  0 disk   
`-md0       9:0    0  196M  0 raid10 
  |-md0p1 259:0    0 39.2M  0 md     /raid_part/fs1
  |-md0p2 259:1    0   38M  0 md     /raid_part/fs2
  |-md0p3 259:2    0   40M  0 md     /raid_part/fs3
  |-md0p4 259:3    0   39M  0 md     /raid_part/fs4
  `-md0p5 259:4    0   39M  0 md     /raid_part/fs5
sdd         8:48   0  100M  0 disk   
`-md0       9:0    0  196M  0 raid10 
  |-md0p1 259:0    0 39.2M  0 md     /raid_part/fs1
  |-md0p2 259:1    0   38M  0 md     /raid_part/fs2
  |-md0p3 259:2    0   40M  0 md     /raid_part/fs3
  |-md0p4 259:3    0   39M  0 md     /raid_part/fs4
  `-md0p5 259:4    0   39M  0 md     /raid_part/fs5
sde         8:64   0  100M  0 disk   
sdf         8:80   0  100M  0 disk   
`-md0       9:0    0  196M  0 raid10 
  |-md0p1 259:0    0 39.2M  0 md     /raid_part/fs1
  |-md0p2 259:1    0   38M  0 md     /raid_part/fs2
  |-md0p3 259:2    0   40M  0 md     /raid_part/fs3
  |-md0p4 259:3    0   39M  0 md     /raid_part/fs4
  `-md0p5 259:4    0   39M  0 md     /raid_part/fs5
sdg         8:96   0  100M  0 disk   
`-md0       9:0    0  196M  0 raid10 
  |-md0p1 259:0    0 39.2M  0 md     /raid_part/fs1
  |-md0p2 259:1    0   38M  0 md     /raid_part/fs2
  |-md0p3 259:2    0   40M  0 md     /raid_part/fs3
  |-md0p4 259:3    0   39M  0 md     /raid_part/fs4
  `-md0p5 259:4    0   39M  0 md     /raid_part/fs5

```

Добавляем монтирование разделов при загрузке системы:
```
# for i in $(seq 1 5); do echo `sudo blkid /dev/md0p$i | awk '{print $2}'` /raid_part/fs$i ext4 defaults 0 2 >> /etc/fstab; done
```
```
# cat /etc/fstab 

#
# /etc/fstab
# Created by anaconda on Thu Apr 30 22:04:55 2020
#
# Accessible filesystems, by reference, are maintained under '/dev/disk'
# See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info
#
UUID=1c419d6c-5064-4a2b-953c-05b2c67edb15 /                       xfs     defaults        0 0
/swapfile none swap defaults 0 0
#VAGRANT-BEGIN
# The contents below are automatically generated by Vagrant. Do not modify.
#VAGRANT-END

UUID="f4f9c619-3b04-44cd-810c-ea0b32a16eb5" /raid_part/fs1 ext4 defaults 0 2
UUID="6abb7a91-2264-47b7-b0d4-12d5d3ff2e5e" /raid_part/fs2 ext4 defaults 0 2
UUID="9c061ac2-dca4-4fcf-aa46-2049cfb87591" /raid_part/fs3 ext4 defaults 0 2
UUID="d246aa0c-3a18-4f06-b3d6-768f27fd72b1" /raid_part/fs4 ext4 defaults 0 2
UUID="937053a5-f81a-457b-86c2-5b1a55f77dbd" /raid_part/fs5 ext4 defaults 0 2
```
