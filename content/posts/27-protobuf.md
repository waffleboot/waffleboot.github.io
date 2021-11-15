---
title: "protobuf"
date: 2021-11-15T15:11:51+03:00
draft: false
---

Пытаюсь понять как работает grpc-gateway

```
syntax = "proto3";
import   "google/api/annotations.proto";
service Service {
    rpc Method(Request) returns (Response) {
        option (google.api.http) = {
          post: "/endpoint"
          body: "*"
        };
      }
}
```

Оказывается:

- protoc это компилятор и содержит <span>parser.cc</span>, который парсит .proto файлы и `option` поначалу выгрызает как строку, как `uninterpreted_option`
- затем работает `DescriptorBuilder` и он эту строку парсит и превращает в message
- и проставляет полученный message в http поле
- поэтому в grpc-gateway плагин приходит описание без uninterpreted_option, а в виде описания опции метода, которое содержит поле http
- а это поле http появляется там из-за механизма extension
- annotations.proto расширяет MethodOptions дополнительным атрибутом http
- а grpc-gateway уже читает это поле как extension
