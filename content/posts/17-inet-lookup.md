---
title: "Как tcp находит свой сокет"
date: 2021-08-20T02:38:04+03:00
draft: false
---

Сетевой стек linux содержит 4 хэш-таблицы для быстрого поиска сокета для входящего пакета:

* `ehash` для установленных соединений
* `bhash` для bind
* `lhash2` для listen соединение как вида addr:port, так и any_addr:port
* `listening_hash` как простое хранилище listen-сокетов

[linux listen socket lookup](https://programmer.ink/think/change-of-tcp-listen-socket-lookup-in-linux.html)

Размер ehash таблицы регулируется параметром thash_entries. The default is 65536 entries for every gigabyte of memory. In other words, an 8-gigabyte system will have 512k hash entries. Но этого более чем достаточно.

А вообще все параметры можно посмотреть в [ip-sysctl.txt](https://elixir.bootlin.com/linux/v5.7/source/Documentation/networking/ip-sysctl.txt).