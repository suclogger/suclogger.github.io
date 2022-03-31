---
title: 记一个线上bug的解决
comments: true
toc: true
categories:
  - 编程
tags:
  - java    
date: 2016-05-27 22:56:03
---
<!-- abstract -->
<!-- 开始正文 -->

## 背景
我司的爬虫现在运行在一个通过ADSL拨号联网的VPS上，通过重新拨号切换IP来应对目标网站的防爬措施。

## 问题现象
应用通过[druid](https://github.com/alibaba/druid)数据源连接到mysql保存数据，每次切换IP后，会通过下面这条简单sql尝试获取数据源连接：
```
SELECT '1' FROM DUAL; 
```
但是通过查看日志这条sql每次都要等待长达17分钟才能执行完成：
![](/image/3c91ebb0cbc84db0befa2f276df8fa19.jpg?r=59)

## 问题分析
数据库的连接是长连接，IP切换之后原先的TCP长连接也都被断开，需要重新获取数据库连接，但是VPS到数据的连接延时稳定在20ms以内，ADSL拨号重建连接的耗时也在2s以内，重建TCP链路不应该耗时17分钟之久：
    ![](/image/2016-05-27 23-28-05.jpg)
### [@bleedfly](https://twitter.com/bleedfly)猜测mysql服务器可能受到DNS污染。
手动重新拨号之后ping接mysql域名，发现解析正确。
通过`yum install mysql`命令在VPS（CentOS）主机上安装mysql客户端后也可以在拨号之后迅速正常连接到mysql，遂排除这一可能。

### 尝试根据内存dump文件发现问题
通过以下命令：
```
 sudo jmap -dump:live,format=b,file=/root/log.bin `pgrep java`
```
获取内存dump文件，放入[MAT](http://www.eclipse.org/mat/downloads.php)中分析：


![](/image/2016-05-28 00-00-55.jpg)


可以看到停顿时内存的使用量很小，不存在内存泄露的可能。
列出的`suspect problem`中也看只反映了当时内存中主要存活的对象是`org.apache.ibatis.session.Configuration`。

### 分析堆栈日志
通过以下命令：
```
 jstack `pgrep java` > jstack.log
```
获取方法调用堆栈进行分析。
```
2016-05-27 20:14:56
Full thread dump OpenJDK 64-Bit Server VM (24.95-b01 mixed mode):

"Attach Listener" daemon prio=10 tid=0x00007f1b14001000 nid=0x28a2 waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"pool-3-thread-1" prio=10 tid=0x00007f1af80c0800 nid=0x274b runnable [0x00007f1b02286000]
   java.lang.Thread.State: RUNNABLE
	at java.net.SocketInputStream.socketRead0(Native Method)
	at java.net.SocketInputStream.read(SocketInputStream.java:152)
	at java.net.SocketInputStream.read(SocketInputStream.java:122)
	at com.mysql.jdbc.util.ReadAheadInputStream.fill(ReadAheadInputStream.java:114)
	at com.mysql.jdbc.util.ReadAheadInputStream.readFromUnderlyingStreamIfNecessary(ReadAheadInputStream.java:161)
	at com.mysql.jdbc.util.ReadAheadInputStream.read(ReadAheadInputStream.java:189)
	- locked <0x0000000785ce9b80> (a com.mysql.jdbc.util.ReadAheadInputStream)
	at com.mysql.jdbc.MysqlIO.readFully(MysqlIO.java:2536)
	at com.mysql.jdbc.MysqlIO.reuseAndReadPacket(MysqlIO.java:2989)
	at com.mysql.jdbc.MysqlIO.reuseAndReadPacket(MysqlIO.java:2978)
	at com.mysql.jdbc.MysqlIO.checkErrorPacket(MysqlIO.java:3526)
	at com.mysql.jdbc.MysqlIO.sendCommand(MysqlIO.java:1989)
	at com.mysql.jdbc.ConnectionImpl.pingInternal(ConnectionImpl.java:4051)
	at com.mysql.jdbc.ConnectionImpl.ping(ConnectionImpl.java:4028)
	at sun.reflect.GeneratedMethodAccessor14.invoke(Unknown Source)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:606)
	at com.alibaba.druid.pool.vendor.MySqlValidConnectionChecker.isValidConnection(MySqlValidConnectionChecker.java:79)
	at com.alibaba.druid.pool.DruidAbstractDataSource.testConnectionInternal(DruidAbstractDataSource.java:1166)
	at com.alibaba.druid.pool.DruidDataSource.getConnectionDirect(DruidDataSource.java:668)
	at com.alibaba.druid.pool.DruidDataSource.getConnection(DruidDataSource.java:628)
	at com.alibaba.druid.pool.DruidDataSource.getConnection(DruidDataSource.java:618)
	at com.alibaba.druid.pool.DruidDataSource.getConnection(DruidDataSource.java:79)
	at org.springframework.jdbc.datasource.DataSourceUtils.doGetConnection(DataSourceUtils.java:111)
	at org.springframework.jdbc.datasource.DataSourceUtils.getConnection(DataSourceUtils.java:77)
	at org.mybatis.spring.transaction.SpringManagedTransaction.openConnection(SpringManagedTransaction.java:83)
	at org.mybatis.spring.transaction.SpringManagedTransaction.getConnection(SpringManagedTransaction.java:69)
	at org.apache.ibatis.executor.BaseExecutor.getConnection(BaseExecutor.java:271)
	at org.apache.ibatis.executor.SimpleExecutor.prepareStatement(SimpleExecutor.java:69)
	at org.apache.ibatis.executor.SimpleExecutor.doQuery(SimpleExecutor.java:56)
	at org.apache.ibatis.executor.BaseExecutor.queryFromDatabase(BaseExecutor.java:259)
	at org.apache.ibatis.executor.BaseExecutor.query(BaseExecutor.java:132)
	at org.apache.ibatis.executor.CachingExecutor.query(CachingExecutor.java:105)
	at org.apache.ibatis.executor.CachingExecutor.query(CachingExecutor.java:81)
	at org.apache.ibatis.session.defaults.DefaultSqlSession.selectList(DefaultSqlSession.java:104)
	at org.apache.ibatis.session.defaults.DefaultSqlSession.selectList(DefaultSqlSession.java:98)
	at org.apache.ibatis.session.defaults.DefaultSqlSession.selectOne(DefaultSqlSession.java:62)
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:57)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:606)
	at org.mybatis.spring.SqlSessionTemplate$SqlSessionInterceptor.invoke(SqlSessionTemplate.java:358)
	at com.sun.proxy.$Proxy13.selectOne(Unknown Source)
	at org.mybatis.spring.SqlSessionTemplate.selectOne(SqlSessionTemplate.java:163)
	at org.apache.ibatis.binding.MapperMethod.execute(MapperMethod.java:63)
	at org.apache.ibatis.binding.MapperProxy.invoke(MapperProxy.java:43)
	at com.sun.proxy.$Proxy27.initCookie(Unknown Source)
	at com.renault.cookie.manager.impl.CookieManagerImpl.getCookie(CookieManagerImpl.java:127)
	at com.renault.core.webmagic.cookie.WcXcCookieProvider.getCookie(WcXcCookieProvider.java:40)
	at us.codecraft.webmagic.downloader.HttpClientGenerator.generateCookie(HttpClientGenerator.java:92)
	at us.codecraft.webmagic.downloader.HttpClientGenerator.generateClient(HttpClientGenerator.java:73)
	at us.codecraft.webmagic.downloader.HttpClientGenerator.getClient(HttpClientGenerator.java:45)
	at us.codecraft.webmagic.downloader.HttpClientDownloader.getHttpClient(HttpClientDownloader.java:69)
	at us.codecraft.webmagic.downloader.HttpClientDownloader.download(HttpClientDownloader.java:97)
	at us.codecraft.webmagic.Spider.processRequest(Spider.java:409)
	at us.codecraft.webmagic.Spider$1.run(Spider.java:322)
	at us.codecraft.webmagic.thread.CountableThreadPool$1.run(CountableThreadPool.java:74)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1145)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:615)
	at java.lang.Thread.run(Thread.java:745)

"DestroyJavaVM" prio=10 tid=0x00007f1b4c00c000 nid=0x2723 waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"pool-2-thread-1" prio=10 tid=0x00007f1b4c1d2800 nid=0x2743 waiting on condition [0x00007f1b03518000]
   java.lang.Thread.State: WAITING (parking)
	at sun.misc.Unsafe.park(Native Method)
	- parking to wait for  <0x00000007831c9fd8> (a java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject)
	at java.util.concurrent.locks.LockSupport.park(LockSupport.java:186)
	at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(AbstractQueuedSynchronizer.java:2043)
	at us.codecraft.webmagic.thread.CountableThreadPool.execute(CountableThreadPool.java:61)
	at us.codecraft.webmagic.Spider.run(Spider.java:318)
	at com.renault.core.webmagic.craw.WcXcCrawler.crawl(WcXcCrawler.java:66)
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:57)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:606)
	at org.springframework.scheduling.support.ScheduledMethodRunnable.run(ScheduledMethodRunnable.java:65)
	at org.springframework.scheduling.support.DelegatingErrorHandlingRunnable.run(DelegatingErrorHandlingRunnable.java:54)
	at org.springframework.scheduling.concurrent.ReschedulingRunnable.run(ReschedulingRunnable.java:81)
	at java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:471)
	at java.util.concurrent.FutureTask.run(FutureTask.java:262)
	at java.util.concurrent.ScheduledThreadPoolExecutor$ScheduledFutureTask.access$201(ScheduledThreadPoolExecutor.java:178)
	at java.util.concurrent.ScheduledThreadPoolExecutor$ScheduledFutureTask.run(ScheduledThreadPoolExecutor.java:292)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1145)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:615)
	at java.lang.Thread.run(Thread.java:745)

"commons-pool-EvictionTimer" daemon prio=10 tid=0x00007f1b4cb7d000 nid=0x2742 in Object.wait() [0x00007f1b034d8000]
   java.lang.Thread.State: TIMED_WAITING (on object monitor)
	at java.lang.Object.wait(Native Method)
	- waiting on <0x0000000782824838> (a java.util.TaskQueue)
	at java.util.TimerThread.mainLoop(Timer.java:552)
	- locked <0x0000000782824838> (a java.util.TaskQueue)
	at java.util.TimerThread.run(Timer.java:505)

"Druid-ConnectionPool-Destory" daemon prio=10 tid=0x00007f1b4ca8f000 nid=0x273f waiting on condition [0x00007f1b1804e000]
   java.lang.Thread.State: TIMED_WAITING (sleeping)
	at java.lang.Thread.sleep(Native Method)
	at com.alibaba.druid.pool.DruidDataSource$DestroyConnectionThread.run(DruidDataSource.java:1292)

"Druid-ConnectionPool-Create" daemon prio=10 tid=0x00007f1b4ca88800 nid=0x273e waiting on condition [0x00007f1b1808f000]
   java.lang.Thread.State: WAITING (parking)
	at sun.misc.Unsafe.park(Native Method)
	- parking to wait for  <0x0000000785a217a8> (a java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject)
	at java.util.concurrent.locks.LockSupport.park(LockSupport.java:186)
	at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(AbstractQueuedSynchronizer.java:2043)
	at com.alibaba.druid.pool.DruidDataSource$CreateConnectionThread.run(DruidDataSource.java:1195)

"Service Thread" daemon prio=10 tid=0x00007f1b4c1b1800 nid=0x2730 runnable [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"C2 CompilerThread1" daemon prio=10 tid=0x00007f1b4c1ae800 nid=0x272f waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"C2 CompilerThread0" daemon prio=10 tid=0x00007f1b4c1ac800 nid=0x272e waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"Signal Dispatcher" daemon prio=10 tid=0x00007f1b4c1aa000 nid=0x272d runnable [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"Surrogate Locker Thread (Concurrent GC)" daemon prio=10 tid=0x00007f1b4c1a8000 nid=0x272c waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"Finalizer" daemon prio=10 tid=0x00007f1b4c17b800 nid=0x272b in Object.wait() [0x00007f1b4854b000]
   java.lang.Thread.State: WAITING (on object monitor)
	at java.lang.Object.wait(Native Method)
	- waiting on <0x0000000785a217c0> (a java.lang.ref.ReferenceQueue$Lock)
	at java.lang.ref.ReferenceQueue.remove(ReferenceQueue.java:135)
	- locked <0x0000000785a217c0> (a java.lang.ref.ReferenceQueue$Lock)
	at java.lang.ref.ReferenceQueue.remove(ReferenceQueue.java:151)
	at java.lang.ref.Finalizer$FinalizerThread.run(Finalizer.java:209)

"Reference Handler" daemon prio=10 tid=0x00007f1b4c179800 nid=0x272a in Object.wait() [0x00007f1b50078000]
   java.lang.Thread.State: WAITING (on object monitor)
	at java.lang.Object.wait(Native Method)
	- waiting on <0x0000000785a217f0> (a java.lang.ref.Reference$Lock)
	at java.lang.Object.wait(Object.java:503)
	at java.lang.ref.Reference$ReferenceHandler.run(Reference.java:133)
	- locked <0x0000000785a217f0> (a java.lang.ref.Reference$Lock)

"VM Thread" prio=10 tid=0x00007f1b4c175000 nid=0x2729 runnable 

"Gang worker#0 (Parallel GC Threads)" prio=10 tid=0x00007f1b4c01d800 nid=0x2724 runnable 

"Gang worker#1 (Parallel GC Threads)" prio=10 tid=0x00007f1b4c01f800 nid=0x2725 runnable 

"Gang worker#2 (Parallel GC Threads)" prio=10 tid=0x00007f1b4c021000 nid=0x2726 runnable 

"Gang worker#3 (Parallel GC Threads)" prio=10 tid=0x00007f1b4c023000 nid=0x2727 runnable 

"Concurrent Mark-Sweep GC Thread" prio=10 tid=0x00007f1b4c0a3800 nid=0x2728 runnable 
"VM Periodic Task Thread" prio=10 tid=0x00007f1b4c1bc800 nid=0x2731 waiting on condition 

JNI global references: 150
```

排除了标明为`RUNNABLE`的线程之后，首先寻找是否有处于`Deadlock`状态的线程。
没有发现Deadlock，如果有可能等待时间就不止17分钟了。

接下来罗列处于`WAITING`状态的线程：
1. "pool-2-thread-1" prio=10 tid=0x00007f1b4c1d2800 nid=0x2743 waiting on condition [0x00007f1b03518000] java.lang.Thread.State: WAITING (parking)
2. "commons-pool-EvictionTimer" daemon prio=10 tid=0x00007f1b4cb7d000 nid=0x2742 in Object.wait() [0x00007f1b034d8000] java.lang.Thread.State: TIMED_WAITING (on object monitor)
3. "Druid-ConnectionPool-Destory" daemon prio=10 tid=0x00007f1b4ca8f000 nid=0x273f waiting on condition [0x00007f1b1804e000] java.lang.Thread.State: TIMED_WAITING (sleeping)
4. "Druid-ConnectionPool-Create" daemon prio=10 tid=0x00007f1b4ca88800 nid=0x273e waiting on condition [0x00007f1b1808f000] java.lang.Thread.State: WAITING (parking)
5. "Finalizer" daemon prio=10 tid=0x00007f1b4c17b800 nid=0x272b in Object.wait() [0x00007f1b4854b000]  java.lang.Thread.State: WAITING (on object monitor)
6. "Reference Handler" daemon prio=10 tid=0x00007f1b4c179800 nid=0x272a in Object.wait() [0x00007f1b50078000]  java.lang.Thread.State: WAITING (on object monitor)

其中，第2个："commons-pool-EvictionTimer" 用于空闲对象的驱逐，第5个："Finalizer"线程用于在垃圾收集前，调用对象的finalize()方法，第6个："Reference Handler"用于处理引用对象本身（软引用、弱引用、虚引用）的垃圾回收问题 ，这3个线程都从属于JVM，暂时忽略。

粗略看来剩下的3个线程中，第1个和第4个都处于`WAITING (parking)`的状态，即依赖于`特定condition`的发生，而第3个处于`TIMED_WAITING (sleeping)`，很可能是方法调用了Thread.wait()或者Thread.sleep()。

查看pom文件，找到版本号`0.2.9`，在druid目录通过check out到对应版本号的tag定位源代码：
```
    public class DestroyConnectionThread extends Thread {

        public DestroyConnectionThread(String name){
            super(name);
            this.setDaemon(true);
        }

        public void run() {
            initedLatch.countDown();

            for (;;) {
                // 从前面开始删除
                try {
                    if (closed) {
                        break;
                    }

                    if (timeBetweenEvictionRunsMillis > 0) {
                        Thread.sleep(timeBetweenEvictionRunsMillis);
                    } else {
                        Thread.sleep(1000); // 这里是1292行
                    }

                    if (Thread.interrupted()) {
                        break;
                    }

                    shrink(true);

                    if (isRemoveAbandoned()) {
                        removeAbandoned();
                    }
                } catch (InterruptedException e) {
                    break;
                }
            }
        }

    }
```
这里的代码逻辑是这个独立的线程是用于回收关闭的链接，由于run方法一开始已经调用了`initedLatch.countDown();`，应该不会影响主线程，所以第3个线程也不是罪魁祸首。

找来找去，怀疑的目光又落在了`pool-3-thread-1`这个最早看来人畜无害的`Runable`线程。
猛烈google之后，发现了相关的讨论：[Long response waiting using Connector](https://bugs.mysql.com/bug.php?id=9515)
>always close the connection and use Socket.setSoTimeout(int) on every connection, I solve my problem. the hanging does never appear again.

又返回去看druid的源码，发现方法调用堆栈中`isValidConnection`是支持设置超时时间的，需要在`DruidDataSource`实例化时作为参数传入。
## 尝试
修改xml配置，给`datasource`的bean中加上下列配置：
```
<property name="validationQueryTimeout" value="1000"/>
```
但是实际发现没有起作用，底层的socket连接依然在等待。又发现druid最新版本（1.0.20）中已经给`validationQueryTimeout`设置了默认值`1000ms`，尝试更新druid，最新版依然存在问题。

## 解决
又认真想了一遍。
druid通过`MySqlValidConnectionChecker`调用`com.mysql.jdbc.ConnectionImpl.pingInternal(boolean checkForClosedConnection, int timeoutMillis)`方法，并将validationQueryTimeout作为它的超时值。

druid作为一个datasource pool，不管是选择在收回connection到pool的时候检验connection，还是从poll拿connection的时候检验connection，选择信任`pingInternal()`方法是无可厚非的。

那锅是要由JDBC的驱动里面的`pingInternal()`方法来背吗？也不尽然：[mysq上的讨论](https://bugs.mysql.com/bug.php?id=31353)
>*Socket timeouts*: 
>UNCLOSED socket connection; JVM specs tell us GC does NOT release these resources thus they exhaust JVM resources related to socket operation, leading to SocketInputStream hanging your program. Always close the connection and use Socket.setSoTimeout(int) on every connection.

>Now this is in relation with *pooled connections* and DBCP (or "why does 1.2.1 work", from Bug#30990):
If you're using a connection pool that _doesn't_ use the ConnectionPoolDataSource API to create connections that have lifecycle callbacks, then it is the _pool's_ responsibility to cleanup "logical" pooled connections when they're returned to the pool. Otherwise there is no way for a JDBC driver to tell that a connection is being pooled "above" it, and thus no event/method call where one can hook in to determine that you've been "logically" closed and returned to the pool.
More recent version of DBCP automatically call rollback() on connections that are returned to the pool.

所以，如果我们还想继续使用druid，还是老老实实在重新切换IP断网之前，调用一下`DruidDataDource.close()`方法显式地关闭connection好了。

其实刚开始我们就通过将数据源切换到`DriverManagerDataSource`已经歪打正着解决了问题，但是`DriverManagerDataSource`不是一个连接池，每次都要重新创建connection，所以既然定位了问题，还是用druid更为优雅一些。

解决后可以实现切换动作在1s内完成。
![](/image/2016-05-30 14-17-50.jpg)





