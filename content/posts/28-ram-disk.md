---
title: "Ram Disk"
date: 2021-12-01T07:09:34+03:00
draft: false
---

RAM disk на 1Гб

2097152 = 2048 * 1024

```
diskutil erasevolume HFS+ "RAMDisk" `hdiutil attach -nomount ram://2097152`
```