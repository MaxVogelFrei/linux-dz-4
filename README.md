# Домашнее задание 4

#### [лог переименования VG](typescript)
#### [лог добавления модуля](typescript2)


## Работа с загрузчиком

* Попасть в систему без пароля несколькими способами
* Установить систему с LVM, после чего переименовать VG
* Добавить модуль в initrd

## Процесс решения

### Попасть в систему без пароля несколькими способами

Во время перезапуска, при появлении меню выбора ядра нажимаю "e", попадаю в редактор опций загрузчика\
![Screen1](VirtualBox_linux-dz-4_lvm_1574089751254_11621_19_11_2019_17_46_50.png)

#### Способ 1

Перехожу в строку "linux16"\
из нее убираю опцию "console=ttyS0,115200n8"\
в конец добавляю "init=/bin/sh"\
![Screen2](VirtualBox_linux-dz-4_lvm_1574089751254_11621_19_11_2019_17_55_35.png)\
нажимаю ctrl-x и после загрузки получаю консоль\
ввожу

		mount -o remount,rw /

получаю доступ на запись к ФС

		touch /root/file
		rm file


#### Способ 2

Перехожу в строку linux16\
убираю опцию console=ttyS0,115200n8\
в конец добавляю rd.break\
нажимаю ctrl-x и после загрузки получаю консоль\
![Screen3](VirtualBox_linux-dz-4_lvm_1574089751254_11621_20_11_2019_17_19_25.png)


#### Способ 3
Перехожу в строку linux16\
заменяю ro на rw\
убираю опцию console=ttyS0,115200n8\
в конце строки добавляю init=/sysroot/bin/sh\
получаю консоль с корневой ФС с доступом на запись

### Установить систему с LVM, после чего переименовать VG

После входа в консоль с правами root смотрю текущее название VG

		[root@lvm ~]# vgs
		  VG         #PV #LV #SN Attr   VSize   VFree
		  VolGroup00   1   2   0 wz--n- <38.97g    0 

Переименовываю VG

		[root@lvm ~]# vgrename VolGroup00 OtusRoot
		  Volume group "VolGroup00" successfully renamed to "OtusRoot"

Меняю название VG в конфигурационных файлах

		[root@lvm ~]# vi /etc/fstab
		[root@lvm ~]# cat /etc/fstab
		..
		/dev/mapper/OtusRoot-LogVol00 /                       xfs     defaults        0 0
		UUID=570897ca-e759-4c81-90cf-389da6eee4cc /boot                   xfs     defaults        0 0
		/dev/mapper/OtusRoot-LogVol01 swap                    swap    defaults        0 0
		..

		[root@lvm ~]# vi /etc/default/grub
		[root@lvm ~]# cat /etc/default/grub
		..
		GRUB_CMDLINE_LINUX="no_timer_check console=tty0 console=ttyS0,115200n8 net.ifnames=0 biosdevname=0 elevator=noop crashkernel=auto rd.lvm.lv=OtusRoot/LogVol00 rd.lvm.lv=OtusRoot/LogVol01 rhgb quiet"
		..


		[root@lvm ~]# vi /boot/grub2/grub.cfg
		[root@lvm ~]# cat /boot/grub2/grub.cfg
		..
			linux16 /vmlinuz-3.10.0-862.2.3.el7.x86_64 root=/dev/mapper/OtusRoot-LogVol00 ro no_timer_check console=tty0 console=ttyS0,115200n8 net.ifnames=0 biosdevname=0 elevator=noop crashkernel=auto rd.lvm.lv=OtusRoot/LogVol00 rd.lvm.lv=OtusRoot/LogVol01 rhgb quiet 
		..

пересоздаю образ initrd

		[root@lvm ~]# mkinitrd -f -v /boot/initramfs-$(uname -r).img $(uname -r)
		*** Creating image file done ***
		*** Creating initramfs image file '/boot/initramfs-3.10.0-862.2.3.el7.x86_64.img' done ***

перезапускаю машину, после включения проверяю имя VG

		[root@lvm ~]# vgs
		  VG       #PV #LV #SN Attr   VSize   VFree
		  OtusRoot   1   2   0 wz--n- <38.97g    0 

### Добавить модуль в initrd

Создаю папку для своего модуля\
		[root@lvm ~]# mkdir /usr/lib/dracut/modules.d/01test
		[root@lvm ~]# cd /usr/lib/dracut/modules.d/01test
создаю файлы скриптов

	установщик\
		[root@lvm 01test]# vi module-setup.sh
		[root@lvm 01test]# cat module-setup.sh
		#!/bin/bash
		
		check() {
		    return 0
		}
		
		depends() {
		    return 0
		}
		
		install() {
		    inst_hook cleanup 00 "${moddir}/test.sh"
		}

	сам скрипт\
		[root@lvm 01test]# vi test.sh
		[root@lvm 01test]# cat test.sh
		#!/bin/bash
		
		exec 0<>/dev/console 1<>/dev/console 2<>/dev/console
		cat <<'msgend'
		Hello! You are in dracut module!
		 ___________________
		< I'm dracut module >
		 -------------------
		   \
		    \
		        .--.
		       |o_o |
		       |:_/ |
		      //   \ \
		     (|     | )
		    /'\_   _/`\
		    \___)=(___/
		msgend
		sleep 10
		echo " continuing...."

пересобираю образ initrd\
		[root@lvm 01test]# dracut -f -v
		*** Creating image file done ***
		*** Creating initramfs image file '/boot/initramfs-3.10.0-862.2.3.el7.x86_64.img' done ***
		
убираю опции rhgb quiet из /etc/default/grub и /boot/grub2/grub.cfg\
перезапускаюсь, при старте вижу, что модуль работает

![Screen4](VirtualBox_linux-dz-4_lvm_1574260838984_47690_20_11_2019_18_33_52.png)
