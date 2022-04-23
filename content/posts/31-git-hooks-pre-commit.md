---
title: "use pre-commit for git hooks"
date: 2022-04-23T16:51:00+03:00
draft: false
---

https://pre-commit.com

How to install

Запускать с PYTHONEXECUTABLE=~/.asdf/shims/python

```bash
$ asdf plugin add pre-commit
$ asdf install pre-commit latest
```

Если не ставится, то надо удалить `~/.asdf/plugins/pre-commit/bin/download` и еще раз попробовать

Создать `.pre-commit-config.yaml`

`repos` в нем это репо до hook plugins


```bash
$ pre-commit install
```

Может не завестись потому что питон не находит pre_commit module

Придется ставить руками `pip3 install pre_commit`

Может не завестись потому что при установке прописывает в `.git/hooks/pre-commit` странный путь до python


