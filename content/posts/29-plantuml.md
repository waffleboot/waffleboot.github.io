---
title: "запустить plantuml из под docker"
date: 2022-04-23T14:25:00+03:00
draft: false
---

```bash
$ docker run -d -p 8080:8080 --read-only -v /tmp/jetty plantuml/plantuml-server:jetty
```
