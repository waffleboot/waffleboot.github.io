---
title: "Avito messenger"
date: 2021-08-11T04:40:32+03:00
draft: false
---

[Архитектура Мессенджера Авито – путь одного сообщения](https://www.youtube.com/watch?v=4tIS58sQ7Mc)

* 2,5 млн уников в день
* 1,5 млн rpm rpc запросов к бэку
* 500k постоянных соединений онлайн в пике websocket
* 7k msg/m

-

* rabbitmq
* mongodb
* redis
* k8s


websocket

json-rpc

server-socket

http-polling (http fallback)

[stripe.com/rate-limiters](https://stripe.com/blog/rate-limiters)

service-aggregator (+graceful degradation)

* сначала сохранение сообщения только в шард отправителя
* публикация событий в rabbitmq

rabbitmq отказоустойчивый кластер из двух машин  
настроены политики отказоустойчивости для exchange и очередей  
retry

rabbitmq DLX+TTL (dead letter exchange)

система каскадных очередей, каждая из которых экспоненциально увеличивает TTL

федерация rabbitmq

rabbitmq написан на Erlang, в компании никто не знает Erlang

[centrifugo](https://github.com/centrifugal/centrifugo)

конференция golang.conf январь 2020 г. Александр Емелин про centrifugo

redis pipeline

lua-процедура

* LPUSH
* LTRIM
* EXPIRE
* PUBLISH (PUBSUB)

redis pubsub буферы ограниченного размера 16mb

как сделаны суперчаты на 100 тыс пользователей?

* идемпотентность - наше всё

