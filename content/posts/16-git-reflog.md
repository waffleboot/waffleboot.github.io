---
title: "Git Reflog"
date: 2021-08-20T01:05:06+03:00
draft: false
---

TL;DR: to restore git history

* `git reflog master`
* `git reset --hard @{1}`

# git reflog

git все изменения в refs (ссылках) пишет в логи `.git/logs/`  
Например, изменения в HEAD (или .git/HEAD) пишутся в `.git/logs/HEAD`  
Обычно `git reflog` показывает только историю HEAD

```
4939906 HEAD@{0}: checkout: moving from test to master
494e06a HEAD@{1}: checkout: moving from master to test
4939906 HEAD@{2}: checkout: moving from 379e743595f56cb0b4177fd248c84c47523a4e6b to master
379e743 HEAD@{3}: checkout: moving from master to 379e743595f56cb0b4177fd248c84c47523a4e6b
4939906 HEAD@{4}: clone: from https://github.com/waffleboot/git_rotten_dump.git
```
Это я сделал `git clone`, потом `git co 379e743` и в ветку test туда и обратно. git reflog показывает историю HEAD и можно откатиться на предыдущее состояние с помощью `git reset --hard HEAD@{x}`, тут HEAD@{x} показывает на нужный коммит, но можно и прямо sha указать. Вообще HEAD@{x} не имеет отношения к reflog, а просто способ указать ревизию, коммит. HEAD@{x} можно использовать везде.

Но git reflog может показать не только историю HEAD, а вообще историю веток. Только будет это `git reflog master`.

```
4939906 master@{0}: clone: from https://github.com/waffleboot/git_rotten_dump.git
```

Чтобы вернуться к состоянию после rebase можно использовать

```
git reflog master
git reset --hard master@{1}
```

git reflog покажет всю активность в HEAD, а в reflog master отложится только самое последнее стабилизирующее действие, потому что чтобы сделать rebase git сохранит HEAD в ORIG_HEAD, дальше начнет лить коммиты, при этом HEAD конечно же меняется, а значит вся эта история попадает в reglog HEAD, а вот в конце git перезаписывает master ref и уже там отложится финальная история, поэтому master@{1} позволяет искать предыдущее состояние по более короткой истории.

### [Какие есть способы указать git revisions](https://git-scm.com/docs/gitrevisions)

#### \<refname>, e.g. master, heads/master, refs/heads/master

A symbolic ref name. E.g. **master** typically means the commit object referenced by **refs/heads/master**. If you happen to have both **heads/master** and **tags/master**, you can explicitly say **heads/master** to tell Git which one you mean. When ambiguous, a **\<refname>** is disambiguated by taking the first match in the following rules:

1. If **.git/\<refname>** exists, that is what you mean (only for `HEAD`, `FETCH_HEAD`, `ORIG_HEAD`, `MERGE_HEAD` and `CHERRY_PICK_HEAD`);

1. **refs/\<refname>**

1. **refs/tags/\<refname>**

1. **refs/heads/\<refname>**

1. **refs/remotes/\<refname>**

1. **refs/remotes/\<refname>/HEAD**

* `HEAD` names the commit on which you based the changes in the working tree.
* `FETCH_HEAD` records the branch which you fetched from a remote repository with your last `git fetch` invocation.
* `ORIG_HEAD` is created by commands that move your `HEAD` in a drastic way, to record the position of the `HEAD` before their operation, so that you can easily change the tip of the branch back to the state before you ran them.
* `MERGE_HEAD` records the commit(s) which you are merging into your branch when you run `git merge`.
* `CHERRY_PICK_HEAD` records the commit which you are cherry-picking when you run `git cherry-pick`.

Note that any of the **refs/*** cases above may come either from the **.git/refs** directory or from the **.git/packed-refs** file.

#### @

**@** alone is a shortcut for `HEAD`

#### \<refname>@{\<n>}, e.g. master@{1}

A ref followed by the suffix **@** with an ordinal specification enclosed in a brace pair (e.g. **{1}, {15}**) specifies the n-th prior value of that ref. For example **master@{1}** is the immediate prior value of master while **master@{5}** is the 5th prior value of master. This suffix may only be used immediately following a ref name and the ref must have an existing log (**.git/logs/\<refname>**).


#### @{\<n>}, e.g. @{1}

You can use the **@** construct with an empty ref part to get at a reflog entry of the current branch. For example, if you are on branch **blabla** then **@{1}** means the same as **blabla@{1}**.


#### @{-\<n>}, e.g. @{-1}

The construct **@{-\<n>}** means the <n>th branch/commit checked out before the current one.

Т.е. можно исправить ошибочный rebase командой

```
git reset --hard @{1}
```
