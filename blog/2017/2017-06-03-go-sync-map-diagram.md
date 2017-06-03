categories:
- 技术

tags:
- Go

title: 图解 Go 新增的并发安全的字典 sync.Map
---

[sync.Map][1] 是并发安全的字典，于 2017 年 4 月 27 日合并到 Go 代码仓库的主分支：

> Map is a concurrent map with amortized-constant-time loads, stores, and deletes.
> It is safe for multiple goroutines to call a Map's methods concurrently.

以下是对 sync.Map 的 Load/Store/Delete 等常用操作的图解:

![go-sync-map](https://raw.githubusercontent.com/RussellLuo/blog/master/blog/2017/go-sync-map_.png)


[1]: https://github.com/golang/go/blob/master/src/sync/map.go
