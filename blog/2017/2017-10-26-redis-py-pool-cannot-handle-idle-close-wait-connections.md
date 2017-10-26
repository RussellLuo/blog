categories:
- 技术

tags:
- Linux
- TCP/IP
- Redis
- Python

title: redis-py 连接池不能处理空闲的 CLOSE_WAIT 连接
---


距离上次排查 [epoll 与 CLOSE_WAIT 连接][1] 的问题，已经过去了将近一年。最近在看 [《UNIX 网络编程》][2]，看到 “TCP 状态转换图” 中提到 CLOSE_WAIT 状态时，突然又想起来上次还有一个 [遗留问题][3]，于是决定再次尝试分析一下。


## 一、问题现象

上次的遗留问题，归纳起来就是：（由于 Redis 的 server 端主动关闭超时连接）在 client 端产生的 CLOSE_WAIT 连接，一直无法被 redis-py 连接池复用，进而无法被正常 close。


## 二、分析 redis-py 连接池机制

以当前最新的 redis-py 2.10.6 为例，[从连接池获取连接][4] 的源码：

```python
def get_connection(self, command_name, *keys, **options):
    "Get a connection from the pool"
    self._checkpid()
    try:
        connection = self._available_connections.pop()
    except IndexError:
        connection = self.make_connection()
    self._in_use_connections.add(connection)
    return connection
```

[释放连接到连接池][5] 的源码：

```python
def release(self, connection):
    "Releases the connection back to the pool"
    self._checkpid()
    if connection.pid != self.pid:
        return
    self._in_use_connections.remove(connection)
    self._available_connections.append(connection)
```

可以看出，redis-py 使用 `_available_connections` 来维护 “空闲可用的连接列表”，获取连接时 pop 出列表末尾的连接，释放连接时 append 连接到列表末尾。因此 “空闲可用的连接列表” 其实是个 **后进先出的栈**。

很显然，基于这种 “后进先出的栈” 的数据结构，redis-py 连接池对连接的获取和释放都发生在 “栈顶”。至此，原因就很明显了：**如果某段时间内由于突发流量产生了大量连接，一旦流量趋于平稳后，位于 “堆底” 的连接就会一直无法被复用，于是这些连接被 Redis 的 server 端超时关闭后，就会一直处于 CLOSE_WAIT 状态**。

关于这个问题，其实在 GitHub 上已经有一个类似的 issue：[ConnectionPool doesn't reap timeout'ed connections][6]，不过一直还未得到处理 :-(


## 三、解决方案

为了让 redis-py 连接池能够更均衡地复用各个连接，很容易想到的一个方案是：**将数据结构从 “后进先出的栈” 改成 “先进先出的队列”**。

通过修改 `get_connection` 的实现可以很容易做到这一点：

```python
# connection = self._available_connections.pop()
connection = self._available_connections.pop(0)  # 获取连接时，从队列首部 pop 出来
```

关于这个方案，其实在 GitHub 上也有一个 pull request：[Connection management improvements][7]，然而还是没有得到响应 :-( 不得不手动尴尬一下...


## 四、复现和验证

为了简化场景，便于问题的复现和方案的验证，这里有一段辅助代码：

```python
# example.py

import select

import redis


def main():
    import os; print('pid: %s' % os.getpid())

    r = redis.StrictRedis(host='localhost', port=6379, db=0)
    pool = r.connection_pool
    epoll = select.epoll()

    conns = (pool.get_connection(''), pool.get_connection(''))
    for conn in conns:
        conn.connect() 
        epoll.register(conn._sock, select.POLLIN)
    for conn in conns:
        pool.release(conn)

    command_args = ('SET', 'foo', 'bar')
    while True:
        conn = pool.get_connection('')
        conn.send_command(*command_args)
        epoll.poll()
        r.parse_response(conn, command_args[0])
        pool.release(conn)


if __name__ == '__main__':
    main()
```

操作步骤提示：

1. 设置 Redis 的 server 端的 timeout 参数（比如 10 秒）
2. 运行代码（python example.py）
3. 一段时间后，观察进程的 CPU 占用率（top）
4. 观察进程是否有 CLOSE_WAIT 连接（lsof -p PID）


[1]: http://russellluo.com/2016/11/epoll-and-close-wait-connection.html
[2]: https://book.douban.com/subject/1500149/
[3]: http://russellluo.com/2016/11/epoll-and-close-wait-connection.html#3-_遗留问题
[4]: https://github.com/andymccurdy/redis-py/blob/2.10.6/redis/connection.py#L959-L967
[5]: https://github.com/andymccurdy/redis-py/blob/2.10.6/redis/connection.py#L985-L991
[6]: https://github.com/andymccurdy/redis-py/issues/306
[7]: https://github.com/andymccurdy/redis-py/pull/886/commits/66e25f58c01b4a3221057e2a8ff973d2e8b3d8ab#diff-b3be2d0fa4e0cf1f1c403f2de43ecdb4R982
