---
title: 'IDA8.4迁移到IDA9.0 i64 bug'
date: 2025-02-20 11:02:20
category: api
tags: [IDA]
published: true
hideInList: false
feature: 
isTop: false
---

## 记录

IDA8.4迁移到IDA9.0时，database的`view's history`好像会迁移出一点问题，在按`Esc`的时候会出现如下报错

```
Navigation failed. This can happen if the view's history was corrupted. If you suspect this is a bug, please send this database to <support@hex-rays.com>
```

解决办法就是，在`IDA9.0`中，不停的往`view's history`里面堆`history`，这样就不会在`Esc`显示报错了

```python
import ida_kernwin
import idaapi
for i in range(0x10000):
    ida_kernwin.ea_viewer_history_push_and_jump(ida_kernwin.get_current_viewer(), idaapi.get_name_ea(0, "main") , 0, i, 0)
```

力大砖飞，只要我塞的`history`够多，这样`Esc`就按不到报错位置🤣