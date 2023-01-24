## 验证 Object Monitors 回收线程是否正常运行

### 背景

在公司内部一个 java 17 的服务出现一次堆外内存泄漏的问题后，经排查，是 jvm 中回收 Object Monitors 线程无法正常获得 cpu 运行时间导致 Object Monitors 无法回收引起的，本文旨在提供提供方法验证服务中回收 Object Monitors 的线程是否正常运行

java 版本: 17.0.5

验证的步骤比较麻烦，需要持续观察验证，每运行一段时间后检查一次

### 1 jstack

通过 jps 命令获取 java 服务进程 pid

```
# jps 
643 Jps 
7 xxx.jar
```

然后使用 jstack 命令验证 Object Monitors 回收线程是否获取到 cpu 调度

异常服务 Object Monitors 回收线程在启动后，使用 jstack 命令多次查看， cpu 调度时间（cpu=）会一直停留在一个非常小的值，即使过了很长一段时间，这个值还是停留在一开始查看的那个值

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

正常服务的 Object Monitors 回收线程在启动后，使用 jstack 命令多次查看，cpu 调度时间（cpu=）会不断增长

```sh
# jstack 7|grep Deflation 
"Monitor Deflation Thread" 
#6 daemon prio=9 os_prio=0 cpu=224.83ms elapsed=11183.25s tid=0x00007fc0cc155450 nid=0x15 runnable  [0x0000000000000000] 
# jstack 7|grep Deflation 
"Monitor Deflation Thread" #6 daemon prio=9 os_prio=0 cpu=224.88ms elapsed=11186.72s tid=0x00007fc0cc155450 nid=0x15 runnable  [0x0000000000000000] 
# jstack 7|grep Deflation 
"Monitor Deflation Thread" #6 daemon prio=9 os_prio=0 cpu=224.96ms elapsed=11190.83s tid=0x00007fc0cc155450 nid=0x15 runnable  [0x0000000000000000] 
# jstack 7|grep Deflation 
"Monitor Deflation Thread" #6 daemon prio=9 os_prio=0 cpu=225.05ms elapsed=11195.56s tid=0x00007fc0cc155450 nid=0x15 runnable  [0x0000000000000000]
```

### 2 monitorinflation 日志

查看服务的日志

异常的服务在启动后，其 object monitors 的使用数量会一直增长，且不会被回收

```sh
# docker logs -f --tail=100 containor 
...... 
[5950.041s][debug][monitorinflation] Checking in_use_list: 
[5950.041s][debug][monitorinflation] count=17161, max=17161 
[5950.042s][debug][monitorinflation] in_use_count=17161 equals ck_in_use_count=17161 
[5950.042s][debug][monitorinflation] in_use_max=17161 equals ck_in_use_max=17161 
[5950.042s][debug][monitorinflation] No errors found in in_use_list checks. 
[5950.076s][debug][monitorinflation] Checking in_use_list: 
[5950.076s][debug][monitorinflation] count=17161, max=17161 
[5950.077s][debug][monitorinflation] in_use_count=17161 equals ck_in_use_count=17161 [5950.077s][debug][monitorinflation] in_use_max=17161 equals ck_in_use_max=17161 
[5950.077s][debug][monitorinflation] No errors found in in_use_list checks. 
[5950.079s][debug][monitorinflation] Checking in_use_list: 
[5950.079s][debug][monitorinflation] count=17161, max=17161 
[5950.079s][debug][monitorinflation] in_use_count=17161 equals ck_in_use_count=17161 [5950.079s][debug][monitorinflation] in_use_max=17161 equals ck_in_use_max=17161 
[5950.079s][debug][monitorinflation] No errors found in in_use_list checks.
```

正常的服务启在动后，其 object monitors 的使用数量会增加到一定数值后开始回收，并不再一直增长，且可以观察到回收日志

object monitors 的使用数量最多值可以通过 **线程数* AvgMonitorsPerThreadEstimate** 估算，从而计算大致回收的位置

这里使用的是 -XX:AvgMonitorsPerThreadEstimate=20，线程数在 160 左右，再乘以一个 0.9，所以 object monitors 的使用数量在 2948 时开始了回收 

```sh
# docker logs -f --tail=100 dp-thrall-webservice 
...... 
[647.296s][debug][monitorinflation] Checking in_use_list: 
[647.296s][debug][monitorinflation] count=2948, max=2948 
[647.296s][debug][monitorinflation] in_use_count=2948 equals ck_in_use_count=2948 
[647.296s][debug][monitorinflation] in_use_max=2948 equals ck_in_use_max=2948 
[647.296s][debug][monitorinflation] No errors found in in_use_list checks. 
[652.254s][debug][monitorinflation] begin deflating: in_use_list stats: ceiling=3240, count=2951, max=2951                          # 回收日志，开始回收 
[652.255s][debug][monitorinflation] before handshaking: unlinked_count=2944, in_use_list stats: ceiling=3240, count=7, max=2951		# 回收日志 
[652.255s][debug][monitorinflation] Checking in_use_list: 
[652.255s][debug][monitorinflation] count=7, max=2951 
[652.255s][debug][monitorinflation] in_use_count=7 equals ck_in_use_count=7 
[652.255s][debug][monitorinflation] in_use_max=2951 equals ck_in_use_max=2951 
[652.255s][debug][monitorinflation] No errors found in in_use_list checks. 
[652.255s][debug][monitorinflation] after handshaking: in_use_list stats: ceiling=3240, count=7, max=2951							# 回收日志 
[652.255s][debug][monitorinflation] deflated 2944 monitors in 0.0007898 secs														# 回收日志 
[652.256s][debug][monitorinflation] end deflating: in_use_list stats: ceiling=3240, count=7, max=2951                               # 回收日志，回收结束 
[656.255s][debug][monitorinflation] Checking in_use_list: 
[656.255s][debug][monitorinflation] count=97, max=2951 
[656.255s][debug][monitorinflation] in_use_count=97 equals ck_in_use_count=97 
[656.255s][debug][monitorinflation] in_use_max=2951 equals ck_in_use_max=2951 
[656.255s][debug][monitorinflation] No errors found in in_use_list checks.
```

### 3 NMT

通过 jps 命令获取 java 进程 pid

```sh
# jps 
643 Jps 
8 xxx.jar
```

然后通过 NMT 命令查看异常和正常服务的 Object Monitors 内存占用情况

异常的服务的 webservice 在启动后，其 object monitors 的占用内存会一直增加，且不会被回收

```sh
# jcmd 8 VM.native_memory 
8: Native Memory Tracking: 
(Omitting categories weighting less than 1KB)
                  Total: reserved=4108017KB, committed=859081KB ......  
-                 Metaspace (reserved=115387KB, committed=111819KB) 
                            (malloc=699KB #594)
                            (mmap: reserved=114688KB, committed=111120KB)   
-      String Deduplication (reserved=3258KB, committed=3258KB) 
                            (malloc=3258KB #33878)   
-           Object Monitors (reserved=3520KB, committed=3520KB) 
                            (malloc=3520KB #17327) 
```

正常的服务在启动后，通过 「方法2」 观察到 Object Monitors 回收事件的前后通过 NMT 观察回收前后的内存变化，回收后 object monitors 的内存占用较回收前会有所降低，且不会像异常的服务那样一直增加

```sh
# jcmd 7 VM.native_memory 
7: Native Memory Tracking: 
(Omitting categories weighting less than 1KB) 
Total: reserved=4115681KB, committed=862385KB
......  
-      String Deduplication (reserved=3354KB, committed=3354KB)       
                            (malloc=3354KB #33911)   
-           Object Monitors (reserved=258KB, committed=258KB)
                            (malloc=258KB #1272) 
```