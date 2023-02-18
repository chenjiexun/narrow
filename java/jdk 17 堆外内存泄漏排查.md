# jdk 17 堆外内存泄漏排查

## 1. 背景

在维护公司内部某个 java 服务时，使用 top 命令发现这个服务其中一个节点的 java 进程使用了大量的内存，远远超出了 jvm 设的最大 10g 

```sh
# top
top - 03:33:23 up 210 days, 35 min,  0 users,  load average: 0.16, 0.06, 0.06
Tasks:   4 total,   1 running,   3 sleeping,   0 stopped,   0 zombie
%Cpu(s):  3.3 us,  0.3 sy,  0.0 ni, 95.8 id,  0.5 wa,  0.0 hi,  0.0 si,  0.0 st
MiB Mem :  32011.0 total,   1605.3 free,  25929.4 used,   4475.5 buff/cache
MiB Swap:      0.0 total,      0.0 free,      0.0 used.   4087.0 avail Mem 

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND                                                                                                                                         
    8 root      20   0   32.7g  21.2g  24180 S  11.3  67.7   1760:10 java                                                                                                                                            
    1 root      20   0    7064    632    328 S   0.0   0.0   0:00.02 bash                                                                                                                                            
 9844 root      20   0    7228   2156   1660 S   0.0   0.0   0:00.01 bash                                                                                                                                            
 9851 root      20   0    9056   1924   1356 R   0.0   0.0   0:00.00 top
```

在 arthas 中发现堆和元数据空间只使用了 1G 不到的内存

```
Memory                                        used           total          max             usage          GC                                                                                                         
heap                                          271M           1120M          2048M           13.28%         gc.shenandoah_pauses.count                            2892                                                 
shenandoah                                    271M           1120M          2048M           13.28%         gc.shenandoah_pauses.time(ms)                         280                                                  
nonheap                                       175M           213M           -1              82.40%         gc.shenandoah_cycles.count                            723                                                  
metaspace                                     131M           132M           -1              99.40%         gc.shenandoah_cycles.time(ms)                         87522                                                
compressed_class_space                        16M            16M            3072M           0.53%                                                                                                                     
codecache                                     27M            64M            128M            21.61%                                                                                                                    
mapped                                        0K             0K             -               0.00%                                                                                                                     
direct                                        24M            24M            -               100.00%                                                                                                                   
mapped - 'non-volatile memory'                0K             0K             -               0.00% 
```

因此可以判断不是堆内空间导致的大量内存占用，于是开始怀疑是堆外内存导致的占用，然后使用 java 自带的 NMT 工具发现 Object Monitors 使用了大量的内存，大概有 11g

```sh
jcmd pid VM.native_memory

11:

Native Memory Tracking:

(Omitting categories weighting less than 1KB)

Total: reserved=23028689KB, committed=19257169KB
-                 Java Heap (reserved=6291456KB, committed=5971968KB)
                            (mmap: reserved=6291456KB, committed=5971968KB) 
 
-                     Class (reserved=3149142KB, committed=20166KB)
                            (classes #26675)
                            (  instance classes #25025, array classes #1650)
                            (malloc=3414KB #75359) 
                            (mmap: reserved=3145728KB, committed=16752KB) 
                            (  Metadata:   )
                            (    reserved=122880KB, committed=118160KB)
                            (    used=117478KB)
                            (    waste=682KB =0.58%)
                            (  Class space:)
                            (    reserved=3145728KB, committed=16752KB)
                            (    used=15962KB)
                            (    waste=790KB =4.72%)
 
-                    Thread (reserved=79160KB, committed=31792KB)
                            (thread #273)
                            (stack: reserved=78364KB, committed=30996KB)
                            (malloc=479KB #1639) 
                            (arena=318KB #543)
 
-                      Code (reserved=135117KB, committed=69069KB)
                            (malloc=3021KB #18438) 
                            (mmap: reserved=132096KB, committed=66048KB) 
 
-                        GC (reserved=403279KB, committed=198479KB)
                            (malloc=8007KB #21054) 
                            (mmap: reserved=395272KB, committed=190472KB) 
 
-                  Compiler (reserved=3543KB, committed=3543KB)
                            (malloc=3380KB #2252) 
                            (arena=163KB #3)
 
-                  Internal (reserved=3717KB, committed=3717KB)
                            (malloc=3681KB #26364) 
                            (mmap: reserved=36KB, committed=36KB) 
 
-                     Other (reserved=83635KB, committed=83635KB)
                            (malloc=83635KB #145) 
 
-                    Symbol (reserved=28699KB, committed=28699KB)
                            (malloc=26581KB #723205) 
                            (arena=2118KB #1)
 
-    Native Memory Tracking (reserved=894898KB, committed=894898KB)
                            (malloc=57KB #868) 
                            (tracking overhead=894841KB)
 
-        Shared class space (reserved=12288KB, committed=12168KB)
                            (mmap: reserved=12288KB, committed=12168KB) 
 
-               Arena Chunk (reserved=195KB, committed=195KB)
                            (malloc=195KB) 
 
-                   Tracing (reserved=32KB, committed=32KB)
                            (arena=32KB #1)
 
-                    Module (reserved=1620KB, committed=1620KB)
                            (malloc=1620KB #6690) 
 
-                 Safepoint (reserved=8KB, committed=8KB)
                            (mmap: reserved=8KB, committed=8KB) 
 
-           Synchronization (reserved=543544KB, committed=543544KB)
                            (malloc=543544KB #869548) 
 
-            Serviceability (reserved=1KB, committed=1KB)
                            (malloc=1KB #6) 
 
-                 Metaspace (reserved=123822KB, committed=119102KB)
                            (malloc=942KB #972) 
                            (mmap: reserved=122880KB, committed=118160KB) 
 
-      String Deduplication (reserved=3402KB, committed=3402KB)
                            (malloc=3402KB #34003) 
 
-           Object Monitors (reserved=11271131KB, committed=11271131KB)
                            (malloc=11271131KB #55488645) 
```

NMT 功能的开启需要在 jvm 启动参数上加 -XX:NativeMemoryTracking= summary | detail

这里我们只能看到 Object Monitors 使用了大量的内存，但没办法看到具体是哪个方法在使用这块的内存，通过 jstack 也没能看到过多有用的信息，因此我们让运维同学在 compose 文件中加上 -XX:NativeMemoryTracking=detail 参数进行重启，开启基准线

```sh
jcmd pid VM.native_memory baseline
```

运行一段时间后，通过命令拿到详细文件

```sh
jcmd pid VM.native_memory detail.diff
```

发现主要使用内存方法是 ObjectSynchronizer::inflate

```sh
[0x00007fdd4f6262aa] ObjectSynchronizer::inflate(Thread*, oopDesc*, ObjectSynchronizer::InflateCause)+0x12a
[0x00007fdd4f6269f0] ObjectSynchronizer::wait(Handle, long, JavaThread*)+0x50
[0x00007fdd4f10d93f] JVM_MonitorWait+0x15f
[0x00007fdd3840dd1b]
                             (malloc=1188029KB type=Object Monitors +383568KB #5848756 +1888335)
```

至此，我们还是没能知道是哪个具体的线程和对象在使用此方法，需要使用其他工具在进行更详细的分析

## 2. 排查尝试

这里尝试了很多的工具，由于篇幅的关系，就不针对每个工具的使用进行介绍，对于使用到的工具，也未能完整深入的使用，如果某个工具还有某些功能是可以分析这个问题的，还望大佬指教，如在使用这些工具时遇到问题，欢迎一起探讨排坑

因为自己开发机上的镜像底包 java 版本是 17.0.1，这个版本的 NMT 没有统计 「Object Monitors」，后面发现这块内存使用被统计到「 Internal」 了，所以部署了和生产环境同一 JAVA 版本的服务，运行两天后，发现也有此问题，然后查看生产环境的其他节点，也有这个问题，因此可以断定这是一个普遍情况

### 2.1 jstack

这里主要想看一下是否有线程异常，或者死锁的情况，jstack 中能显示当前现场的调用栈，但没办法显示 native 方法的调用，也没办法对 ObjectSynchronizer::inflate 的使用方法进行统计

因此拿不到过多的信息，只能看到有上百个线程在等待，其中的 allocated 代表的是线程所有 alloc 的总计，无法说是堆内还是堆外空间

和自己开发机上部署的服务 jstack 结果进行对比，也没有观察到异常情况

### 2.2 pmap + gdb

通过 pmap 并没有发现进程中有开辟大内存的现象，都是缓慢增加，并且通过 NMT 得到的地址也不一定是内存开辟的地址，这里的主要问题是，没有办法拿到内存开辟的具体地址，因此也无法通过 gdb 查看内存中的数据

```sh
pmap -x -p 8|sort -n -k3 
...... 
00007fb6ecc00000    5120    5120    5120 rw---   [ anon ] 
00007fb716800000    5120    5120    5120 rw---   [ anon ] 
00007fb6ec6f0000    5184    5184    5184 rw---   [ anon ] 
00007fb715ec0000    5376    5376    5376 rw---   [ anon ] 
00007fb754000000    5460    5396    5396 rw---   [ anon ] 
00007fb6f4000000    6056    5964    5964 rw---   [ anon ] 
00007fb720000000    6084    6080    6080 rw---   [ anon ] 
00007fb6e7a00000    6144    6144    6144 rw---   [ anon ] 
00007fb714e00000    6144    6144    6144 rw---   [ anon ] 
00007fb6fc000000    6596    6584    6584 rw---   [ anon ] 
00007fb700000000    6652    6652    6652 rw---   [ anon ] 
00007fb734000000    6944    6944    6944 rw---   [ anon ] 
00007fb72c000000    7464    7464    7464 rw---   [ anon ] 
00007fb6ed400000    8192    8192    8192 rw---   [ anon ] 
00007fb728000000    8332    8332    8332 rw---   [ anon ] 
00007fb704000000   10092   10084   10084 rw---   [ anon ] 
00007fb78b18f000   19324   11560       0 r-x-- /opt/java/openjdk/lib/server/libjvm.so
0000000800000000   12100   11804    4248 rw--- /opt/java/openjdk/lib/server/classes.jsa 00007fb708000000   11860   11844   11844 rw---   [ anon ] 
00007fb6d8000000   14816   11960   11960 rw---   [ anon ] 
00007fb6e0000000   37796   13116   13116 rw---   [ anon ] 
000055acffb6a000   14996   14812   14812 rw---   [ anon ] 
00007fb6e8000000   15820   15740   15740 rw---   [ anon ] 
00007fb750000000   21704   21696   21696 rw---   [ anon ] 
00007fb76c000000   65536   44096   44096 rw---   [ anon ] 
00007fb784000000   65536   52096   52096 rw---   [ anon ] 
00007fb730000000   65516   65492   65492 rw---   [ anon ] 
00007fb71c000000   65532   65532   65532 rw---   [ anon ] 
00007fb774400000   65536   65536   65536 rwx--   [ anon ] 
0000000080000000 2097152 2097152 2097152 rw---   [ anon ] 
total kB         8136404 2761180 2736628
```

### 2.3 jemalloc

有不少文章使了 jemalloc、temalloc 来进行堆外内存的分析，它的主要原理是替换 linux 自带的 malloc 方法，通过 hock 的方式查看哪个方法调用 malloc 方法

不过这个工具使用效果并不理想，只能看到系统的方法调用，并不能看到 java 相关的代码调用链路

```sh
jeprof --show_bytes --pdf `which java` jeprof.*.heap > /tmp/jemalloc.pdf
```

尝试打开 jemalloc 打印的 heap 文件，也没能看到有用信息

```sh
# cat jeprof.out.8.20.i20.heap  
...... 
7f4513be9000-7f4513bfa000 r-xp 00000000 fd:01 6163114                    /tmp/libnetty_transport_native_epoll_x86_6412962575108046029426.so (deleted) 
7f4513bfa000-7f4513df9000 ---p 00011000 fd:01 6163114                    /tmp/libnetty_transport_native_epoll_x86_6412962575108046029426.so (deleted) 
7f4513df9000-7f4513dfc000 r--p 00010000 fd:01 6163114                    /tmp/libnetty_transport_native_epoll_x86_6412962575108046029426.so (deleted) 
7f4513dfc000-7f4513dfd000 rw-p 00000000 00:00 0  
7f4513dfd000-7f4513dff000 r-xp 00000000 fd:01 4325855                   /opt/java/openjdk/lib/libextnet.so 
7f4513dff000-7f4513ffe000 ---p 00002000 fd:01 4325855                    /opt/java/openjdk/lib/libextnet.so 
7f4513ffe000-7f4513fff000 r--p 00001000 fd:01 4325855                    /opt/java/openjdk/lib/libextnet.so 
7f4513fff000-7f4514000000 rw-p 00002000 fd:01 4325855                    /opt/java/openjdk/lib/libextnet.s
```

### 2.4 JMC

看到一些文章用 JMC 监控 java 进程的内存使用，所以对 JMC 进行了尝试，后面发现这个监控和 arthas 的 memory 功能一样，也没法看到 internal 的使用情况

其他一些功能均有替代品

### 2.5 arthas

因为 JVM_MonitorWait 这个方法是 java Object.wait() 的底层实现方法，所以想查看一下  Object.wait()  这个方法被些线程调用过

arthas  中有很多功能可以对一个方法进行监控，比如 monitor，watch，trace 等，针对某个具体方法可以使用这些功能

但是对于一个所有对象都有的方法，应该选用一个可以全局统计的功能，因此选用了火焰图功能，对所有方法进行了统计，因此也有了意外收获

![SearchWait](https://github.com/chenjiexun/narrow/blob/main/pic/java/SearchWait.png)

在搜索 wait 时，发现了我们一直苦苦追寻的 JVM_MonitorWait，但这个方法并不是真凶，因为它并没有调用 ObjectSynchronizer::inflate 方法

因此我们该用 JVM_MonitorWait 进行检索

![HistoricalStatJobWait.png](https://github.com/chenjiexun/narrow/blob/main/pic/java/HistoricalStatJobWait.png)

发现主要是 HistoricalStatJob 这个任务在调用 ObjectSynchronizer::inflate

于是做了关闭这个 job 的对比实验

## 3. 深入排查

通过 HistoricalStatJob 关闭对比实验，我们大体定位到了产生堆外内存不断增长的主要因素，与生产环境获取到的火焰图对比，是同一元素导致的堆外内存增长，但对于这个任务为什么会占用这么多堆外内存的原因还需继续排查

这里大概有两个方向可以去排查

a. ES 相关的组件存在对象长时间引用未回收

b. jdk 的问题

这里想的是，如果是 jdk 的问题，那么所有使用这个版本的公司都应该会有这个问题，而我们在互联网上却基本搜索不到 Object Monitors 占用大量堆外内存相关的文档，所以这里就先排查了是否为 ES 组件的问题

### 3.1 ES 相关的问题

这里主要通过的是 MAT 的直方图功能分析的，如果是 ES 相关的组件存在对象长时间引用未回收的问题，那么在直方图中应该可以搜索到 ES 组件创建的对象，然而我们直方图中却搜索不到任何相关的对象

同时如果是堆内对象长时间占用 Object Monitors 引起的堆外内存增加，按照一个 object 实例对应一个 Object Monitors 的逻辑，那么堆内的内存应该也会不断增大

HistoricalStatJob 的 wait 最终是调用的 BasicFuture 的 wait 方法，所以我们在 MAT 中搜索的 BasicFuture，却发现 BasicFuture 在 JVM 中是没有存活实例的

并且我们通过 idea 在 BasicFuture get 方法上 debug 时，发现 ES 相关的组件是正常调用 get 方法的，并且每次请求都是创建新的 BasicFuture，并且在请求完成后能正常退出

然后我们通过 arthas 的 monitor 功能监控 ，发现该方法确实是正常返回的，所以不存在 wait 方法阻塞协程的情况

```sh
 monitor org.apache.http.concurrent.BasicFuture get
Press Q or Ctrl+C to abort.
Affect(class count: 1 , method count: 2) cost in 122 ms, listenerId: 1
 timestamp                        class                                            method                                            total           success          fail            avg-rt(ms)       fail-rate      
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 2023-01-24 09:56:14              org.apache.http.concurrent.BasicFuture           get                                               133             133              0               5.31             0.00%          

 timestamp                        class                                            method                                            total           success          fail            avg-rt(ms)       fail-rate      
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 2023-01-24 09:57:14              org.apache.http.concurrent.BasicFuture           get                                               130             130              0               4.70             0.00%          
```

在关闭 HistoricalStatJob 的实验中，我们通过 arthas 对服务进行采样生成了火焰图，发现有另一个定时任务也在调用 ObjectSynchronizer::inflate 方法，同时，Object Monitors 堆外内存占用大小也在增加

这一系列现象可以证明 Object Monitors 占用的大量堆外内存的元凶不是 ES 相关的组件

### 3.2 jdk 的问题

目前很多介绍 Object Monitors 的文章都是基于 jdk 8 进行分析的，很少有基于 jdk 17 的文章，为进一步验证 Object Monitors 的回收逻辑，于是查看了 jdk 17 的源码

在 jdk 17 中，Object Monitors 的回收主要有两个线程负责

```
// This function is called by the MonitorDeflationThread to deflate 
// ObjectMonitors. It is also called via do_final_audit_and_print_stats() 
// by the VMThread. 
size_t ObjectSynchronizer::deflate_idle_monitors() {    ...... }
```

一是 MonitorDeflationThread，每隔一段数据检查是否需要回收 Object Monitors，如果需要，则进行回收

```c++
void MonitorDeflationThread::monitor_deflation_thread_entry(JavaThread* jt, TRAPS) { 
	while (true) {    
		{      
			......     
			MonitorLocker ml(MonitorDeflation_lock, Mutex::_no_safepoint_check_flag);      
			while (!ObjectSynchronizer::is_async_deflation_needed()) {        
				// Wait until notified that there is some work to do.        
				// We wait for GuaranteedSafepointInterval so that       
				// is_async_deflation_needed() is checked at the same interval.     	
				ml.wait(GuaranteedSafepointInterval);      
			}   
		}    
		(void)ObjectSynchronizer::deflate_idle_monitors(); 
	}
}
```

二是 VMThread，在 java 进程退出时进行 Object Monitors 的回收

所以这里去检查了一下发生 OOM 节点的 jstack 下的线程运行情况，java 进程运行了三周左右的时间，MonitorDeflationThread 线程才运行了 0.07 ms

```sh
"Monitor Deflation Thread" #6 daemon prio=9 os_prio=0 cpu=0.07ms elapsed=1891404.33s tid=0x00007f91b81469c0 nid=0x19 runnable  [0x0000000000000000]   java.lang.Thread.State: RUNNABLE
```

于是检查了一下自己开发机上的 MonitorDeflationThread 回收情况，和发生 OOM 节点类似，启动后就再也没获得过 cpu 运行时间

```sh
# jstack 8|grep Deflation 
"Monitor Deflation Thread" #6 daemon prio=9 os_prio=0 cpu=0.09ms elapsed=42.98s tid=0x00007f37e8139c60 nid=0x16 runnable  [0x0000000000000000] 
# jstack 8|grep Deflation 
"Monitor Deflation Thread" #6 daemon prio=9 os_prio=0 cpu=0.09ms elapsed=46.14s tid=0x00007f37e8139c60 nid=0x16 runnable  [0x0000000000000000] 
# jstack 8|grep Deflation 
"Monitor Deflation Thread" #6 daemon prio=9 os_prio=0 cpu=0.09ms elapsed=52.77s tid=0x00007f37e8139c60 nid=0x16 runnable  [0x0000000000000000] 
# jstack 8|grep Deflation 
"Monitor Deflation Thread" #6 daemon prio=9 os_prio=0 cpu=0.09ms elapsed=57.49s tid=0x00007f37e8139c60 nid=0x16 runnable  [0x0000000000000000] 
# jstack 8|grep Deflation 
"Monitor Deflation Thread" #6 daemon prio=9 os_prio=0 cpu=0.09ms elapsed=611.73s tid=0x00007f37e8139c60 nid=0x16 runnable  [0x0000000000000000]
```

因此可以确定是 MonitorDeflationThread 线程调度的问题，至于其为什么没有获取 cpu 运行时间，需要再排查

## 4.最终排查

在阅读完相应的 jdk 源码后，我们将所有与 object monitors 回收相关的参数都试了一遍，发现 -XX:GuaranteedSafepointInterval 这个参数是能使 object monitors 回收线程进行工作的，为确定是否是 jdk 的问题，我们在相同的环境中，使用此 jdk 部署了另一个 java 服务，发现新部署的服务却是可以回收 ，说明是我们这个服务自己的问题，于是在 idea 中搜索了一下，发现另一个同事在构建这个服务的镜像时，将这个参数配置成了 0，导致 jvm 一直没法生成安全点

## 5.一点点总结

1. 首先这个问题应该和客户那边的运维同学沟通的时候，他们说只有这个客户有这个问题，后面自己在开发机上部署相同的版本后，发现也有这个问题，于是去查了一下其他客户的情况，也有这个问题，于是判断这是一个普遍存在的问题；一般运维同学观察一个服务是否正常，只是观察一些很基础的东西，其他客户那边配置的内存可能并没有这个客户那么高，所以现象不明显；在排查过程中，可能受限于运维同学的技术水平，他们对一个问题的理解可能并不到位，有时候需要自己真正的去证实一些事情
2. 自己在排查中，使用了 java 版本不同底包构建的镜像，在自己使用的这个版本中，object monitors 的统计放在了 internal 中，导致自己花费了不少时间才复现出来，在后续问题排查过程中，应尽量使用和客户一样的版本进行复现
3. 在 排查到是 -XX:GuaranteedSafepointInterval 这个参数影响MonitorDeflationThread 线程调度时，就一直怀疑时 jdk 的bug，翻看了多次源码后，还是没找到问题，自己就一直在死胡同里面钻，后面想想应该还是那个逻辑：jdk 做为一个如此成熟的产品，不应该有这种低级问题，为什么我们会有这个问题，而其他公司没有出现过这个问题，所以还是得回到我们自己的产品上来排查。当然这和自己对公司项目代码结构不熟悉也有关