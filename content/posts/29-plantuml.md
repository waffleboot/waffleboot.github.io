---
title: "запустить plantuml из под docker"
date: 2022-04-23T14:25:00+03:00
draft: false
---

```bash
$ docker run -d -p 8080:8080 --name plantuml --read-only -v /tmp/jetty plantuml/plantuml-server:jetty
```

Запустить в Visual Studio Code preview - `option + d`

https://github.com/plantuml/plantuml-server

http://localhost:8080
