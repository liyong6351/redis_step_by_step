<a href="../README.md">README</a>

# Redis Signals Handling

本文档提供了有关Redis如何对接收不同POSIX信号（如SIGTERM，SIGSEGV等）做出反应的信息。  
本文档中包含的信息仅适用于Redis 2.6或更高版本。

## Handling of SIGTERM

SIGTERM信号告诉Redis正常关机。当收到此信号时，服务器实际上并不会退出，但它会调度一个非常类似于调用SHUTDOWN命令时执行的关闭。计划关闭会尽快启动，特别是只要执行中的当前命令终止（如果有），可能会有0.1秒或更短的额外延迟。  
如果服务器被占用太多时间的Lua脚本阻止，如果脚本可以使用SCRIPT KILL运行，则在脚本被终止后立即执行计划关闭，或者如果自发终止，则执行关闭。  
在此条件下执行的关闭包括 以下行动：

* 如果有个子进程在处理RDB写入或者AOF重定向写入的话，那么直接kill掉
* 如果AOF正在使用，那么Redis将会调用fsync系统函数将缓冲区的数据写入到AOF文件中。
* 如果Redis配置为使用RDB文件保留在磁盘上，则执行同步（阻塞）保存。 由于以同步方式执行保存，因此不使用额外的存储器。
* 如果服务是后台运行的，那么pid将会被删除
* 如果启用了Unix域套接字，则会删除它。
* 服务器退出时退出代码为零。

如果无法保存RDB文件，则关闭失败，服务器继续运行以确保不会丢失数据。 由于Redis 2.6.11将不再进一步关闭，除非接收到新的SIGTERM或发出SHUTDOWN命令。

## Handling of SIGSEGV, SIGBUS, SIGFPE and SIGILL

以下跟随信号作为Redis崩溃处理：

* SIGSEGV
* SIGBUS
* SIGFPE
* SIGILL

一旦捕获了其中一个信号，Redis就会中止任何当前操作并执行以下操作：

* 在日志文件上生成错误报告。 这包括堆栈跟踪，寄存器转储以及有关客户端状态的信息。
* 从Redis 2.8开始，执行快速内存测试，作为崩溃系统可靠性的第一次检查。
* 如果服务是后台运行的，那么pid将会被删除
* 最后，服务器为接收到的信号注销其自己的信号处理程序，并再次向自身发送相同的信号，以确保执行默认操作，例如将核心转储到文件系统上。

## What happens when a child process gets killed

当执行AOF重定向的子进程被信号终止的时候，Redis认为这是个Error并且丢弃这个AOF文件。稍后将重新触发重写。  
当执行RDB保存的子节点被杀死时，Redis将处理该条件作为更严重的错误，因为虽然缺少AOF文件重写的效果是AOF文件放大，但是失败的RDB文件创建的效果缺乏持久性。  
由于子节点生成RDB文件被信号杀死，或者子节点出现错误（非零退出代码），Redis进入特殊错误条件，不再接受写入命令。

* Redis将继续回复读取命令。
* Redis将回复所有写入命令并出现MISCONFIG错误。

只有在可以成功创建RDB文件后才会清除此错误情况。

## Killing the RDB file without triggering an error condition
但是，有时用户可能想要杀死RDB保存子，而不会产生错误。 从Redis版本2.6.10开始，这可以使用以特殊方式处理的特殊信号SIGUSR1来完成：它会像其他信号一样杀死子进程，但父进程不会将此检测为严重错误并将继续提供服务 通常写请求。