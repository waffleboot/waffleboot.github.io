---
title: "Golang interfaces"
date: 2021-08-30T02:38:04+03:00
draft: false
---

```
if x, ok := obj.(interface{ MyMethod(int) bool }); ok && x.MyMethod(42) {
    ...
}
```

Т.е. можно определять интерфейс на лету если он используется только в одной точке. Но и проверку на соответствие интерфейсу наружу тоже не вынести.

```
type MyInterface interface {
    MyMethod(int) bool
}
```