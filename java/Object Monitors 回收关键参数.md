# Object Monitors 回收关键参数

本文基于源码 jdk 17+35 分支进行的分析，不同的 jdk 版本可能不适用

源码地址:https://github.com/openjdk/jdk

## 背景

在公司内部某个服务线上环境出现 oom 问题后，经排查，为回收 Object Monitors 线程未能正常获取 cpu 运行时间而导致 Object Monitors 无法被正常回收，为解决此问题，可在 jvm_opt 中添加以下参数

## 1. -XX:GuaranteedSafepointInterval

间隔多少 ms 生成一个安全点

```sh
-XX:GuaranteedSafepointInterval=1000 # 默认值是 1000，为 0 时无意义，最大为 max_jint
```

安全点，同 STW，线程运行到安全点时，具体使用的对象，锁等信息才能被确认，GC 线程才确认哪部分对象是可以被回收的等等，感兴趣的同学可以自行摆渡

## 2. -XX:AvgMonitorsPerThreadEstimate

平均每个线程创建 Object Monitors 的数量，**推荐配置 -XX:AvgMonitorsPerThreadEstimate=50**，并发高的程序可适当调整，不够时 JVM 会自动膨胀

在 ObjectSynchronizer 初始化是用于设置当个每个线程创建的上限（ceiling），不够时，上限将会按一定比例膨胀，见配置 4

通过 NMT 和 monitorinflation 日志分析，一个 Object Monitor 的大小大概在 204bytes，所以理论上 jvm 中 Object Monitors  将占用 AvgMonitorsPerThreadEstimate * count(thrad) * MonitorUsedDeflationThreshold * 204的空间

```sh
-XX:AvgMonitorsPerThreadEstimate=1024  # 最大为 max_uintx，range(0, max_uintx)
```

## 3. -XX:MonitorUsedDeflationThreshold

Object Monitors 回收系数，达到上限的百分之多少进行回收，**可不配置**

```sh
-XX:MonitorUsedDeflationThreshold=90 # 默认配置 90% ，range(0, 100)，为 0 可能导致 Object Monitors 无法被回收
```

## 4. -XX:NoAsyncDeflationProgressMax

调用 Object Monitors 回收方法却没有任何 Object Monitors 可以回收的最大容忍次数，超过或等于这个次数将会对平均每个线程创建的最大上限进行膨胀，**可不配置**

```sh
-XX:NoAsyncDeflationProgressMax=3 # 默认配置，range(0, max_uintx)，为 0 时将不对上限进行膨胀，最大为 max_jint
```

膨胀公式为

```sh
ceiling = ceiling + (1 - MonitorUsedDeflationThreshold) * ceiling +1  # +1 避免 MonitorUsedDeflationThreshold 为 100% 时做无效膨胀
```

## 5. -XX:AsyncDeflationInterval

Object Monitors 回收间隔，单位 ms，**可不配置**

```sh
-XX:AsyncDeflationInterval=250 # 默认配置，range(0, max_jint)，为 0 时可能导致 Object Monitors 无法被回收，最大为 max_jint
```

## 6. -XX:MonitorDeflationMax

单次回收 Object Monitors 的最大数量，避免长时间 gc，**可不配置**

如果 Object Monitors 回收占用了大量时间，可以适当调低此值

```sh
-XX:MonitorDeflationMax=1000000 # 默认配置，range(1024, max_jint)
```

## 7. -Xlog:monitorinflation=debug

打印 Object Monitors 膨胀和回收的日志信息，**线上环境不推荐配置**，可用于开发环境和测试环境观测 Object Monitors 的使用情况

```sh
-Xlog:monitorinflation=debug # [647.296s][debug][monitorinflation] Checking in_use_list: [647.296s][debug][monitorinflation] count=2948, max=2948 [647.296s][debug][monitorinflation] in_use_count=2948 equals ck_in_use_count=2948 [647.296s][debug][monitorinflation] in_use_max=2948 equals ck_in_use_max=2948 [647.296s][debug][monitorinflation] No errors found in in_use_list checks. 13:04:56,009 [DataValidatorHealthCheckJob-web-async-7]  INFO   webservice.service.job.DataValidatorHealthCheckJob start executing job webservice-service-jobs.DataValidatorHealthCheckJob 13:04:56,012 [DataValidatorHealthCheckJob-web-async-7]  ERROR  webservice.service.job.DataValidatorHealthCheckJob Get bad response for health check reason: 获取数据任务状态失败 13:04:56,012 [DataValidatorHealthCheckJob-web-async-7]  INFO   webservice.service.job.DataValidatorHealthCheckJob finished executing job webservice-service-jobs.DataValidatorHealthCheckJob [652.254s][debug][monitorinflation] begin deflating: in_use_list stats: ceiling=3240, count=2951, max=2951 [652.255s][debug][monitorinflation] before handshaking: unlinked_count=2944, in_use_list stats: ceiling=3240, count=7, max=2951 [652.255s][debug][monitorinflation] Checking in_use_list: [652.255s][debug][monitorinflation] count=7, max=2951 [652.255s][debug][monitorinflation] in_use_count=7 equals ck_in_use_count=7 [652.255s][debug][monitorinflation] in_use_max=2951 equals ck_in_use_max=2951 [652.255s][debug][monitorinflation] No errors found in in_use_list checks. [652.255s][debug][monitorinflation] after handshaking: in_use_list stats: ceiling=3240, count=7, max=2951 [652.255s][debug][monitorinflation] deflated 2944 monitors in 0.0007898 secs [652.256s][debug][monitorinflation] end deflating: in_use_list stats: ceiling=3240, count=7, max=2951 13:05:00,008 [HistoricalStatJob-web-async-16]  INFO   webservice.service.job.HistoricalStatJob start executing job webservice-service-jobs.HistoricalStatJob 13:05:00,374 [HistoricalStatJob-web-async-16]  INFO   webservice.service.job.HistoricalStatJob finished executing job webservice-service-jobs.HistoricalStatJob [656.255s][debug][monitorinflation] Checking in_use_list: [656.255s][debug][monitorinflation] count=97, max=2951 [656.255s][debug][monitorinflation] in_use_count=97 equals ck_in_use_count=97 [656.255s][debug][monitorinflation] in_use_max=2951 equals ck_in_use_max=2951 [656.255s][debug][monitorinflation] No errors found in in_use_list checks.
```

为 trace 时将会打印 Object Monitors 的具体情况

```sh
[3055.469s][trace][monitorinflation] in_use_max=11739 equals ck_in_use_max=11739 [3055.469s][trace][monitorinflation] No errors found in in_use_list checks. [3055.469s][trace][monitorinflation] In-use monitor info: [3055.469s][trace][monitorinflation] (B -> is_busy, H -> has hash code, L -> lock status) [3055.469s][trace][monitorinflation]            monitor  BHL              object         object type [3055.469s][trace][monitorinflation] ==================  ===  ==================  ================== [3055.469s][trace][monitorinflation] 0x00007f5ea4092c30  000  0x0000000000000000   [3055.469s][trace][monitorinflation] 0x00007f5ea4084040  000  0x0000000000000000   [3055.469s][trace][monitorinflation] 0x00007f5e88049310  000  0x0000000000000000   [3055.469s][trace][monitorinflation] 0x00007f5ea402dd80  000  0x0000000000000000 [3055.482s][trace][monitorinflation] 0x00007f5e6049e3f0  000  0x0000000000000000   [3055.482s][trace][monitorinflation] 0x00007f5eb012d720  000  0x00000000e55e4fd0  org.apache.coyote.http11.upgrade.UpgradeGroupInfo [3055.482s][trace][monitorinflation] 0x00007f5ebc035550  000  0x00000000e0f114c8  java.util.concurrent.ConcurrentHashMap [3055.482s][trace][monitorinflation] 0x00007f5ebc041c70  000  0x0000000000000000   [3055.482s][trace][monitorinflation] 0x00007f5ebc027570  000  0x0000000000000000   [3055.482s][trace][monitorinflation] 0x00007f5eac306c80  000  0x0000000000000000   [3055.482s][trace][monitorinflation] 0x00007f5eac2ff580  000  0x0000000000000000   [3055.482s][trace][monitorinflation] 0x00007f5ebc03fc90  000  0x0000000000000000   [3055.482s][trace][monitorinflation] 0x00007f5ebc027170  000  0x0000000000000000   [3055.482s][trace][monitorinflation] 0x00007f5e7403bc80  000  0x00000000e205f6d0  jdk.internal.net.http.ConnectionPool
```