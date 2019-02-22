# Redis Admin

[toc]

## redis-cli

-a 使用密码
-n 指定不同的db  
-h 指定hostname  
-p 指定port
-r 重复执行
-i 延迟时间(秒)  
--csv csv的输出格式

```EL
redis-cli是redis通过终端的命令窗口
它主要有两种命令模式:一种是进入交互式的命令窗口模式；另一种是带参可执行程序获取执行结果
在交互命令窗口模式中，redis-cli具备基本的行编辑功能。
在特殊模式下，redis-cli可以执行更复杂的任务，例如模拟slave并打印master的复制流，检查延迟，显示统计信息甚至延迟样本和频率的频谱图等。
本文将从最简单的应用到最复杂的应用讲解redis-cli的使用情况。
如果你使用redis比较深入的话，那么redis-cli将会很有用。一旦你理解了redis-cli的使用方式之后，将会帮助你进一步理解redis本身
```

### 命令航使用模式

#### 直接使用参数模式

```bash
\$ redis-cli incr mycounter
(integer)1
```

* 用于增加key=mycounter的计数器的值。同时可以将输出参数重定向

```bash
 \$ redis-cli incr mycount > /tmp/output.txt
 $ cat /tmp/output.txt
 2
```

* 可以使用--raw强制使用原始输出

```bash
\$ redis-cli --raw incr mycount
3
```

* 类似地，您可以使用--no-raw在写入文件或管道中强制输出人类可读输出。

##### Host, port, password and database

* 可以使用redis-cli -h redis15.localnet.org -p 6390 ping通过连接到不同的host或者port进行交互测试  
-a 使用密码
-n 指定不同的db  
-h 指定hostname  
-p 指定port

```bash
\$ redis-cli -h redis15.localnet.org -p 6390 ping
PONG
```

* 以上的测试用例也可以使用-u \<uri> 来操作:

```bash
\$ redis-cli -u redis://p%40ssw0rd@redis-16379.hosted.com:16379/0 ping
```

##### Getting input from other programs

* 有两种方式可以完成输入参数的作用，第一种是 command < argument模式，另一种是 argment| command 模式

```bash
$ resid-cli -x < /etc/services
OK
$ redis-cli getRange foo 0 50 
"# /etc/services:\n# $Id: services,v 1.55 2013/04/14 "
```

```bash
$ cat /tmp/commands.txt
set foo 100
incr foo
append foo xxx
get foo
$ cat /tmp/commands.txt | redis-cli
OK
(integer) 101
(integer) 6
"101xxx"
```

#### Continuously run the same command(重复执行命令)

```bash
$ redis-cli -r 5 incr foo
(integer) 1
(integer) 2
(integer) 3
(integer) 4
(integer) 5
```  

```bash
$ redis-cli -r -1 -i 1 INFO | grep rss_human
used_memory_rss_human:1.38M
used_memory_rss_human:1.38M
used_memory_rss_human:1.38M
... a new line will be printed each second ..
```

#### Mass insertion of data using redis-cli(大量插入数据)

使用redis-cli的大量插入包含在一个单独的页面中，因为它本身就是一个有价值的主题。 请参阅我们的质量插入指南。

#### CSV output

```bash
$ redis-cli lpush mylist a b c d
(integer) 4
$ redis-cli --csv lrange mylist 0 -1
"d","c","b","a"
```

#### Running Lua scripts (使用Lua脚本)  

```bash
$ cat /tmp/script.lua
return redis.call('set',KEYS[1],ARGV[1])
$ redis-cli --eval /tmp/script.lua foo , bar
OK
```

### Interactive mode(交互模式)

#### 连接进入到redis server

```bash
$ redis-cli
127.0.0.1:6379> ping
PONG
```

#### redis相关操作

```bash
127.0.0.1:6379> select 2
OK
127.0.0.1:6379[2]> dbsize
(integer) 1
127.0.0.1:6379[2]> select 0
OK
127.0.0.1:6379> dbsize
(integer) 503
```

#### Handling connections and reconnections(处理或重连接)

* 连接到其他的实例

```bash
127.0.0.1:6379> connect metal 6379
metal:6379> ping
PONG
```

* 当新的连接是不可达的，那么redis-cli会首先断掉当前的连接，然后不停尝试新的连接:

```bash
127.0.0.1:6379> connect 127.0.0.1 9999
Could not connect to Redis at 127.0.0.1:9999: Connection refused
not connected> ping
Could not connect to Redis at 127.0.0.1:9999: Connection refused
not connected> ping
Could not connect to Redis at 127.0.0.1:9999: Connection refused
```

* 通常在检测到断开连接后，CLI始终尝试透明地重新连接：如果尝试失败，则显示错误并进入断开连接状态。以下是断开连接和重新连接的示例
  
```bash
127.0.0.1:6379> debug restart
Could not connect to Redis at 127.0.0.1:6379: Connection refused
not connected> ping
PONG
127.0.0.1:6379> (now we are connected again)
```

#### Editing, history and completion

* redis-cli可以通过<tab>智能提示可用的命令

```bash
127.0.0.1:6379> Z<TAB>
127.0.0.1:6379> ZADD<TAB>
127.0.0.1:6379> ZCARD<TAB>
```

* redis-cli可以使用向上或者向下找到之前执行的命令
* 可以使用help指令获取到指令的使用方式
* 可以使用clear清理当前的屏幕信息

### Special modes of operation

在特殊模式下，redis-cli可以执行如下操作

* 持续监控redis的统计信息
* 扫描redis中的超大的key
* 按照某种模式扫描redis所有的key
* 按照发布订阅模式订阅某个channel
* 监控redis命令的使用情况
* 以不同的方式监控redis的延迟情况
* 检查本地计算机的调度延迟
* 在本地从远程redis服务器传输RDB备份
* 模拟slave获取slave接收到的信息
* 模拟LRU算法获取到key被命中的情况
* Lua debugger的客户端

---

* Continuous stats mode
这可能是redis-cli鲜为人知的功能之一，也是一个非常有用的实时监控Redis实例。 要启用此模式，请使用--stat选项。 在此模式下，CLI的行为非常明确：

```bash
redis-cli -i --stat
```

* Scanning for big keys

```bash
redis-cli --bigkeys
```

* 该程序使用SCAN命令，因此可以在不影响操作的情况下对繁忙的服务器执行，但是可以使用-i选项来为每个请求的100个密钥限制指定的秒数的扫描过程。 例如，-i 0.1会大大减慢程序执行速度，但也会将服务器上的负载减少到很小的数量。

* Getting a list of keys

```bash
redis-cli --scan --pattern '*-11*'
```

* get total of the key
  
```bash
redis-cli --scan --pattern 'user:*' | wc -l
```

* pub/sub mode
CLI只能使用PUBLISH命令在Redis Pub / Sub通道中发布消息。 这是预期的，因为PUBLISH命令与任何其他命令非常相似。 订阅频道以接收消息是不同的 - 在这种情况下，我们需要阻止并等待消息，因此这在redis-cli中实现为特殊模式。 与其他特殊模式不同，此模式不是通过使用特殊选项启用的，而是通过使用SUBSCRIBE或PSUBSCRIBE命令在交互模式或非交互模式下启用：

```bash
$ redis-cli psubscribe '*'
Reading messages... (press Ctrl-C to quit)
1) "psubscribe"
2) "*"
3) (integer) 1
```

* Monitoring commands executed in Redis
用于监听命令的执行情况

```bash
$ redis-cli monitor
OK
1460100081.165665 [0 127.0.0.1:51706] "set" "foo" "bar"
1460100083.053365 [0 127.0.0.1:51707] "get" "foo"
```

* Monitoring the latency of Redis instances
Redis有多个检测延迟的方法，使用--latency是最简单的一个，

```el
$ redis-cli --latency
min: 0, max: 1, avg: 0.19 (427 samples)
```

统计持续时间

```el
redis-cli --latency-history
```

生成ASCII类型的彩色图表

```el
redis-cli --latency-dist
```

检测固有延迟

```el
$ redis-cli --intrinsic-latency 5
Max latency so far: 1 microseconds.
Max latency so far: 10 microseconds.
Max latency so far: 13 microseconds.
Max latency so far: 37 microseconds.
Max latency so far: 42 microseconds.
Max latency so far: 59 microseconds.
Max latency so far: 203 microseconds.
Max latency so far: 204 microseconds.
Max latency so far: 205 microseconds.
Max latency so far: 245 microseconds.

137075272 total runs (avg latency: 0.0365 microseconds / 36.48 nanoseconds per run).
Worst run took 6717x longer than the average latency.
```

注意点:上述命令只能执行在redis的server中  
在上面的例子中，我的系统不能比245微秒的最坏情况延迟做得更好，所以我可以预期某些查询会在不到0.5毫秒的时间内运行。

* Remote backups of RDB files
在Redis复制的第一次同步期间，主服务器和从服务器以RDB文件的形式交换整个数据集。redis-cli利用此功能来提供远程备份工具。对应的命令为 redis-cli --rdb dest-filename

```el
$ redis-cli --rdb /tmp/dump.rdb
SYNC sent to master, writing 265 bytes to '/tmp/dump.rdb'
Transfer finished with success.
```

这是一种简单但有效的方法，可确保您具有Redis实例的灾难恢复RDB备份。 但是，在脚本或cron作业中使用此选项时，请确保检查命令的返回值。

* Slave mode
redis-cli可以模拟slave节点，以此获取到master同步给slave的信息。 redis-cli --slave

```el
$ redis-cli --slave
SYNC with master, discarding 265 bytes of bulk transfer...
SYNC done. Logging commands from master.
"PING"
"SELECT","0"
"set","hello","nihao"
"PING"
"SELECT","2"
"set","name","liyong2"
```

* Performing an LRU simulation
注意点:请务必配置maxMemory属性，因为如果不配置此属性，那么过期策略是没有任何意义的。同时请不要在生产环境这么做。

```el
$ redis-cli --lru-test 5000000
5000 Gets/sec | Hits: 4866 (97.32%) | Misses: 134 (2.68%)
5750 Gets/sec | Hits: 5595 (97.30%) | Misses: 155 (2.70%)
4750 Gets/sec | Hits: 4615 (97.16%) | Misses: 135 (2.84%)
5000 Gets/sec | Hits: 4853 (97.06%) | Misses: 147 (2.94%)
5250 Gets/sec | Hits: 5117 (97.47%) | Misses: 133 (2.53%)
5250 Gets/sec | Hits: 5112 (97.37%) | Misses: 138 (2.63%)
```