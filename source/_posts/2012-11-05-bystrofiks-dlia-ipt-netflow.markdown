---
layout: post
title: "Быстрофикс для Ipt_netflow"
date: 2012-11-05 11:47
comments: true
categories: 
---

В работе с этим модулем выяснилось, что при некоторых условиях (похоже, некорректное завершение слушающего процесса, но четких причин пока не выяснил), что он перестает отгружать статистику по своему сокету.
Проявляется это следующим образом:

**Нормальное состояние ipt_netflow:**

	Flows: active 317 (peak 1403 reached 0d5h31m ago), mem 27K
	Hash: size 8192 (mem 64K), metric 1.0, 1.0, 1.0, 1.0. MemTraf: 9476 pkt, 2642 K (pdu 73, 12534).
	Timeout: active 1800, inactive 15. Maxflows 2000000
	Rate: 419646 bits/sec, 242 packets/sec; Avg 1 min: 422334 bps, 185 pps; 5 min: 530004 bps, 185 pps
	cpu#  stat: , sock: , traffic: , drop: 
	Total stat: 230598 11701111 643411,    0    0    0    0, sock:   3798 0 78, 5429 K, traffic: 12344522, 7323 MB, drop: 10433296, 6472263 K
	cpu0  stat: 230598 11701111 643411,    0    0    0    0, sock:      0 0 0, 0 K, traffic: 12344522, 7323 MB, drop: 10433296, 6472263 K
	cpu1  stat:      0      0      0,    0    0    0    0, sock:   3798 0 78, 5429 K, traffic: 0, 0 MB, drop: 0, 0 K
	sock0: 127.0.0.1:9999, sndbuf 229376, filled 1, peak 1; err: sndbuf reached 0, other 0

**ipt_netflow с отпавшим сокетом:**

	Flows: active 249 (peak 1403 reached 0d4h14m ago), mem 21K
	Hash: size 8192 (mem 64K), metric 1.0, 1.0, 1.0, 1.0. MemTraf: 7273 pkt, 2099 K (pdu 57, 5697).
	Timeout: active 1800, inactive 15. Maxflows 2000000
	Rate: 226648 bits/sec, 126 packets/sec; Avg 1 min: 326792 bps, 137 pps; 5 min: 453795 bps, 157 pps
	cpu#  stat: , sock: , traffic: , drop: 
	Total stat: 181095 10582830 571842,    0    0    0    0, sock:   1415 0 78, 2023 K, traffic: 11154672, 6711 MB, drop: 10433296, 6472263 K
	cpu0  stat: 181095 10582830 571842,    0    0    0    0, sock:      0 0 0, 0 K, traffic: 11154672, 6711 MB, drop: 10433296, 6472263 K
	cpu1  stat:      0      0      0,    0    0    0    0, sock:   1415 0 78, 2023 K, traffic: 0, 0 MB, drop: 0, 0 K

К сожалению, выгрузить и загрузить модуль ядра не получится — получим ошибку:

	# rmmod ipt_NETFLOW
	ERROR: Module ipt_NETFLOW is in use

Избавиться от нее можно лишь выгрузив x_tables, что в случае с роутером вызовет отказ форвардинга — проще перезагрузить тачку. Выгружать модуль с флагом —force тоже не советую — после кернел паника машину перезагружать придется точно, проверял:)

**Фиксим через передергивание в sysctl:**

	sysctl -w net.netflow.destination=127.0.0.1:9000
	sysctl -w net.netflow.destination=127.0.0.1:9999

**Справедливость восстановлена:**

	sock0: 127.0.0.1:9999, sndbuf 229376, filled 1, peak 1; err: sndbuf reached 0, other 0