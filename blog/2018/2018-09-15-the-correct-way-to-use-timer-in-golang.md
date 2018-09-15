categories:
- 技术

tags:
- Go

title: 使用 Golang Timer 的正确方式
---

## 一、标准 Timer 的问题

以下讨论只针对由 [NewTimer][1] 创建的 Timer，因为这种 Timer 会使用 channel 来传递到期事件，而正确操作 channel 并非易事。

### Timer.Stop

按照 [Timer.Stop 文档][2] 的说法，每次调用 Stop 后需要判断返回值，如果返回 false（表示 Stop 失败，Timer 已经在 Stop 前到期）则需要排掉（drain）channel 中的事件：

```go
if !t.Stop() {
	<-t.C
}
```

但是如果之前程序已经从 channel 中接收过事件，那么上述 `<-t.C` 就会发生阻塞。可能的解决办法是借助 select 进行 **非阻塞** 排放（draining）：

```go
if !t.Stop() {
	select {
	case <-t.C: // try to drain the channel
	default:
	}
}
```

但是因为 channel 的发送和接收发生在不同的 goroutine，所以 [存在竞争条件][3]（race condition），最终可能导致 channel 中的事件未被排掉。

以下就是一种有问题的场景，按时间先后顺序发生：

- goroutine A：Go 运行时判断 Timer 已经到期，于是从最小堆中删除该 Timer
- goroutine B：应用程序执行 Timer.Stop，发现 Timer 已经到期，进而返回 false
- goroutine B：应用程序继续执行 `select...case <-t.C`，因为 channel 中并没有事件，所以会立即返回
- goroutine A：Go 运行时将到期事件发送到该 Timer 的 channel 中

### Timer.Reset

按照 [Timer.Reset 文档][4] 的说法，每要正确地 Reset Timer，需要首先正确地 Stop Timer。因此 Reset 的问题跟 Stop 基本相同。


## 二、正确方式

参考 Russ Cox 的回复（[这里][5] 和 [这里][6]），目前 Timer 唯一合理的使用方式是：

- 程序始终在同一个 goroutine 中进行 Timer 的 Stop、Reset 和 receive/drain channel 操作
- 程序需要维护一个状态变量，用于记录它是否已经从 channel 中接收过事件，进而作为 Stop 中 draining 操作的判断依据

如果每次使用 Timer 都要按照上述方案来处理，无疑是一件很费神的事。为此，我专门写了一个 Go 库 [goodtimer][7] 来解决标准 Timer 的问题。懒是一种美德 :-)


## 三、相关阅读

- [论golang Timer Reset方法使用的正确姿势][8]
- [time.Timer doc][9]


[1]: https://golang.org/pkg/time/#NewTimer
[2]: https://golang.org/pkg/time/#Timer.Stop
[3]: https://github.com/golang/go/issues/14383#issuecomment-185977844
[4]: https://golang.org/pkg/time/#Timer.Reset
[5]: https://github.com/golang/go/issues/11513#issuecomment-157062583
[6]: https://groups.google.com/d/msg/golang-dev/c9UUfASVPoU/tlbK2BpFEwAJ
[7]: https://github.com/RussellLuo/goodtimer
[8]: https://tonybai.com/2016/12/21/how-to-use-timer-reset-in-golang-correctly/
[9]: https://golang.org/pkg/time/#Timer
