---
layout: post
title: "Автоматизация работы с серверами. Часть 1.1 Установка и конфигурация FAI"
date: 2012-11-23 15:47
comments: true
categories: linux fai puppet python fabric
---

Итак, что же представляет из себя процедура установки через FAI?

 Для наглядности сделаем схему: 


{% img http://geeque.ru/images/blog/fai.png %}

1. Новый сервер запрашивает у DHCP-сервера свою конфигурацию сети. В ответе, помимо параметров сети, он получает также и конфигурацию PXE, позволяющую загрузить ядро по сети.

1. Производится загрузка ядра из конфигурации pxelinux.

1. initramfs, загруженный из конфигурации pxelinux, монтирует в качестве корневой файловой системы NFS-ресурс. Начинает загружаться окружение fai-installer (не более, чем Debian c необходимыми пакетами)

1. fai-installer монтирует NFS-ресурс с конфигурацией. Ради справедливости нужно сказать, что конфигурации могут доставляться и по http, git, svn, но read-only NFS является довольно хорошим решением в масштабах локальной сети.

1. После разметки, форматирования и подготовки сервера к установке, производится развертка base-образа (по своему содержанию он похож на stage3 от gentoo). По окончании данного процесса начинает работу debootstrap, выполняя установку пакетов, указанных в классах, соответствующих хосту.

Начнем с первого пункта.

##Конфигурация DHCP-сервера##


Исходные данные:

Используется подсеть 172.16.0.0/255.255.0.0
Пул для неизвестных хостов - 172.16.20.0/255.255.255.0
Все известные хосты привязаны к своим MAC-адресам.
ОС - Ubuntu 12.04.1
Собственный IP DHCP-сервера - 172.16.4.1, также он является шлюзом по умолчанию.
FAI-сервер располагается по адресу 172.16.4.2
Интерфейс, направленный во внутреннюю сеть - eth1.

1. Устанавливаем DHCP-сервер:

       # aptitude install isc-dhcp-server

1. Конфигурируем интерфейсы, на которых будут приниматься DHCP Request-пакеты:

        # nano /etc/default/isc-dhcp-server
Изменяем значение в переменной INTERFACES на eth1:

        INTERFACES="eth1"

1. Настраиваем DHCP-сервер и пулы адресов:

        # nano /etc/dhcp/dhcpd.conf

Пример рабочей конфигурации:

        ddns-update-style none;

        option domain-name "testdomain.com";
        option domain-name-servers 8.8.8.8; ##TODO: set to local resolver

        default-lease-time 600;
        max-lease-time 7200;

        authoritative;

        log-facility local7;


        subnet 172.16.0.0 netmask 255.255.0.0 {
          range 172.16.20.1 172.16.20.254;
          option routers 172.16.4.1;
          option time-servers 172.16.4.2;
          option ntp-servers 172.16.4.2;
          next-server 172.16.4.2;
          filename "fai/pxelinux.0";
        }


        host fai1 {
          hardware ethernet 08:00:27:07:e6:65;
          fixed-address 172.16.4.2;
        }

        host test2 {
          hardware ethernet 08:00:27:c7:94:98;
          fixed-address 172.16.4.3;
          option host-name gt-spb-test2;
        }
        

Обратим внимание на строки:

        next-server 172.16.4.2;
        filename "fai/pxelinux.0";

в конфигурации подсетей — они необходимы для корректной загрузки по PXE (хост с IP 172.16.4.2 является нашим FAI-сервером).

Также, в конфигурации фиксиованных хостов есть следующий параметр:
        
        option host-name test2;

Он необходим для передачи хосту при запросе его DHCP ещё и его hostname - подобным образом FAI определяет набор пакетов для данного хоста, и устанавливает этот hostname в целевую систему.

1. Перезагружаем DHCP-сервер:

        # /etc/init.d/isc-dhcp-server restart

Готово, сейчас адреса и параметры для загрузки по PXE должны передаваться корректно.

Переходим к установке самого FAI.

##Установка FAI##

Как уже было сказано ранее, FAI в процессе загрузки монтирует для себя две директории по NFS: **nfsroot** и **nfsconfig**. Для загрузки ядра там также необходимо передать его в PXE через tftp-сервер.

Использовать будем следующие директории:

* nfsroot: /srv/fai/nfsroot
* nfsconfig: /srv/fai/config
* tftproot: /srv/tftp/fai

 В Debian и Ubuntu установка FAI сводится к простой установке мета-пакета fai-quickstart:

    # aptitude install fai-quickstart

_Сомнительным выглядит решение включить в этот мета-пакет ещё и isc-dhcp-server (чаще всего он находится на шлюзе, а архитектура, в которой хост с FAI будет отвечать за раздачу адресов по DHCP представить сложно), но ничто не мешает нам  деактивировать данный сервис :-)_

 Выполняем в том случае, если DHCP активирован на другом узле:

    # update-rc.d -f isc-dhcp-server remove && update-rc.d -f isc-dhcp-server6 remove

**Редактируем конфигурацию скрипта создания среды:**

    # nano /etc/fai/make-fai-nfsroot.conf

Описание не всех, но частоиспользуемых переменных, находящихся в данном файле:

| Название параметра | Описание |
|:--------------:|:---------------:|
|NFSROOT|Директория, в которой будет располагаться корень установщика|
|TFTPROOT|Директория, в которой будет раcполагаться PXE-загрузчик и его конфигурация|
|FAI_CONFIGDIR|Директория, в которой будут располагаться файлы конфигурации классов и хостов|
|NFSROOT_ETC_HOSTS|Строка, которая будет добавлена в файл /etc/hosts установщика (необходимо, если DNS не будет доступен в среде установщика)|
|FAI_DEBOOTSTRAP|Параметры, передаваемые в debootstrap. Для Ubuntu 12.04: **precise http://ru.archive.ubuntu.com/ubuntu**|
|FAI_ROOTPW|Пароль для пользователья root на целевой системе после установки. MD5 или Crypt-формат. Пароль по умолчанию - fai|
|SSH_IDENTITY|Расположение файла identity.pub для логина в сам установщик.|
|FAI_DEBOOTSTRAP_OPTS|Параметры, передаваемые в debootstrap дополнительно|

Рабочее содержание этого файла:

    NFSROOT=/srv/fai/nfsroot
    TFTPROOT=/srv/tftp/fai
    FAI_CONFIGDIR=/srv/fai/config
    FAI_DEBOOTSTRAP="precise http://mirror.yandex.ru/ubuntu"
    FAI_ROOTPW='$1$kBnWcO.E$djxB128U7dMkrltJHPf6d1'
    FAI_DEBOOTSTRAP_OPTS="--exclude=info,dhcp-client --include=aptitude"

**Редактируем настройки среды, которая будет передана в установщик FAI:**

    # nano /etc/fai/fai.conf

Описание переменных в этом файле:


| Название параметра | Описание |
|:--------------:|:---------------:|
|LOGUSER|Пользователь в среде установщика, который сохраняет все логи установщика и имеет право беспарольного входа в него.|
|FAI_LOGPROTO|Протокол для сохранения логов|
|FAI_DEBMIRROR|Путь до локального зеркала дистрибутива (NFS/HTTP/FTP). Если не указать - использоваться будет адрес зеркала из хост-системы|
|MNTPOINT|Точка монтирования локального зеркала|
|FAI|Директория с конфигурациями установщика|
|FAI_CONFIG_SRC|Откуда эта конфигурация будет монтироваться|

Рабочее содержание достаточно краткое, но этого достаточно:

    FAI_LOGPROTO=ssh
    FAI=/var/lib/fai/config
    FAI_CONFIG_SRC=nfs://172.16.4.2/srv/fai/config

Переходим к важному шагу - создание рабочей среды установщика:

**Выполняем создание среды:**
    
    # fai-setup -v

В процессе мы получим примерно следующий вывод в STDOUT:

    faiserver[~]# fai-setup -v
    Using configuration files from /etc/fai
    Creating FAI nfsroot in /srv/fai/nfsroot/live/filesystem.dir.
    By default it needs more than 380 MBytes disk space.
    This may take a long time.
    Creating base system using debootstrap version 1.0.10lenny1
    Calling debootstrap lenny /srv/fai/nfsroot/live/filesystem.dir http://cdn.debian.net/debian
    .
    Creating base.tgz
    .
    Upgrading /srv/fai/nfsroot/live/filesystem.dir
    .
    nfs-common fai-nfsroot module-init-tools ssh rdate lshw portmap rsync lftp less dump reiserfsprogs e2fsprogs usbutils hwinfo psmisc pciutils hdparm smartmontools parted mdadm lvm2 dnsutils ntpdate dosfstools jove xfsprogs xfsdump procinfo dialog discover console-tools console-common iproute udev subversion liblinux-lvm-perl cfengine2 libapt-pkg-perl grub lilo read-edid linux-image-486 aufs-modules-2.6-486
    install_packages: reading config files from directory /etc/fai
    install_packages: read config file NFSROOT
    .
    .
    `/etc/fai/NFSROOT' -> `/srv/fai/nfsroot/live/filesystem.dir/etc/fai/NFSROOT'
    `/etc/fai/apt' -> `/srv/fai/nfsroot/live/filesystem.dir/etc/fai/apt'
    `/etc/fai/apt/sources.list' -> `/srv/fai/nfsroot/live/filesystem.dir/etc/fai/apt/sources.list'
    `/etc/fai/fai.conf' -> `/srv/fai/nfsroot/live/filesystem.dir/etc/fai/fai.conf'
    `/etc/fai/live.conf' -> `/srv/fai/nfsroot/live/filesystem.dir/etc/fai/live.conf'
    `/etc/fai/make-fai-nfsroot.conf' -> `/srv/fai/nfsroot/live/filesystem.dir/etc/fai/make-fai-nfsroot.conf'
    `/etc/fai/menu.lst' -> `/srv/fai/nfsroot/live/filesystem.dir/etc/fai/menu.lst'
    Shadow passwords are now on.
    Removing `local diversion of /usr/sbin/update-initramfs to /usr/sbin/update-initramfs.distrib'
    update-initramfs: Generating /boot/initrd.img-2.6.26-2-486
    W: mdadm: unchecked configuration file: /etc/mdadm/mdadm.conf
    W: mdadm: please read /usr/share/doc/mdadm/README.upgrading-2.5.3.gz .
    W: mkconf: MD subsystem is not loaded, thus I cannot scan for arrays.
    W: mdadm: failed to auto-generate temporary mdadm.conf file.
    W: mdadm: no configuration file available.
    `/srv/fai/nfsroot/live/filesystem.dir/boot/vmlinuz-2.6.26-2-486' -> `/srv/tftp/fai/vmlinuz-2.6.26-2-486'
    `/srv/fai/nfsroot/live/filesystem.dir/boot/initrd.img-2.6.26-2-486' -> `/srv/tftp/fai/initrd.img-2.6.26-2-486'
    DHCP environment prepared. If you want to use it, you have to enable the dhcpd and the tftp-hpa daemon.
    Removing `local diversion of /sbin/discover-modprobe to /sbin/discover-modprobe.distrib'
    make-fai-nfsroot finished properly.    <=== *
    No diversion `any diversion of /sbin/discover-modprobe', none removed
    Log file written to /var/log/fai/make-fai-nfsroot.log
    Re-exporting directories for NFS kernel daemon....

       You have no FAI configuration space yet. Copy the simple examples with:
       cp -a /usr/share/doc/fai-doc/examples/simple/* /srv/fai/config
       Then change the configuration files to meet your local needs.
    Please don't forget to fill out the FAI questionnaire after you've finished your project with FAI.

    FAI setup finished.                    <=== *
    Log file written to /var/log/fai/fai-setup.log

Важно, чтобы в выводе были строки, отмеченные звездочкой — это означает, что установка завершилась корректно.

В результате работы данного скрипта мы получим:

* Готовое окружение загрузчика — экспортированную по NFS директорию /srv/fai/nfsroot
* Набор шаблонов для хостов (подробнее они будут рассмотрены в следующей статье) — экспортированная по NFS директория /srv/fai/config
* Ядро, initramfs и конфигурация PXE, которые возможно будет передать по TFTP. Находятся в директории /srv/tftp/.

Переходим к конфигурации TFTP.

**Конфигурируем TFTP:**

Конфигурация демона TFTP располагается в файле /etc/default/tftpd-hpa, редактируем его:

    # nano /etc/default/tftpd-hpa

Смысл переменных очевиден, и в пояснении не нуждается. Нам необходимо изменить переменную TFTP_DIRECTORY.

Пример рабочего файла:

    TFTP_USERNAME="tftp"
    TFTP_DIRECTORY="/srv/tftp/"
    TFTP_ADDRESS="0.0.0.0:69"
    TFTP_OPTIONS="--secure

**Перезапуаскаем TFTP:**

    # /etc/init.d/tftpd-hpa restart


**Конфигурируем NFS:**

В процессе выполнения fai-setup скрипт должен был создать необходимые экспортируемые ресурсы, проверяем, так ли это. Содержание файла /etc/exports должно быть примерно похоже на следующее:

    /srv/fai/nfsroot 172.16.0.0/16(async,ro,no_subtree_check)
    /srv/fai/config  172.16.0.0/16(async,ro,no_subtree_check,no_root_squash)

**Экспортируем точки монтирования:**

    # exportfs -a

**Активируем конфигурацию в TFTP/PXE:**

Данная команда создает PXE-конфигурацию для хоста test1 в директории /srv/tftp/fai/pxelinux.cfg/:

    # fai-chboot -IFv test1

IP-адрес, соответствующий хосту, преобразуется в hex-формат, после чего в указанной директории создается файл с таким именем, и наполняется примерно следующим содержимым:

    # generated by fai-chboot for host gt-spb-test1 with IP 172.16.0.1
    default fai-generated

    label fai-generated
    kernel vmlinuz-3.2.0-35-generic
    append initrd=initrd.img-3.2.0-35-generic ip=dhcp  root=/dev/nfs nfsroot=/srv/fai/nfsroot boot=live  FAI_FLAGS=verbose,sshd,createvt FAI_ACTION=install

Для корректной работы установщика в DHCP-ответе для PXE необходимо передать hostname. Как это сделать - смотрим в начале статьи (option host-name).

В результате всех действий мы получим загружаемый по сети установщик, устанавливающий на хост ОС по стандартному шаблону. Какие именно шаблоны будут использоваться, и как именно их нужно настраивать - рассмотрим в [следующей статье](http://geeque.ru/blog/automation-part1.2-patterns/)