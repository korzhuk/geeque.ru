---
layout: post
title: "Jails & PostgreSQL"
date: 2012-11-08 20:43
comments: true
categories: freebsd jails
---

Очередные странности FreeBSD, на этот раз - 9.1-RC1 (не стоит думать, что в релизной версии этого не было бы - проблема намного глубже).

Итак, имеем сервер на FreeBSD 9 и с десяток Jail-контейнеров в нем. На некоторых из них запущен PostgreSQL. В определенный момент времени (когда появляется сколь-либо заметная нагрузка) сразу несколько воркеров в разных (!) Jail'ах падают. В логах мы видим примерно следующее:

Со стороны Питона:
	TimeoutError: Unable to get database connection within 0 seconds. (OperationalError: FATAL:  the database system is in recovery mode
	) 

Или, что значительно круче:

	InterfaceError: connection already closed

Со стороны PostgreSQL:

	Nov  8 14:20:03 monitoring postgres[4691]: [5-3] HINT:  In a moment you should be able to reconnect to the database and repeat your command.
	Nov  8 14:21:16 monitoring postgres[4914]: [5-1] FATAL:  semctl(1245218, 6, SETVAL, 0) failed: Invalid argument
	Nov  8 14:21:16 monitoring postgres[4856]: [5-1] WARNING:  terminating connection because of crash of another server process
	Nov  8 14:21:16 monitoring postgres[4856]: [5-2] DETAIL:  The postmaster has commanded this server process to roll back the current transaction and exit, because another server process exited abnormally and possibly corrupted shared memory.


Причина возникновения подобной проблемы - отсутствие виртуализации Shared memory (shm) в Jail'ах. Точнее - мы имеем несколько запущеных в jail'ах (практически в chroot) процессов PostgreSQL, и в какой то момент времени сам PostgreSQL, работающей с моделью памяти (System V)[http://gentoo.theserverside.ru/book/system_5_shared_memory.html] залезает в свои сегменты Shared memory. К сожалению, он жестоко ошибается - эти сегменты не его (поскольку shm не виртуализуется - он один на всю систему), и были потроганы всеми, кто только мог до них дотянуться. Здесь он обижается и терминирует сам себя. К сожалению, после того, как он потрогал не свой сегмент данных в неправильных местах - сегмент видоизменяется, и им не может воспользоваться даже реальный владелец этого сегмента (читай - он тоже падает). 

So, what we can do? В общем-то, ничего. Мы не можем выделить отдельную shm память на каждый из контейнеров. Есть косвенное решение - изменить UID'ы поьзователя pgsql в разных контейнерах на разные. Это поможет хоть как-то разграничить сегменты выделяемой памяти - она все равно будет доступна всем, но ссылки на сегменты будут свои для каждого пользователя. Сделаем хотя бы это:

**Меняем ID для пользователя pgsql:**

__изменить можно на любой незанятый UID.__

	vipw /etc/passwd

**Меняем владельца домашней директории PostgreSQL:**

	chown -R pgsql /usr/local/pgsql/data

**Перезапускаем PostgreSQL:**

	/usr/local/etc/rc.d/postgres restart

Готово, сейчас проблема не должна проявляться так часто, но неизвестно, что будет, если shm начнет переполняться.

Виртуализация и FreeBSD - хреновая затея. Лучше не надо.