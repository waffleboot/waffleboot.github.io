---
title: "git rebase onto"
date: 2022-04-23T15:00:32+03:00
draft: false
---

Есть три ветки

![три ветки](/30.png)

Можно, находясь в `master`, делать `rebase` ветки `exp/feature` поверх `feature`

```bash
$ git rebase --onto feature feature exp/feature
Successfully rebased and updated refs/heads/exp/feature.
```

```plantuml
@startuml
hide empty description
[*] -right->  C1
C1 -right-> C2
C2 -right-> C3
C2 -up-> C4
C4 -right-> C5
C5 -right-> C6
C5 -up-> C7
C7 -right-> C8
note right of C8
exp/feature
end note
note right of C3
master
end note
note right of C6
feature
end note
@enduml
```