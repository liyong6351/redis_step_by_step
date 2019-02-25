<a href="../README.md">README</a>

<!-- TOC -->

- [Redis Replication](#redis-replication)
  - [Safety of replication when master has persistence turned off](#safety-of-replication-when-master-has-persistence-turned-off)
  - [How Redis replication works](#how-redis-replication-works)
  - [Replication ID explained](#replication-id-explained)
  - [Diskless replication](#diskless-replication)
  - [Configuration](#configuration)
  - [Read-only slave](#read-only-slave)
  - [Setting a slave to authenticate to a master](#setting-a-slave-to-authenticate-to-a-master)
  - [Allow writes only with N attached replicas](#allow-writes-only-with-n-attached-replicas)
  - [How Redis replication deals with expires on keys](#how-redis-replication-deals-with-expires-on-keys)
  - [Configuring replication in Docker and NAT](#configuring-replication-in-docker-and-nat)
  - [The INFO and ROLE command](#the-info-and-role-command)
  - [Partial resynchronizations after restarts and failovers](#partial-resynchronizations-after-restarts-and-failovers)

<!-- /TOC -->

# Redis Replication

* Redis基本的复制是很简单的(排除Redis集群以及哨兵等高可用方案),他主要通过master-slave方式进行复制:slave节点能从master节点进行精确的复制。每当master-slave节点断开之后，slave节点都能自动重连并且不管master发生了什么都从master精确复制数据到slave。  
他们主要通过以下三种途径:
1. 当主实例和从属实例连接良好时，主设备通过向从设备发送命令流来保持从设备更新，以便复制对主设备端发生的数据集的影响，这些影响包括：客户端写入，key过期或者删除，更改主数据集的任何其他操作。
2. 当master和slave之间的网络出现问题时，slave会尝试重连并且继续部分更新同步:也就是说它将尝试获取断开期间执行的命令流。
3. 如果部分同步的命令失败的话，那么slave将进行一个全局的同步。这样会比较复杂，master首先会生成一个snapshot的命令集合同步到slave，然后载同步在同步snapshot集合期间数据发生的变化。

Redis默认采用异步同步机制，保证低延迟和高可用，这也是绝大多数redis使用用例的默认配置。Redis slave会异步给master反馈当前处理的命令集合，所以master不必每次都等待slave处理掉命令的信息。但是如果需要master事实的获取到slave处理情况的话，我们可以选择使用同步的逻辑。  
如果client同步数据的时候使用wait命令词，那么就是同步的同步数据。然而，wait只是能确保在一台redis slave中执行的命令列表，但是并不能确保强烈的数据一致性:在故障转移期间，数据在写的时候依然可能失败，这具体取决于Redis持久化的配置信息。然而，发生这样的情况是很少见的。  
可以通过集群或者哨兵模式提高可用性。但是本文并不讨论，本文只是讨论redis 复制的最基本特征   
以下是Redis最重要的几个特征:

* Redis在master-slave中使用异步模式同步大量的数据
* 一个master可以拥有多个slave
* slave可以和其他的slave建立连接。处理master-slave这种连接模式之外，还可以使用slave-slave模式，这样类似于级联模式。从Redis 4.0开始，sub-slave可以接收到master一样的复制命令。
* 复制在slave端在很大程度上也是非阻塞的。当slave在执行初始化同步的时候，查询可以在老版本的数据集上进行，这取决于你在redis.conf中配置的信息。同时，也可以配置如果在同步过程中那么返回错误信息。然而，当初始化同步结束之后，老的镜像会被删除掉，同时最新的数据将被加载到内存，在这个短暂的窗口期，redis将拒绝新的连接(如果数据集绝对大的话，这个同步过程可能也比较长)。从Redis 4.0开始，可以将同步的节点使用新的进程进行管理。然而，加载新的数据集依然会阻塞主线程。
* 复制机制可以增强可扩展性，比如扩展多个只读的从节点，或者为了安全或者可用性配置多个从节点。
* 复制机制可以减少master写磁盘所必须的消耗:最典型的应用是配置master根本不用写入到磁盘中。然后配置AOF模式，或者slave节点每次都同步到磁盘。然而这种情况下也需要注意，因为master重启的话将会载入一个空的数据集:如果这个时候从数据集刚好从master进行同步，那么slave将会变成空数据集。

## Safety of replication when master has persistence turned off

在生产环境中使用Redis，强烈建议在master和slave中启用持久性。如果无法做到这点，例如磁盘写入速度太慢导致大的延迟，应该配置实例以避免重启之后自动重新加载数据。  
为了更好的理解为什么master关闭持久化之后的危险性，检查以下故障模式，从主站及其所有从站擦除数据：  
1. 我们设置节点A为master，关闭其持久性，节点B和C为slave节点。
2. 节点A crash掉了，然而他有自动重启加载数据机制。因为持久化被关闭掉了，所以节点A将加载一个空的数据集。
3. 节点B和节点C从节点A同步，同样会同步成一个空的数据集。

当Redis 哨兵被配置之后，关闭持久化同样会导致问题。例如:如果master比哨兵启动还快，那么哨兵根本就没有探测到出现了错误，那么结果就是所有的节点都出现了空数据集的情况   
每次数据安全很重要，并且复制与没有持久性配置的主服务器一起使用时，应禁用实例的自动重启。

## How Redis replication works

每个redis master都有一个replication ID:他是一个记录已经被持久化的数据集的伪随机数。每个主服务器还会为生成的每个复制流字节递增一个偏移量，以便将其发送到从属服务器，以便使用修改数据集的新更改来更新从属服务器的状态。即使没有实际连接的slave，复制偏移也会增加，所以基本上每个给定的对：

```bash
Replication ID, offset
```

标识主数据集的确切版本。  
当Redis 的slave连接到redis master时，会向Redis master发送当前的Replication ID和偏移量。然而，如果master上的backlog 的缓存不足或者slave提供的replicatioID已经过期很久了，那么slave将处理一个全部的同步工作:slave将会获取到一个全部的数据集合。  
下面是一个全部的数据集合同步的详细情况:  
master启动一个后台处理程序生成一个RDB 文件，同时她开始缓冲从客户端接收到的新的写命令。当写入rdb的工作完成时，master将rdb文件传递给slave，slave保存这个rdb文件到磁盘文件并且加载的内存中。然后master将缓冲的写入命令同步给slave节点，这是作为命令流完成的，并且与Redis协议本身的格式相同。  
实际上，sync命令在Redis中已经不被使用了，为了向上兼容性才保留的，当前替换的方式是使用PSync  

## Replication ID explained

在上述的节点中我们说过，如果slave接收到相同的replicationId和offset，那么他们处理的数据时一模一样的。然而，了解Redis的复制ID和主ID和二级ID很有用。  
一个Replication ID代表一个历史命令中的数据集。每次Redis master从头开始启动，或者slave被推举成为master，一个新的replicationId被生成，slave连接到master时slave将会从master获取到这个replication Id。因此，具有两个相同replication Id的两个实例他们拥有相同的数据集，但是却不在同样的时间，这个时候就需要使用偏移量来区分谁拥有最新的数据集。  
比如：节点A和节点B拥有相同的replicationId，但是一个的偏移量是1000，一个是1024，这意味着第一个没有第二个多的应用命令，也就意味着A可以通过执行某些命令达到和B相同的状态。  
Redis拥有两个replicationId的原因为一个slave被推举成为master节点。在一个故障转移之后，被推介为master的slave节点将会保存新的replicationId和老的replicationId，当新的slave需要从新的master同步数据时，首先需要check老的replicationId进行处理，这将按预期工作，因为当slave被提升为master时，它将其辅助ID设置为其主ID，记住此ID切换发生时的偏移量，当同步完成之后会生成一个新的随机的replicationId ，因为新的历史记录将会开始。处理新的slave连接时，master将会匹配其replicationId和偏移量与replicationId和secondaryId。简而言之，这意味着在故障转移后，连接到新升级主站的从站不必执行完全同步。  

如果您想知道为什么提升为master的slave需要在故障转移后更改其复制ID：由于某些网络分区，旧主服务器可能仍然作为主服务器工作：保留相同的复制ID会违反以下事实： 任何两个随机实例的相同ID和相同偏移量意味着它们具有相同的数据集。

## Diskless replication

通常情况下，在每次全量复制的时都需要在磁盘上生成一个rdb文件，然后复制到slave节点进行同步。  
在磁盘速度很慢的情况下这个操作将非常耗时，Redis从2.8.18版本之后支持diskless 复制。在此设置中，子进程直接通过线路将RDB发送到从站，而不将磁盘用作中间存储。

## Configuration

配置一个slave节点很简单:只需要在slave中添加一下行即可:

```bash
slaveof 192.168.1.1 6379
```

master将开启向slave同步的子进程。  
还有一些参数用于调整主机在内存中采取的复制积压，以执行部分​​重新同步。请参阅发行版附带的redis.conf  
Diskless模式可使用repl-diskless-sync来设置，其延迟相关的配置为repl-diskless-sync-delay，更多的请参阅发行版附带的redis.conf  

## Read-only slave

从Redis2.6开始，slave在默认情况下都是read-only模式。这样的行为使用redis.conf中的read-only选项进行配置，同时可以使用 CONFIG SET 命令进行关闭或者打开操作。  
在Read-only模式下，slave将拒绝所有的write命令。这并不意味着slave不可信。然而通过使用rename-command指令禁用redis.conf中的命令，可以提高只读实例的安全性

## Setting a slave to authenticate to a master

如果一个master有密码，那么可以在运行中实例中使用redis-cli执行

```bash
config set masterauth <password>
```

如果要永久性生效的话，需要在redis.conf中配置如下:

```bash
masterauth <password>
```

## Allow writes only with N attached replicas

从Redis 2.8开始，只有当至少N个从站连接到主站时，才可以将Redis主站配置为接受写入查询。  
然而，因为redis是异步的，master无法精确知道slave是否真的接收到redis的写操作，所以存在一些数据遗漏的情况。  
以下是工作原理:  
* slave每秒钟ping master一次，表示已经处理
* master会记录每个slave发送ping的时间
* 用户可以配置滞后不大于最大秒数的最小数量的从站。

如果配置了至少N个slave，M秒的最大延迟，master接受写入。  
您可以将其视为尽力而为的数据安全机制，其中对于给定的写入不确保一致性，但是至少数据丢失的时间窗口被限制为给定的秒数。 通常，绑定数据丢失优于未绑定数据丢失。  
如果不满足条件，主机将以错误回复，并且不接受写入。  
配置项为:

```bash
min-slaves-to-write <number of slaves>
min-slaves-max-lag <number of seconds>
```

## How Redis replication deals with expires on keys

Redis的key允许存在一个过期时间，这样的特性主要取决于计算过期时间的能力。  
Redis主要使用以下三种策略处理过期key的问题:

1. slave不会过期keys，他们等待master过期这些key。当master过期这些key的时候他们会同步一个del指令给slave
2. slave发现有过期的数据，那么会返回一个数据不存在
3. 在Lua脚本执行期间，不执行任何密钥到期。 当Lua脚本运行时，从概念上讲，主服务器中的时间被冻结，因此在脚本运行的所有时间内，给定的密钥将存在或不存在。 这可以防止密钥在脚本中间到期，并且需要以确保在数据集中具有相同效果的方式将相同的脚本发送到从属服务器

一旦slave被推举为master，那么它会执行key的过期操作，这和上一次的master无关

## Configuring replication in Docker and NAT

## The INFO and ROLE command

info指令和role指令  

## Partial resynchronizations after restarts and failovers

从Redis 4.0开始，一旦一个slave被选举为master，它将提供之前的master对应的replication操作。为此，从站会记住旧的复制ID和其以前的主站的偏移量，因此即使它们要求旧的复制ID，也可以将部分积压提供给连接的从站。