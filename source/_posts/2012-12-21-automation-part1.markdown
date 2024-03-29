---
layout: post
title: "Автоматизация работы с серверами. Часть 1. Установка ОС на серверы."
date: 2012-11-21 12:47
comments: true
categories: linux fai puppet python fabric
---

При введении нового сервера в кластер необходимо руководствоваться простым правилом — запускай хост в известном состоянии. То есть, мы должны знать, что на всех этапах установки были соблюдены все зависимости приложения, которое должно будет работать на нем. Без соблюдения этого правила мы рискуем получить кучу задней боли при поддержке и шишку на голове от тестировщиков.


Для установки ОС в автоматическом и полуавтоматическом режиме существует достаточно много средств:

* **Hetzner installimage**
* **Debian preseed**
* **kickstart**
* **FAI (Fully Automated Installation)**

## Hetzner installimage ##
 **Hetzner installimage** был разработан Флорианом Уиком для довольно известного в Европе хостинга Hetzner Online. Написан на bash'e, доступен под лицензией GPL. К сожалению, я не имею понятия, где его скачать - он доступен напрямую из Rescue-режима на серверах в Hetzner, но в публичном доступе мне его найти не удалось. Если кто-то найдет этот скрипт, или его домашнюю страничку - напишите мне на почту или в джаббер.

 Установка производится в полуавтоматическом режиме, что заметно уменьшает ценность скрипта. Существует также [костыль, нежно называемый workaround'ом на ruby](https://github.com/rmoriz/hetzner-bootstrap) для автоматизации установки серверов в Hetzner именно этим скриптом.

Для работы этого скрипта ему необходимо передать файл конфигурации, который включает в себя:

1. Используемые на сервере диски
1. Тип RAID-а на сервере
1. Разметку дисков (в том числе и LVM)
1. Ссылку на образ, который необходимо развернуть.

Собственно, что делает данная балалайка:

1. Размечает диски
1. Формирует Soft-RAID 
1. Монтирует партиции
1. Разливает указанный образ в корень target-системы
1. Выполняет минимальную конфигурацию системы (сетевые интерфейсы, пароли, hostname)
1. Устанавливает загрузчик.

Большая проблема, которая возникает при использовании — необходимость создания образа для развертки. Складывается эта проблема из нескольких мелких:

* В случае использования нескольких классов серверов нам нужно иметь не просто несколько конфигураций, но ещё и несколько файлов образов.
* Образы имеют неиллюзорно большой размер.
* При установке нескольких серверов одновременно есть вероятность того, что bottleneck скорости установки обнаружится в канале раздающего образы сервера.
* Самая неприятная проблема - обновление образов. Делать это необходимо вручную, отслеживая, чтобы ничего лишнего (к примеру, маленький публичный ключик, или bash_history) не попало в этот образ.

В целом, это решение годится, если есть желание все-таки следить за процессом установки серверов, но не тратить время на конфигурацию пакетов, или если количество серверов позволяет тратить время на установку ОС в таком режиме. Устанавливать может любые *nix-ОС - работает с системой на уровне файлов.

## Debian preseed ##

**Preseed,** по своей сути, автоматизатор стандартного установщика Debian. Параметры для установки можно передать следующими способами:

* initrd.gz. Страшно. Ребилдить initramfs при каждом изменении конфигурации - очень громоздко. 
* DHCP-опции. Достаточно удобно. Подробнее [здесь](http://hands.com/d-i/)
* Установка пути до preseed-файла вручную в процессе установки. Не круто.

Подробнее об этом методе установки можно прочитать в [официальной документации Debian](http://wiki.debian.org/DebianInstaller/Preseed).

К сожалению, этот метод очень громоздкий, проводить изменения в конфигурационном файле неудобно, сам файл — на удивление нечитабельный. Ко всему прочему, может устанавливать только Debian-based дистрибутивы. Примеры конфигураций: [layer-acht.org](http://layer-acht.org/d-i/preseeding-examples/)


##kickstart##

**kickstart** является аналогом preseed, но для RPM-based дистрибутивов (RHEL, CentOS). Конфигурация мало чем отличается от preseed, но является все же более удобной за счет лучшей читаемости ks-файла перед preseed-файлом. Устанавливать может только RPM-based ОС. Для интересующихся этим способом установки - рекомендую [эту статью](http://www.opennet.ru/base/sys/pxe_redhat_install.txt.html)

##FAI (Fully Automated Installation) ##
**FAI** является героем [этой отдельной статьи](http://geeque.ru/blog/automation-part-1.1-fai)  — он включает в себя все плюсы вышеописанных средств неинтерактивной установки. По сути, FAI - это набор скриптов, производящих установку по предоставленной им конфигурации. 

Для его работы на сервере создается окружение, содержащее все необходимые установщику пакеты, и точка монтирования окружения экспортируется по NFS на удаленный сервер. Общая процедура установки ОС с помощью  FAI выглядит следующим образом:

1. Загрузка ядра по PXE.
1. Монтирование удаленного NFS-ресурса с установщиком, необходимыми модулями и пакетами.
1. Выбор конфигурации для установке на базе маски hostname, который выдает DHCP.
1. Загрузка конфигурации по NFS.
1. Разметка диска в соответствии с выбранной конфигурацией.
1. debootstrap целевой системы c указанными в конфигурации пакетами на размеченный жесткий диск.
1. Завершение установки с помощью postinstall-скриптов.
1. Загрузка в целевую систему.

Конфигурацию FAI достаточно легко редактировать -  она является набором файлов, относящихся к определенному классу серверов. Концепция классов - невероятно гибкая. К примеру, она позволяет разделить, какие пакеты будут нужны воркер-серверам на аппаратной платформе, а какие - серверам на гипервизоре. Выбор класса делается на этапе начала установки на основе hostname, переданному от DHCP.

Итак, для того, чтобы инициировать установку для сервера, нам необходим только его MAC-адрес. Связку mac-hostname мы укажем в нашем DHCP-сервере. Больше никаких действий для установки ОС на сервере нам предпринимать не нужно (разве что, перезагрузить сервер)!

Описание настройки FAI читай в [этой статье](http://geeque.ru/blog/automation-part-1.1-fai) 




