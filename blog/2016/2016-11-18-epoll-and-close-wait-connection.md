categories:
- 技术

tags:
- Linux
- TCP/IP
- Redis
- Celery
- Python

title: epoll 与 CLOSE_WAIT 连接
---


本周在公司专项排查一个问题，最终问题被解决了，自己也感觉收获颇丰，特此总结一下。


## 一、问题背景

公司产品有一个数据导出功能，该功能一直以来饱受诟病，客户经常反馈说导出不了数据，于是就让客户支持人员帮忙手动导数据。客户体验差不说，客户支持人员也是苦不堪言。

内部实现上，该数据导出功能是通过 [Celery][1] 异步任务的方式来处理的。如果是因为导出数据量太大，导致任务超时被中途停掉，倒还可以理解。但大部分情况是，即使是很少量的数据，也无法导出。


## 二、排查分析

### 1. 查看 Celery 错误日志

当数据导出不了的时候，查看 Celery 错误日志，一般都是这样的：

```
[2016-11-15 14:17:09,478: ERROR/MainProcess] Exception during reset or similar
Traceback (most recent call last):
  File "/data/app/eggs/SQLAlchemy-1.0.15-py2.7-linux-x86_64.egg/sqlalchemy/pool.py", line 636, in _finalize_fairy
    fairy._reset(pool)
  File "/data/app/eggs/SQLAlchemy-1.0.15-py2.7-linux-x86_64.egg/sqlalchemy/pool.py", line 774, in _reset
    self._reset_agent.rollback()
  File "/data/app/eggs/SQLAlchemy-1.0.15-py2.7-linux-x86_64.egg/sqlalchemy/engine/base.py", line 1563, in rollback
    self._do_rollback()
  File "/data/app/eggs/SQLAlchemy-1.0.15-py2.7-linux-x86_64.egg/sqlalchemy/engine/base.py", line 1601, in _do_rollback
    self.connection._rollback_impl()
  File "/data/app/eggs/SQLAlchemy-1.0.15-py2.7-linux-x86_64.egg/sqlalchemy/engine/base.py", line 670, in _rollback_impl
    self._handle_dbapi_exception(e, None, None, None, None)
  File "/data/app/eggs/SQLAlchemy-1.0.15-py2.7-linux-x86_64.egg/sqlalchemy/engine/base.py", line 1341, in _handle_dbapi_exception
    exc_info
  File "/data/app/eggs/SQLAlchemy-1.0.15-py2.7-linux-x86_64.egg/sqlalchemy/util/compat.py", line 202, in raise_from_cause
    reraise(type(exception), exception, tb=exc_tb, cause=cause)
  File "/data/app/eggs/SQLAlchemy-1.0.15-py2.7-linux-x86_64.egg/sqlalchemy/engine/base.py", line 668, in _rollback_impl
    self.engine.dialect.do_rollback(self.connection)
  File "/data/app/eggs/SQLAlchemy-1.0.15-py2.7-linux-x86_64.egg/sqlalchemy/dialects/mysql/base.py", line 2542, in do_rollback
    dbapi_connection.rollback()
  File "/data/app/eggs/PyMySQL-0.7.6-py2.7.egg/pymysql/connections.py", line 772, in rollback
    self._execute_command(COMMAND.COM_QUERY, "ROLLBACK")
  File "/data/app/eggs/PyMySQL-0.7.6-py2.7.egg/pymysql/connections.py", line 1055, in _execute_command
    self._write_bytes(packet)
  File "/data/app/eggs/PyMySQL-0.7.6-py2.7.egg/pymysql/connections.py", line 1007, in _write_bytes
    raise err.OperationalError(2006, "MySQL server has gone away (%r)" % (e,))
OperationalError: (pymysql.err.OperationalError) (2006, "MySQL server has gone away (error(32, 'Broken pipe'))")
[2016-11-15 14:17:09,478: ERROR/MainProcess] Hard time limit (1800s) exceeded for app.jobs.download.download_data[63c0d55e-76a5-42ef-8d65-7bd432bc0877]
```

分析上述日志：

1. 似乎是 MySQL 报错了，但是 "MySQL server has gone away" 通常表明：client 端连接超时了，然后 server 端受不了了，所以强制 kill 掉了数据库连接（参考 [Error 2006: MySQL server has gone away][2]）

2. 导出任务执行超过了 1800 秒（30 分钟）！！然后被强制停掉了。。。

所有矛头都指向了一点：导出任务执行过慢。

### 2. 加日志作性能分析

导出任务执行过慢，最直接的判断就是：执行过程中有些步骤耗时过长。没啥好说的，加日志分别计算下每个步骤的执行时间呗。

这里特别提一点：对于使用了 SQLAlchemy 的代码，在对数据库查询作性能分析时，不必为每个查询语句都加日志，相关技巧请参考这篇官方文档 [How can I profile a SQLAlchemy powered application?][3]

加上日志后，（为了让修改后的代码生效）重启 celery-download，然后观察日志。然而。。。并没有发现异常，每个步骤的执行时间看起来都很合理。。。

### 3. 由 CPU 占用率引发的思考

事实证明，上面的排查手段并不奏效。重新整理思路后，决定不再过早地作判断，而是先搜集更多的有用信息。

很自然地，这一次注意到了 celery-download 的 CPU 占用率。一个 top 命令，发现 celery-download 的 CPU 占用率竟然接近 100%：

```
  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND         
28912 tester    20   0  477576 116500  14600 S  99.3  8.5  71:07.62 [celeryd: celery_download...]
```

惊讶地同时，似乎也能解释得通了：这么高的 CPU 占用率，一定是哪里出现了死循环，导致整个 celery-download 进程几乎瘫痪，进而无法正常工作。

除此以外，经过一些尝试和观察，还注意到一个现象：如果重启 celery-download，CPU 占用率会瞬间降下来，并且维持一段时间的正常值，然后过一会儿 CPU 占用率又会飙到很高。这也解释了在上一步 `加日志作性能分析` 的时候，为什么没有发现问题了：刚刚重启后的一段时间内，一切都是正常的。

但是为何会出现上述这些现象，对此我毫无头绪。请教 Google 大神后，意外地搜到了这个 [celery:issue#1845][4]。仔细看完以后，简直可以用 “醍醐灌顶” 来形容。

### 4. 顺藤摸瓜找元凶

按照 [celery:issue#1845][4] 中给出的思路，进行一一排查：

#### 1）strace 跟踪 celery-download 进程

```bash
$ strace -p 28912
...
epoll_wait(11, {{EPOLLIN|EPOLLOUT, {u32=30, u64=21474836725}}}, 130, 1) = 1
clock_gettime(CLOCK_MONOTONIC, {29956630, 774274775}) = 0
clock_gettime(CLOCK_MONOTONIC, {29956630, 774404883}) = 0
clock_gettime(CLOCK_MONOTONIC, {29956630, 774497018}) = 0
epoll_wait(11, {{EPOLLIN|EPOLLOUT, {u32=30, u64=21474836725}}}, 130, 1) = 1
clock_gettime(CLOCK_MONOTONIC, {29956630, 774643990}) = 0
clock_gettime(CLOCK_MONOTONIC, {29956630, 774770274}) = 0
clock_gettime(CLOCK_MONOTONIC, {29956630, 774861865}) = 0
...
```

可以看出，celery-download 进程一直在重复调用 epoll_wait 和 clock_gettime。参考 [Linux 手册][5]，我们知道 epoll_wait 返回 1 表示：有 1 个 fd（文件描述符）可用于读写。在这里，这个导致 epoll_wait 返回的 fd 就是 30。

再来看看这个 fd 有什么特别之处：

```bash
$ lsof -d 30|grep 28912
[celeryd: 28912 tester   30u  IPv4 1026883657      0t0      TCP xx-celery-app0:3759->ip-10-10-10-10.xx:6379 (CLOSE_WAIT)
```

很明显地，"xx-celery-app0:3759->ip-10-10-10-10.xx:6379" 是一个指向 Redis 的 socket 连接，而且这个连接处于 CLOSE_WAIT 状态！正是这个 CLOSE_WAIT 状态的 Redis 连接，导致 epoll_wait 总是会立即返回，从而让 celery-download 进程陷入了不断调用 epoll_wait 的死循环中！！

而对于 “celery-download 重启后，CPU 占用率会恢复正常” 的现象，可以这样解释：因为进程结束时，会关闭它用到的所有文件描述符（包括 CLOSE_WAIT 连接）；而对于新启动的进程，运行一段时间后，才会莫名其妙地产生 CLOSE_WAIT 连接 :-(

#### 2）分析 CLOSE_WAIT 连接

那么上述 Redis 连接为什么会处于 CLOSE_WAIT 状态呢？

我们知道，Redis 有个 timeout 参数，参考 [Redis 配置文件][6]：

```
# Close the connection after a client is idle for N seconds (0 to disable)
timeout 0
```

如果 timeout 不为 0，当 client 端连接的空闲时间超过了 timeout 秒，server 端会主动关闭该连接（更多说明参考 [Client timeouts][7]）。这种 “server 端主动关闭超时的 client 端连接” 的机制，与之前提到的 MySQL 的机制，其实是类似的。

当然这里的 `timeout 0` 是官方的参考值，"ip-10-10-10-10.xx" 对应的 Redis 实例（注意：与 [celery:issue#1845][4] 中描述的情况不同，这个 Redis 实例不是用作 Celery 的 broker 或 result backend，而是用作普通的缓存），实际的 timeout 配置为 1200（秒）。

于是，我们可以大胆猜测：fd 为 30 的那个 Redis 连接，因为空闲时间超过了 1200 秒，进而被 Redis 的 server 端主动关闭了（发送 FIN 报文），但是因为 client 端没有正确响应（回复 ACK 报文），导致该连接一直处于 CLOSE_WAIT 状态。

到这里，尚存两点疑惑：

1. 为什么 Redis 连接超过了 1200 秒，client 端还不主动关闭，非要等到 server 端关闭呢？
2. 为什么 server 端关闭后，client 端不正确响应呢？

#### 3）redis-py 的连接池机制

考虑到使用的 Redis 客户端库是 [redis-py][8] ，参考文档发现 redis-py 内部使用了 [连接池机制][9]，因此：一个连接用完后，不会被立即关闭，而会被释放到连接池中，等待下次取用。

查看 redis-py 的源码：

```python
# redis-2.10.5-py2.7.egg/redis/client.py

class StrictRedis(object):

    ...

    def execute_command(self, *args, **options):
        "Execute a command and return a parsed response"
        pool = self.connection_pool
        command_name = args[0]
        connection = pool.get_connection(command_name, **options)
        try:
            connection.send_command(*args)
            return self.parse_response(connection, command_name, **options)
        except (ConnectionError, TimeoutError) as e:
            connection.disconnect()
            if not connection.retry_on_timeout and isinstance(e, TimeoutError):
                raise
            connection.send_command(*args)
            return self.parse_response(connection, command_name, **options)
        finally:
            pool.release(connection)
```

结合 [redis-py:issue#306][10] 的说法，可以进一步得知：redis-py 对于关闭连接的处理是被动的，只会在下一次使用该连接的时候，检测该连接的可用性；如果使用连接时，遇到报错 ConnectionError 或 TimeoutError，才会调用 disconnect() 关闭该连接。 

综上所述，前面提到的两点疑惑，具体来讲，可以归结为一点：

- 为什么 fd 为 30 的 Redis 连接，在 redis-py 中没有被再次使用过，进而导致 server 端关闭该连接后，redis-py 不能正确关闭该连接？

遗憾地是，目前为止，这个问题还没能得到解答。（因为 Celery 的并发使用了 [gevent][11]，所以怀疑过是 `gevent 魔幻的 patch 处理` 跟 `redis-py 的连接池机制` 产生了化学反应，然而这种猜测暂时无法得到验证。）


## 三、归纳总结

前面长篇大论地说了很多，关于这个问题，总结起来其实只有三点：

### 1. 根本原因

celery-download 进程，运行一段时间后，产生了处于 CLOSE_WAIT 状态的连接，这种连接会让系统调用 epoll_wait 立即返回，从而让进程陷入不断调用 epoll_wait 的死循环中，进而导致该进程无法正常工作。

### 2. 解决办法

将 Redis 的 timeout 配置修改为 0（即禁止 server 端主动关闭超时的 client 端连接），从源头上避免 CLOSE_WAIT 连接的产生。

### 3. 遗留问题

celery 3.1.24 + gevent 1.2a1 + redis-py 2.10.5 的组合，会出现这种问题：redis-py 的连接池机制无法复用某些连接，进而导致这些连接处于失控状态。


## 四、一些技巧

### 1. 问题复现

对于上述 `epoll 与 CLOSE_WAIT 连接` 的问题，用这个 [简化示例][12] 可以完美复现。

server 端的代码如下：

```python
# server.py

import SocketServer
import time

HOST = '127.0.0.1'
PORT = 9999

class ReusableTCPServer(SocketServer.TCPServer):
    allow_reuse_address = True

class ServeAndCloseHandler(SocketServer.BaseRequestHandler):
    """
    Sends some data and then closes, to test epoll behavior against.
    """

    def handle(self):
        for i in range(3):
            self.request.sendall('data %d\n' % i)
            time.sleep(.5)

if __name__ == '__main__':
    server = ReusableTCPServer((HOST, PORT), ServeAndCloseHandler)
    server.serve_forever()
```

client 端的代码如下：

```python
# client.py

import select
import socket

HOST = '127.0.0.1'
PORT = 9999

def main():
    s = socket.socket()
    s.connect((HOST, PORT))
    epoll = select.epoll()
    epoll.register(s, select.POLLIN)
    while True:
        epoll.poll()
        data = s.recv(256)

if __name__ == '__main__':
    exit(main())
```

复现步骤提示：

1. 启动 server 端（python server.py）
2. 启动 client 端（python client.py）
3. 观察 client 端进程的 CPU 占用率（top）
4. 跟踪 client 端进程的执行情况（strace）
5. 观察 client 端进程的 CLOSE_WAIT 连接（lsof）

### 2. 关闭 CLOSE_WAIT 连接

我们知道，重启进程可以去掉 CLOSE_WAIT 连接。但是重启进程毕竟动作太大，有没有办法在进程运行的同时，去掉该进程中产生的 CLOSE_WAIT 连接呢？

答案是肯定的，借助 gdb 就可以做到。继续上面的例子，假设进程号为 28912、CLOSE_WAIT 连接的 fd 为 30，可以使用以下命令：

```bash
$ gdb -p 28912 -ex 'p close(30)' -ex 'set confirm off' -ex 'quit'
```


[1]: https://github.com/celery/celery
[2]: http://docs.peewee-orm.com/en/latest/peewee/database.html#error-2006-mysql-server-has-gone-away
[3]: http://docs.sqlalchemy.org/en/latest/faq/performance.html#how-can-i-profile-a-sqlalchemy-powered-application
[4]: https://github.com/celery/celery/issues/1845
[5]: http://www.man7.org/linux/man-pages/man2/epoll_wait.2.html
[6]: http://download.redis.io/redis-stable/redis.conf
[7]: http://redis.io/topics/clients#client-timeouts
[8]: https://github.com/andymccurdy/redis-py
[9]: https://github.com/andymccurdy/redis-py#connection-pools
[10]: https://github.com/andymccurdy/redis-py/issues/306
[11]: https://github.com/gevent/gevent
[12]: https://github.com/bpowers/epoll-close-wait-example
