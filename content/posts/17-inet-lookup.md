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
