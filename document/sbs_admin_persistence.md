<a href="../README.md">README</a>

<!-- TOC -->

- [Redis Persistence](#redis-persistence)
  - [RDB advantages](#rdb-advantages)
  - [RDB disadvantages](#rdb-disadvantages)
  - [AOF advantages](#aof-advantages)
  - [AOF disadvantages](#aof-disadvantages)
  - [Ok, so what should I use](#ok-so-what-should-i-use)
    - [Snapshotting](#snapshotting)
      - [How it works](#how-it-works)
    - [Append-only file](#append-only-file)
      - [Log rewriting](#log-rewriting)
      - [How durable is the append only file](#how-durable-is-the-append-only-file)
      - [What should I do if my AOF gets truncated?](#what-should-i-do-if-my-aof-gets-truncated)
      - [What should I do if my AOF gets corrupted](#what-should-i-do-if-my-aof-gets-corrupted)
        - [How it works](#how-it-works-1)
      - [How I can switch to AOF, if I'm currently using dump.rdb snapshots](#how-i-can-switch-to-aof-if-im-currently-using-dumprdb-snapshots)
    - [Interactions between AOF and RDB persistence](#interactions-between-aof-and-rdb-persistence)
    - [Backing up Redis data](#backing-up-redis-data)
    - [Disaster recovery](#disaster-recovery)

<!-- /TOC -->

本文探讨Redis-Persistence相关的技术，这对所有使用redis的用户都是非常有用的。对于存储持久性的讨论你可能还需要阅读Redis persistence demystified.

# Redis Persistence

Redis提供了不同的持久性选项:

* RDB:指定时间间隔建立数据快照的方式
* AOF:记录每个Redis的写入操作，这将在服务器重启时重建原数据集。使用Redis协议本身相同的格式以追加方式记录命令。当Redis太大时，Redis能够重写日志背景。
* 如果可以的话，可以不使用任何持久化操作。
* 可以在实例中同时使用RDB和AOF方式。注意点:在恢复数据时需要使用AOF日志恢复，因为他保存的最完美

最重要的是要理解RDB和AOF的区别

## RDB advantages

* RDB是Redis数据的一个非常紧凑的单文件时间点表示，RDB文件非常适合于备份。例如，您可能希望在最近24小时内每小时归档您的RDB文件，并且每天保存RDB快照30天。这使您可以在发生灾难时轻松恢复不同版本的数据集。
* RDB非常适合灾难恢复，可以将单个压缩文件传输到远端数据中心或Amazon S上
* RDB最大限度地提高了Redis的性能，因为Redis父进程为了坚持不懈而需要做的唯一工作就是分配一个将完成所有其余进程执行。父实例永远不会执行磁盘I / O或类似操作
* 和AOF相比较，RDB可以更快的恢复实例

## RDB disadvantages

* 如果您的数据库在乎少量的数据丢失，那么RDB不是一个好的选择。AOF可以做到实时备份
* 如果数据集很大，Fork（）可能会非常耗时，如果数据集非常大且CPU性能不佳，可能会导致Redis停止服务客户端几毫秒甚至一秒钟。 AOF也需要fork（），但你可以调整你想要重写日志的频率而不需要对耐久性进行任何权衡。

## AOF advantages

* 使用AOF Redis更持久：您可以使用不同的fsync策略：根本不使用fsync，每秒执行fsync，每次查询都使用fsync。 使用fsync的默认策略，每秒写入性能仍然很好（使用后台线程执行fsync，并且当没有fsync正在进行时，主线程将努力执行写入。）但是您只能丢失一秒的写入。
* AOF日志是仅附加日志，因此如果停电，则没有搜索，也没有损坏问题。 即使由于某种原因（磁盘已满或其他原因）日志以半写命令结束，redis-check-aof工具也能够轻松修复它。
* 当Redis太大时，Redis能够在后台自动重写AOF。 重写是完全安全的，因为当Redis继续附加到旧文件时，使用创建当前数据集所需的最小操作集生成一个全新的文件，并且一旦第二个文件准备就绪，Redis会切换两个并开始附加到 新的那一个。
* AOF以易于理解和解析的格式一个接一个地包含所有操作的日志。 您甚至可以轻松导出AOF文件。 例如，即使您使用FLUSHALL命令刷新了所有错误，如果在此期间未执行重写日志，您仍然可以保存数据集，只需停止服务器，删除最新命令，然后重新启动Redis。

## AOF disadvantages

* 在相同的Redis实例中，AOF通常比RDB更大
* 根据确切的fsync策略，AOF可能比RDB慢。 一般来说，fsync设置为每秒性能仍然非常高，并且在fsync禁用的情况下，即使在高负载下也应该与RDB一样快。 即使在写入负载很大的情况下，RDB仍能够提供有关最大延迟的更多保证。
* 在过去，我们遇到了特定命令中的罕见错误（例如，有一个涉及阻塞命令，如BRPOPLPUSH）导致生成的AOF在重新加载时不会重现完全相同的数据集。这个错误很少见，我们在测试套件中进行测试，自动创建随机复杂数据集并重新加载它们以检查一切是否正常，但RDB持久性几乎不可能出现这种错误。为了更清楚地说明这一点：Redis AOF逐步更新现有状态，如MySQL或MongoDB，而RDB快照一次又一次地创建所有内容，这在概念上更加健壮。但是 -  1）应该注意的是，每次通过Redis重写AOF时，都会从数据集中包含的实际数据开始从头开始重新创建，与始终附加的AOF文件（或一个重写的读取文件）相比，可以更好地抵抗错误旧的AOF而不是读取内存中的数据）。 2）我们从未向用户提供过关于在现实世界中检测到的AOF损坏的单一报告。

## Ok, so what should I use
一般的迹象是，如果您希望一定程度的数据安全性与PostgreSQL为您提供的数据安全性相当，则应使用两种持久性方法。  
如果你非常关心你的数据，但是在发生灾难的情况下仍然会有几分钟的数据丢失，你可以单独使用RDB。  
有许多用户单独使用AOF，但我们不鼓励它，因为不时有RDB快照是进行数据库备份，更快重启以及AOF引擎中出现错误的好主意。  
注意：由于所有这些原因，我们可能最终将AOF和RDB统一为未来的单一持久性模型（长期计划）。  
以下部分将说明有关两种持久性模型的更多详细信息。

### Snapshotting

默认情况下，Redis将数据集的快照保存在磁盘上，名为dump.rdb的二进制文件中。 如果数据集中至少有M个更改，则可以将Redis配置为每N秒保存一次数据集，或者您可以手动调用SAVE或BGSAVE命令。  
例如，如果至少更改了1000个密钥，则此配置将使Redis每60秒自动将数据集转储到磁盘：

```bash
save 60 1000
```

此策略称为快照。

#### How it works

每当Redis需要将数据集转储到磁盘时，就会发生以下情况：

* Redis forks,这时候系统拥有一个主进程和一个子进程
* 子进程开始将数据同步写入到RDB临时文件中
* 当子进程结束时，会替代老的rdb文件

此方法允许Redis受益于写时复制语义。

### Append-only file

快照不是很耐用。 如果运行Redis的计算机停止运行，电源线发生故障，或者您意外杀死了实例，则Redis上写入的最新数据将丢失。  
虽然这对某些应用程序来说可能不是什么大问题，但是有完整持久性的用例，在这些情况下，Redis不是一个可行的选择。  
仅附加文件是Redis的另一种完全持久的策略。 它在1.1版中可用。  
您可以在配置文件中打开AOF：

```bash
appendonly yes
```

从现在开始，每次Redis收到更改数据集（例如SET）的命令时，它都会将其附加到AOF。  
重新启动Redis时，它将重新播放AOF以重建状态。


#### Log rewriting
你可以猜到，随着执行写操作，AOF变得越来越大。 例如，如果您将计数器递增100次，则最终会在数据集中包含一个包含最终值的单个键，但在AOF中会有100个条目。 重建当前状态不需要其中99个条目。  
所以Redis支持一个有趣的功能：它能够在后台重建AOF而不会中断对客户端的服务。 每当您发出BGREWRITEAOF时，Redis都会编写在内存中重建当前数据集所需的最短命令序列。 如果您在Redis 2.2中使用AOF，则需要不时运行BGREWRITEAOF。 Redis 2.4能够自动触发日志重写（有关详细信息，请参阅2.4示例配置文件）。

#### How durable is the append only file

你可以配置Redis fync数据到磁盘的频率。这里有三种配置方:
* appendfsync always:每次有新的命令写入时同步到磁盘
* appendfsync everysec:每秒同步磁盘，可能会导致一秒的数据丢失
* appendfsync no:最快的方法。不同步，数据仅仅存在于内存当中，通常Linux会在30秒同步一次数据  
  
建议的（和默认）策略是每秒fsync。 它非常快速且非常安全。 总是策略在实践中非常慢，但它支持组提交，因此如果有多个并行写入，Redis将尝试执行单个fsync操作。

#### What should I do if my AOF gets truncated?

服务器可能在写入AOF文件时崩溃，或者存储AOF文件的卷存储已满。当发生这种情况时，AOF仍然包含表示数据集的给定时间点版本的一致数据（使用默认的AOF fsync策略可能会持续一秒钟），但AOF中的最后一个命令可能会被截断。Redis的最新主要版本无论如何都能够加载AOF，只是丢弃文件中最后一个非格式化的命令。 在这种情况下，服务器将发出如下所示的日志  

```bash
* Reading RDB preamble from AOF file...
* Reading the remaining AOF tail...
# !!! Warning: short read while loading the AOF file !!!
# !!! Truncating the AOF at offset 439 !!!
# AOF loaded anyway because aof-load-truncated is enabled
```

如果需要，您可以更改默认配置以强制Redis在此类情况下停止，但无论文件中的最后一个命令格式不正确，默认配置都要继续，以便在重新启动后保证可用性。  
旧版本的Redis可能无法恢复，可能需要执行以下步骤:  
* 制作AOF文件的备份副本。
* 使用Redis附带的redis-check-aof工具修复原始文件：

```bash
$ redis-check-aof --fix
```

* 使用diff -u来检查两个文件之间的区别。
* 使用fixed之后的文件进行重启

#### What should I do if my AOF gets corrupted

如果AOF文件不仅被截断，而是在中间被无效的字节序列破坏，那么事情就更复杂了。 Redis会在启动的时候报错并将中止：

```bash
* Reading the remaining AOF tail...
# Bad file format reading the append only file: make a backup of your AOF file, then use ./redis-check-aof --fix <filename>
```

最好的办法是运行redis-check-aof实用程序，最初没有--fix选项，然后理解了错误，跳转到出错的行数，看看是否能采用手工方式恢复数据：AOF采用和redis相同的协议，所以比较简单而且容易发现错误。否则可以让实用程序为我们修复文件，但是在这种情况下，从无效部分到文件末尾的所有AOF部分都可能被丢弃，如果发生损坏，会导致大量数据丢失 在文件的初始部分。

##### How it works

日志重写使用已用于快照的相同的写时复制技巧。 这是它的工作原理：  
* Redis forks，当前Redis有主线程了子线程。
* 子线程将数据写入到AOP的临时文件中
* 主线程会在内存缓冲区中累积所有新更改（但同时它会将旧更改写入旧的仅附加文件中，因此如果重写失败，我们就是安全的）。
* 当子进程重写文件时，父进程会获得一个信号，并在子进程生成的文件末尾附加内存缓冲区。
* 现在，Redis以原子方式将旧文件重命名为新文件，并开始将新数据附加到新文件中。

#### How I can switch to AOF, if I'm currently using dump.rdb snapshots

这讨论的是redis 2.0和redis 2.2的问题，略过

### Interactions between AOF and RDB persistence

Redis> = 2.4确保在RDB快照操作正在进行时避免触发AOF重写，或者在AOF重写正在进行时允许BGSAVE。 这可以防止两个Redis后台进程同时执行大量磁盘I / O.

当快照正在进行且用户使用BGREWRITEAOF明确请求日志重写操作时，服务器将回复一个OK状态代码，告诉用户操作已安排，并且一旦快照完成，重写将开始。

在启用AOF和RDB持久性并且Redis重新启动的情况下，AOF文件将用于重建原始数据集，因为它保证是最完整的。

### Backing up Redis data

在开始本节之前，请务必确保一下条件：确保备份您的数据库。 磁盘中断，云中的实例消失，等等：没有备份意味着数据消失在/ dev / null中的巨大风险。  
Redis非常适合数据备份，因为您可以在数据库运行时复制RDB文件：RDB一旦生成就永远不会被修改，并且在生成它时会使用临时名称并仅使用rename（2）以原子方式重命名为其最终目标 新快照完成时。  
这意味着在服务器运行时复制RDB文件是完全安全的。 这就是我们的建议：
* 在服务器中创建一个cron作业，在一个目录中创建RDB文件的每小时快照，在另一个目录中创建每日快照。
* 每次运行cron脚本时，请确保调用find命令以确保删除过旧的快照：例如，您可以获取最近48小时的每小时快照，以及一到两个月的每日快照。 确保使用数据和时间信息命名快照。
* 每天至少有一次确保在数据中心外部或至少在运行Redis实例的物理机器外部传输RDB快照。

如果运行仅启用AOF持久性的Redis实例，则仍可以复制AOF以创建备份。 该文件可能缺少最后一部分，但Redis仍然可以加载它（请参阅前面有关截断的AOF文件的部分）。

### Disaster recovery

Redis环境中的灾难恢复基本上与备份相同，并且能够在许多不同的外部数据中心中传输这些备份。 这样，即使在某些灾难性事件影响Redis正在运行并生成其快照的主数据中心的情况下，数据也是安全的。

由于许多Redis用户都处于启动阶段，因此没有足够的资金支出，我们将审查最有趣的灾难恢复技术，这些技术的成本不会太高。

* Amazon S3和其他类似服务是实施灾难恢复系统的好方法。 只需以加密形式将每日或每小时RDB快照传输到S3即可。 您可以使用gpg -c加密数据（在对称加密模式下）。 确保将密码存储在许多不同的安全位置（例如，将副本提供给组织中最重要的人员）。 建议使用多个存储服务以提高数据安全性。
* 使用SCP（SSH的一部分）将快照传输到远程服务器。 这是一个相当简单和安全的途径：在离你很远的地方获得一个小VPS，在那里安装ssh，并生成一个没有密码短语的ssh客户端密钥，然后将其添加到小VPS的authorized_keys文件中。 您已准备好以自动方式传输备份。 在两个不同的提供商中获得至少两个VPS以获得最佳结果。

重要的是要理解，如果没有以正确的方式实施，该系统很容易失败。 至少要确保在传输完成后您能够验证文件大小（应该与您复制的文件相匹配）以及可能的SHA1摘要（如果您使用的是VPS）。

如果由于某种原因传输新备份不起作用，您还需要某种独立的警报系统。