---
title: 从锁死的RUNNABLE线程谈UNIX的I/O模型
comments: true
toc: true
categories:
  - 编程
tags:
  - IO
date: 2018-06-13 09:48:16
---

记录一下JAVA IO库（java version "1.8.0_131"）的一个坑。

<!-- more -->
## 背景
背景是一个爬虫，实际执行网络请求是通过共用一个固定核心线程数的线程池（FixedThreadPool）做下载操作，但是每次运行一段时间后，线程池就被僵尸进程塞满了，表现为新提交到线程池的下载操作都无法被执行。

## 排查
问题是线程池中的线程僵死，很明显要dump出应用的栈信息来分析一下，一般通过jdk的`jstack`工具：

```shell
jstack -l <应用进程号> >> /path/jstack.log
```


jstack输出内容的分析可以参考[
怎么分析线程栈](https://blog.csdn.net/fred_lzy/article/details/53064673)，这里就不赘述。

机械化的操作可以通过高效的工具来完成，直接将输出文件提交到在线堆栈分析的[fastthread](http://fastthread.io/)

各个线程的状态如下：

![](./image/2018-06-13/2018-06-13-23-59-53.jpg)


其中`download`ThreadGroup即为我的下载线程池，可以看到其中8个核心线程都是`RUNNABLE`状态。
> Runnable：一般指该线程正在执行状态中，该线程占用了资源，正在处理某个请求

一个HTTP请求，要么请求成功，要么请求失败，要么超时，为什么会停留在`RUNNABLE`状态僵死呢？
查看堆栈详情如下：

```
download-thread - priority:5 - threadId:0x00007fa7bc003800 - nativeId:0x42c4 - state:RUNNABLE
stackTrace:
java.lang.Thread.State: RUNNABLE
at java.net.SocketInputStream.socketRead0(Native Method)
at java.net.SocketInputStream.socketRead(SocketInputStream.java:116)
at java.net.SocketInputStream.read(SocketInputStream.java:171)
at java.net.SocketInputStream.read(SocketInputStream.java:141)
at sun.security.ssl.InputRecord.readFully(InputRecord.java:465)
at sun.security.ssl.InputRecord.read(InputRecord.java:503)
at sun.security.ssl.SSLSocketImpl.readRecord(SSLSocketImpl.java:983)
- locked <0x00000000c81da728> (a java.lang.Object)
at sun.security.ssl.SSLSocketImpl.readDataRecord(SSLSocketImpl.java:940)
at sun.security.ssl.AppInputStream.read(AppInputStream.java:105)
- locked <0x00000000c81f73c8> (a sun.security.ssl.AppInputStream)
at org.apache.http.impl.io.SessionInputBufferImpl.streamRead(SessionInputBufferImpl.java:137)
at org.apache.http.impl.io.SessionInputBufferImpl.fillBuffer(SessionInputBufferImpl.java:153)
at org.apache.http.impl.io.SessionInputBufferImpl.readLine(SessionInputBufferImpl.java:282)
at org.apache.http.impl.conn.DefaultHttpResponseParser.parseHead(DefaultHttpResponseParser.java:138)
at org.apache.http.impl.conn.DefaultHttpResponseParser.parseHead(DefaultHttpResponseParser.java:56)
at org.apache.http.impl.io.AbstractMessageParser.parse(AbstractMessageParser.java:259)
at org.apache.http.impl.DefaultBHttpClientConnection.receiveResponseHeader(DefaultBHttpClientConnection.java:163)
at org.apache.http.impl.conn.CPoolProxy.receiveResponseHeader(CPoolProxy.java:165)
at org.apache.http.protocol.HttpRequestExecutor.doReceiveResponse(HttpRequestExecutor.java:273)
at org.apache.http.protocol.HttpRequestExecutor.execute(HttpRequestExecutor.java:125)
at org.apache.http.impl.execchain.MainClientExec.execute(MainClientExec.java:272)
at org.apache.http.impl.execchain.ProtocolExec.execute(ProtocolExec.java:185)
at org.apache.http.impl.execchain.RetryExec.execute(RetryExec.java:89)
at org.apache.http.impl.execchain.RedirectExec.execute(RedirectExec.java:111)
at org.apache.http.impl.client.InternalHttpClient.doExecute(InternalHttpClient.java:185)
at org.apache.http.impl.client.CloseableHttpClient.execute(CloseableHttpClient.java:83)
at org.apache.http.impl.client.CloseableHttpClient.execute(CloseableHttpClient.java:56)
...
at java.util.concurrent.FutureTask.run(FutureTask.java:266)
at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149)
at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
at java.lang.Thread.run(Thread.java:748)
Locked ownable synchronizers:
- <0x00000000c6e6fb08> (a java.util.concurrent.ThreadPoolExecutor$Worker)
```

看到这8个线程都是阻塞在了`socketRead0`方法上。
`socketRead0`是jdk提供的native方法，方法签名为：

```java
    /**
     * Reads into an array of bytes at the specified offset using
     * the received socket primitive.
     * @param fd the FileDescriptor
     * @param b the buffer into which the data is read
     * @param off the start offset of the data
     * @param len the maximum number of bytes read
     * @param timeout the read timeout in ms
     * @return the actual number of bytes read, -1 is
     *          returned when the end of the stream is reached.
     * @exception IOException If an I/O error has occurred.
     */
    private native int socketRead0(FileDescriptor fd,
                                   byte b[], int off, int len,
                                   int timeout)
        throws IOException;
```

具体查看jdk中的实现细节（SocketInputStream.c）：

```c
JNIEXPORT jint JNICALL
Java_java_net_SocketInputStream_socketRead0(JNIEnv *env, jobject this,
                                            jobject fdObj, jbyteArray data,
                                            jint off, jint len, jint timeout)
{
..
    if (timeout) {
        nread = NET_Timeout(fd, timeout);
       ...
    }
...
    nread = NET_Read(fd, bufP, len);
...
}
```

如果设置了超时时间，具体调用的是`NET_Timeout`方法，位于`aix_close.c`:

```c
int NET_Timeout(int s, long timeout) {
       ...
        rv = poll(&pfd, 1, timeout);
        ...
```

直接调用了`poll`函数。
> poll提供的功能与select类似，不过在处理流设备时，它能够提供额外的信息。

`poll`函数在`POLLIN`和`POLLERR`时返回，分别指代`普通或优先级带数据可读`和`发生错误`的情况。
如果是可读，调用`NET_Read`方法阻塞读取：

```c
int NET_Read(int s, void* buf, size_t len) {
    BLOCKING_IO_RETURN_INT( s, recv(s, buf, len, 0) );
}

/************** Basic I/O operations here ***************/

/*
 * Macro to perform a blocking IO operation. Restarts
 * automatically if interrupted by signal (other than
 * our wakeup signal)
 */
#define BLOCKING_IO_RETURN_INT(FD, FUNC) {      \
    int ret;                                    \
    threadEntry_t self;                         \
    fdEntry_t *fdEntry = getFdEntry(FD);        \
    if (fdEntry == NULL) {                      \
        errno = EBADF;                          \
        return -1;                              \
    }                                           \
    do {                                        \
        startOp(fdEntry, &self);                \
        ret = FUNC;                             \
        endOp(fdEntry, &self);                  \
    } while (ret == -1 && errno == EINTR);      \
    return ret;                                 \
}

```

好了，代码都看完了，所以为什么`poll`函数返回可读之后，阻塞的读取操作会永远阻塞下去呢？
参考man page中关于`poll`的bug描述：
> See the discussion of spurious readiness notifications under the BUGS section of select(2).

再看man page中关于`select(2)`的bug描述：
> Under Linux, select() may report a socket file descriptor as "ready for reading", while nevertheless a subsequent read blocks. This could for example happen when data has arrived but upon examination has wrong checksum and is discarded. There may be other circumstances in which a file descriptor is spuriously reported as ready. Thus it may be safer to use O_NONBLOCK on sockets that should not block.

大致意思是在某些极端情况下，poll/select返回可读标识不代表真的有数据处于可读状态，一种典型场景就是数据包到达，但是由于checksum校验没有通过而被抛弃（这是由TCP的可靠机制决定的）。

## 总结
由于某些请求的TCP包传输过程中出现异常导致`poll`在没有真实可读数据情况下返回可读标识，使得阻塞的`recv`方法永远阻塞下去，从而使得当前线程一直处于`RUNNABLE`，当线程池的核心线程都被这种线程占据之后，就再也无法处理新提交的任务了。

## 解决

在不更新JDK而且继续使用HttpClient的前提下，可以参考apache提供的[文档](https://hc.apache.org/httpcomponents-client-4.3.x/tutorial/html/connmgmt.html#d5e405)实现某种`Connection eviction`机制，新启一个守护进程定时移除idle长达一定时间的连接。

```java
    ThreadPoolFactory.getThreadpoll().submit(() -> {
        Thread.currentThread().setName("idle-evict-thread");
        try {
            while (true) {
                synchronized (this) {
                    wait(5000);
                    // Close expired connections
                    poolingHttpClientConnectionManager.closeExpiredConnections();
                    // Optionally, close connections
                    // that have been idle longer than 30 sec
                    poolingHttpClientConnectionManager.closeIdleConnections(30, TimeUnit.SECONDS);
                }
            }
        } catch (InterruptedException ex) {
            // terminate
        }
    });
```

另一种方式，就是用异步IO取代现有的阻塞IO，也是JDK修复这个bug的方式，具体代码可以参考[openJDK](https://bugs.openjdk.java.net/browse/JDK-8075484)

## 扩展

### I/O模型

Unix下共有5中可用的I/O模型：
* 阻塞I/O
* 非阻塞I/O
* I/O复用（select和poll）
* 信号驱动式I/O（SIGIO）
* 异步I/O（POSIX的aio_系列函数）

主要介绍下常见的几种I/O。

#### 阻塞I/O
即blocking I/O，体现在JDK的源码中的`NET_RecvFrom`方法，进程调用`recvfrom`函数，其系统调用知道数据报到达且被复制到应用进程的缓冲区中或者发生错误才返回。进程在从调用`recvfrom`函数开始到它返回的整段时间内饰被阻塞的。

![](./image/2018-06-13/2018-06-14-00-51-55.jpg)

#### 非阻塞I/O
即nonblocking I/O，应用进程持续轮询（polling）内核，即对一个非阻塞描述符循环调用上述的`recvfrom`方法，以查看某个操作是否就绪。这么做往往耗费大量的CPU时间。

![](./image/2018-06-13/2018-06-14-01-03-29.jpg)



#### I/O复用模型
即I/O multiplexing，上述介绍的select/poll采用了这一模型。这一模型相对以上，阻塞发生在这两个系统调用之上，而不是真正的I/O系统调用上。看起来由于需要使用select+recvfrom两组命令，这一模型还稍有劣势，而优势就体现在，**使用select/poll可以等待多个描述符就绪**，这样就可以做到用单个线程管理监听多个I/O操作的事件。与其密切相关的阻塞的变种是在多线程中使用阻塞I/O。

![](./image/2018-06-13/2018-06-14-01-23-26.jpg)


#### 异步I/O
即asynchronous I/O，由POSIX规范定义。告知内核启动某个操作，并在**整个操作**（包括将数据从内核复制到用户空间）完成后调用回调函数。

![](./image/2018-06-13/2018-06-14-01-32-48.jpg)

### I/O模型对应的设计模式

#### Reactor
`select/poll` 实现的I/O复用技术归纳起来有两个关键实现点：

* 当多条连接共用一个阻塞对象后，进程只需要在一个阻塞对象上等待，而无须再轮询所有连接。
* 当某条连接有新的数据可以处理时，操作系统会通知进程，进程从阻塞状态返回，开始进行业务处理。

I/O复用结合线程池使用，就是俗称的`Reactor`模式，中文是“反应堆”。有的地方也称为`Dispatcher`模式（在很多开源的系统里面会看到这个名称的类，其实就是实现 Reactor 模式的），更加贴近模式本身的含义，即 I/O 多路复用统一监听事件，收到事件后分配（Dispatch）给某个进程。

Reactor 模式的核心组成部分包括 Reactor 和处理资源池（进程池或线程池），其中 Reactor 负责监听和分配事件，处理资源池负责处理事件。根据Reactor和资源池的数量可以引申出多种组合，以多 Reactor 多进程为例：

![](./image/2018-06-13/2018-06-14-14-43-23.jpg)

1. 父进程中 mainReactor 对象通过 select 监控连接建立事件，收到事件后通过 Acceptor 接收，将新的连接分配给某个子进程。
2. 子进程的 subReactor 将 mainReactor 分配的连接加入连接队列进行监听，并创建一个 Handler 用于处理连接的各种事件。
3. 当有新的事件发生时，subReactor 会调用连接对应的 Handler（即第 2 步中创建的 Handler）来进行响应。
4. Handler 完成 read→业务处理→send 的完整业务流程。

目前著名的开源系统 Nginx 采用的是多 Reactor 多进程，采用多 Reactor 多线程的实现有 Memcache 和 Netty

#### Proactor

Reactor 是非阻塞同步网络模型，因为真正的 read 和 send 操作都需要用户进程同步操作。这里的“同步”指用户进程在执行 read 和 send 这类 I/O 操作的时候是同步的，如果把 I/O 操作改为异步就能够进一步提升性能，这就是异步网络模型 Proactor。

Proactor 称为“前摄器”或者“主动器”。Reactor 可以理解为“来了事件我通知你，你来处理”，而 Proactor 可以理解为“来了事件我来处理，处理完了我通知你”。这里的“我”就是操作系统内核，“事件”就是有新连接、有数据可读、有数据可写的这些 I/O 事件，“你”就是我们的程序代码。

![](./image/2018-06-13/2018-06-14-14-47-28.jpg)

1. Proactor Initiator 负责创建 Proactor 和 Handler，并将 Proactor 和 Handler 都通过 Asynchronous Operation Processor 注册到内核。
2. Asynchronous Operation Processor 负责处理注册请求，并完成 I/O 操作。
3. Asynchronous Operation Processor 完成 I/O 操作后通知 Proactor。
4. Proactor 根据不同的事件类型回调不同的 Handler 进行业务处理。
5. Handler 完成业务处理，Handler 也可以注册新的 Handler 到内核进程。

Windows 下通过 IOCP 实现了真正的异步 I/O，而在 Linux 系统下的 AIO 并不完善，因此在 Linux 下实现高并发网络编程时都是以 Reactor 模式为主。所以即使 Boost.Asio 号称实现了 Proactor 模型，其实它在 Windows 下采用 IOCP，而在 Linux 下是用 Reactor 模式（采用 epoll）模拟出来的异步模型。



## 参考
>  UNIX环境高级编程
>  单服务器高性能模式：Reactor与Proactor 李运华



