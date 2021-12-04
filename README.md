## 2 Могут ли файлы, являющиеся жесткой ссылкой на один объект, иметь разные права доступа и владельца? Почему?

Не могут. Т.к. это только линки на один и тотже inode. Данные по правам доступа, аттрибутам, владельце и т.п. хранятся в inode, жесткая ссылка - это дополнительное имя для файла.

## 5 Используя sfdisk, перенесите данную таблицу разделов на второй диск.

	sfdisk -d /dev/sdb|sfdisk /dev/sdc
  
## 6 Соберите mdadm RAID1 на паре разделов 2 Гб

	root@ubuntu-20:~# mdadm --create /dev/md0 --level=1 --raid-devices=2 /dev/sdb1 /dev/sdc1
	mdadm: Note: this array has metadata at the start and
	    may not be suitable as a boot device.  If you plan to
	    store '/boot' on this device please ensure that
	    your boot-loader understands md/v1.x metadata, or use
	    --metadata=0.90
	Continue creating array? y
	mdadm: Defaulting to version 1.2 metadata
	mdadm: array /dev/md0 started.
	
## 7 Соберите mdadm RAID0 на второй паре маленьких разделов.

	root@ubuntu-20:~# mdadm --create /dev/md1 --chunk=64 --level=0 --raid-devices=2 /dev/sdb2 /dev/sdc2
	mdadm: Defaulting to version 1.2 metadata
	mdadm: array /dev/md1 started.
	
## 8 Создайте 2 независимых PV на получившихся md-устройствах.

	root@ubuntu-20:~# pvcreate /dev/md0 /dev/md1
	  Physical volume "/dev/md0" successfully created.
	  Physical volume "/dev/md1" successfully created.

## 9 Создайте общую volume-group на этих двух PV.

	root@ubuntu-20:~# vgcreate testvg /dev/md0 /dev/md1
	  Volume group "testvg" successfully created

## 10 Создайте LV размером 100 Мб, указав его расположение на PV с RAID0.

	root@ubuntu-20:~# lvcreate -n testr0 -L 100M testvg /dev/md1
	  Logical volume "testr0" created.

## 11 Создайте mkfs.ext4 ФС на получившемся LV.

	root@ubuntu-20:~# mkfs.ext4 /dev/mapper/testvg-testr0
	mke2fs 1.45.5 (07-Jan-2020)
	Discarding device blocks: done
	Creating filesystem with 25600 4k blocks and 25600 inodes
	
	Allocating group tables: done
	Writing inode tables: done
	Creating journal (1024 blocks): done
	Writing superblocks and filesystem accounting information: done
	
## 12 Смонтируйте этот раздел в любую директорию, например, /tmp/new.

	root@ubuntu-20:~# mkdir /tmp/new
	root@ubuntu-20:~# mount /dev/mapper/testvg-testr0 /tmp/new/
	
## 13 Поместите туда тестовый файл, например wget https://mirror.yandex.ru/ubuntu/ls-lR.gz -O /tmp/new/test.gz.

	root@ubuntu-20:~# wget https://mirror.yandex.ru/ubuntu/ls-lR.gz -O /tmp/new/test.gz
	--2021-12-04 19:06:20--  https://mirror.yandex.ru/ubuntu/ls-lR.gz
	Resolving mirror.yandex.ru (mirror.yandex.ru)... 213.180.204.183, 2a02:6b8::183
	Connecting to mirror.yandex.ru (mirror.yandex.ru)|213.180.204.183|:443... connected.
	HTTP request sent, awaiting response... 200 OK
	Length: 22674002 (22M) [application/octet-stream]
	Saving to: ‘/tmp/new/test.gz’
	
	/tmp/new/test.gz              100%[=================================================>]  21.62M  12.5MB/s    in 1.7s
	
	2021-12-04 19:06:22 (12.5 MB/s) - ‘/tmp/new/test.gz’ saved [22674002/22674002]
	
## 14 Прикрепите вывод lsblk.

	root@ubuntu-20:~# lsblk
	NAME                    MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINT
	sda                       8:0    0   64G  0 disk
	├─sda1                    8:1    0  512M  0 part  /boot/efi
	└─sda2                    8:2    0 63.5G  0 part
	  ├─vgubuntu--20-root   253:0    0 62.6G  0 lvm   /
	  └─vgubuntu--20-swap_1 253:1    0  980M  0 lvm   [SWAP]
	sdb                       8:16   0    3G  0 disk
	├─sdb1                    8:17   0    2G  0 part
	│ └─md0                   9:0    0    2G  0 raid1
	└─sdb2                    8:18   0 1023M  0 part
	  └─md1                   9:1    0    2G  0 raid0
	    └─testvg-testr0     253:2    0  100M  0 lvm   /tmp/new
	sdc                       8:32   0    3G  0 disk
	├─sdc1                    8:33   0    2G  0 part
	│ └─md0                   9:0    0    2G  0 raid1
	└─sdc2                    8:34   0 1023M  0 part
	  └─md1                   9:1    0    2G  0 raid0
	    └─testvg-testr0     253:2    0  100M  0 lvm   /tmp/new
	sr0                      11:0    1 1024M  0 rom
	root@ubuntu-20:~#
	
## 15 Протестируйте целостность файла

	root@ubuntu-20:~# gzip -t /tmp/new/test.gz
	root@ubuntu-20:~# echo $?
	0
	
## 16 Используя pvmove, переместите содержимое PV с RAID0 на RAID1

	root@ubuntu-20:~# pvmove -n testr0 /dev/md1 /dev/md0
	  /dev/md1: Moved: 16.00%
	  /dev/md1: Moved: 100.00%

## 17 Сделайте --fail на устройство в вашем RAID1 md.

	root@ubuntu-20:~# mdadm /dev/md0 --fail /dev/sdb1
	mdadm: set /dev/sdb1 faulty in /dev/md0

## 18 Подтвердите выводом dmesg, что RAID1 работает в деградированном состоянии.

	[ 3161.648679] md/raid1:md0: Disk failure on sdb1, disabling device.
	               md/raid1:md0: Operation continuing on 1 devices.
								 
## 19 Протестируйте целостность файла, несмотря на "сбойный" диск он должен продолжать быть доступен

	root@ubuntu-20:~# gzip -t /tmp/new/test.gz
	root@ubuntu-20:~# echo $?
	0
