---
title: "git cherry pick"
date: 2021-09-08T14:30:00+03:00
draft: false
---

[Несколько версий продукта в Git](https://www.youtube.com/watch?v=qpr8iaEQXZU)

`git cherry-pick` это команда по копированию коммитов. Каждый коммит внутри git это некоторое состояние проекта, а cherry-pick вычисляет разницу (по сути каждый коммит это разница между состояниями) и применяет эту разницу в нужной ветке. В вышеуказанном видео упоминается `git cherry-pick -m` который позволяет скопировать merge-коммит в нужную ветку. Есть master, от него отпочковывается ветка, в ней идет разработка, несколько коммитов, потом эти коммиты сливаются в master и надо например все эти коммиты перенести в следующие ветки. Вот чтобы не таскать все эти коммиты можно скопировать один merge-коммит.

```
git checkout master
git merge --no-ff hotfix
git checkout release
git cherry-pick -m 1 hotfix-2-master-merge-commit-sha
```

![git merge + cherry-pick](/22.png)

Второй способ это `git merge --squash + cherry-pick`

`git merge -squash` делает все тоже самое что и обычный merge, только не помечает его как merge, т.е. у этого коммита один родитель.

```
git checkout master
git merge --squash hotfix
git checkout release
git cherry-pick hotfix-2-master-squash-commit-sha
```

![git merge squash + cherry-pick](/22_2.png)
