categories:
- 技术

tags:
- Redis

title: Redis 4.0 非阻塞删除
---

## 一、要解决的问题

一直以来，Redis 都是单线程的（参考 [FAQ][1]）。这种模型使得 Redis 简单、高效，但缺点也很明显：如果执行一个比较耗时的命令，那么在该命令执行期间，整个 Redis 服务都将被阻塞（无法并发地执行其他命令）。

大部分 Redis 命令的执行速度都很快，所以不是问题；但也有一些命令，比如 [ZUNIONSTORE][2]、[LRANGE][3]、[SINTER][4]，以及臭名昭著的 [KEYS][5]，根据处理数据集大小的不同，可能会阻塞 Redis 数秒或几分钟。

以 [DEL][6] 命令为例，当被删除的 key 是 list、set、sorted set 或 hash 类型时，时间复杂度为 O(M)，其中 M 是 key 中包含的元素的个数。


## 二、非阻塞删除

Redis 的作者 Salvatore Sanfilippo 自己也意识到了上述问题，并提出了对应的解决方案：非阻塞删除（参考 [Lazy Redis is better Redis][7]）。简而言之，「非阻塞删除」就是将删除操作放到另外一个线程（而非 Redis 主线程）去处理。

最终「非阻塞删除」在 Redis 4.0 中得以实现（参考 [Redis 4.0 release notes][8]），从此 Redis 开启了 “多线程” 时代。

新增实现的「非阻塞删除」包括以下命令：

| 命令 | （原来的）阻塞版本 |
| --- | --- |
| [UNLINK][9] | [DEL][6] |
| [FLUSHALL][10] ASYNC | [FLUSHALL][10] |
| [FLUSHDB][11] ASYNC | [FLUSHDB][11] |


## 三、DEL vs UNLINK

### 1. 源码实现

参考 [Redis 源码][12] 可以发现，[DEL][6] 和 [UNLINK][9] 分别对应不同的处理函数：

| 命令 | 处理函数 |
| --- | --- |
| [DEL][6] | [dbSyncDelete][13]
| [UNLINK][9] | [dbAsyncDelete][14]

具体的实现细节请自行研读源码。

### 2. 耗时对比

下面我们来实际对比一下 [DEL][6] 和 [UNLINK][9] 的耗时差异。

#### 开启 Slow log

设置 Slow log 记录每条命令的耗时（参考 [SLOWLOG][15]）：

```
127.0.0.1:6379> CONFIG SET slowlog-log-slower-than 0
OK
127.0.0.1:6379> CONFIG GET slowlog-log-slower-than
1) "slowlog-log-slower-than"
2) "0"
```

#### 创建两个大 hash

准备一个 lua 脚本：

```lua
local bulk = 1000
local fvs = {}
local j
for i = 1, ARGV[1] do
  j = i % bulk
  if j == 0 then
    fvs[2 * bulk - 1] = "field" .. i
    fvs[2 * bulk] = "value" .. i
    redis.call("HMSET", KEYS[1], unpack(fvs))
    fvs = {}
  else
    fvs[2 * j - 1] = "field" .. i
    fvs[2 * j] = "value" .. i
  end
end
if #fvs > 0 then
  redis.call("HMSET", KEYS[1], unpack(fvs))
end
return "OK"
```

将上述脚本保存为 `huge_hmset.lua`，然后创建两个大 hash，分别为 `hash1` 和 `hash2`，它们各自拥有 100 万个 field：

```bash
$ redis-cli --eval huge_hmset.lua hash1 , 1000000
"OK"
$ redis-cli --eval huge_hmset.lua hash2 , 1000000
"OK"
```

上述操作会在 Slow log 中产生大量 [HMSET][16] 命令，这里先清除掉：

```
127.0.0.1:6379> slowlog reset
OK
```

#### DEL hash1

```
127.0.0.1:6379> DEL hash1
(integer) 1
(0.63s)
```

#### UNLINK hash2

```
127.0.0.1:6379> UNLINK hash2
(integer) 1
```

#### 查看 Slow log

```
127.0.0.1:6379> SLOWLOG GET 2
1) 1) (integer) 5089
   2) (integer) 1534653951
   3) (integer) 17
   4) 1) "UNLINK"
      2) "hash2"
   5) "127.0.0.1:56560"
   6) ""
2) 1) (integer) 5088
   2) (integer) 1534653948
   3) (integer) 630305
   4) 1) "DEL"
      2) "hash1"
   5) "127.0.0.1:56560"
   6) ""
```

耗时对比结果：

| 命令 | 耗时 |
| --- | --- |
| [DEL][6] hash1 | 630305 us |
| [UNLINK][9] hash2 | 17 us |

值得注意的是：[UNLINK][9] 执行如此之快，并非使用了什么快速算法，而是因为它将真正的删除操作异步化了。


## 四、更多资料

- [Never Stop Serving: Making Redis Concurrent With Modules][17]


[1]: https://redis.io/topics/faq#redis-is-single-threaded-how-can-i-exploit-multiple-cpu--cores
[2]: https://redis.io/commands/zunionstore
[3]: https://redis.io/commands/lrange
[4]: https://redis.io/commands/sinter
[5]: https://redis.io/commands/keys
[6]: https://redis.io/commands/del
[7]: http://antirez.com/news/93
[8]: https://raw.githubusercontent.com/antirez/redis/4.0/00-RELEASENOTES
[9]: https://redis.io/commands/unlink
[10]: https://redis.io/commands/flushall
[11]: https://redis.io/commands/flushdb
[12]: https://github.com/antirez/redis/blob/c7613cf34ed8f49681607d247cdf12a7e80cec94/src/db.c#L450-L475
[13]: https://github.com/antirez/redis/blob/c7613cf34ed8f49681607d247cdf12a7e80cec94/src/db.c#L261-L271
[14]: https://github.com/antirez/redis/blob/b0392e75ec37ffae0cb2277723168d8179861bef/src/lazyfree.c#L54-L91
[15]: https://redis.io/commands/slowlog
[16]: https://redis.io/commands/hmset
[17]: https://redislabs.com/blog/making-redis-concurrent-with-modules/
