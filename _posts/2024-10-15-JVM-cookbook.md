---
title: JVM 内存排查 Cookbook
date: 2024-10-15 14:17:00 +0800 
categories: [Java, JVM]
tags: [JVM]
author: <Bryce1010>
description: 系统性整理 JVM 问题排查思路，等下次遇到问题的时候可以有个全面的排查流程，不至于漏掉某些可能性误入歧途浪费时间。
toc: true
comments: true
pin: false
---

## 一、收集问题
### 1.1 基本信息收集
首先JAVA内存使用率高并不全是内存问题。可能是新业务或者大促本身流量高导致内存打高。在判断内存问题之前需要先和明确以下几个基本情况：
1.目前的现象是什么：是内存居高不下，内存缓慢增加还是进程突然Dump掉？
2.现象发生的节点，有无变更，有无新业务上线，有无应用本身监控数据留痕。

### 1.2 依赖上面信息作出基本判断：
业务无损情况：
1.业务增加导致的内存增加，往阿里云弹性能力方向引导
2.业务未变，近期未有变更，内存增长是周期性，偶发性。基于现象有基本判断，后续可以围绕基本判断推进排查
	a.周期性增加：往定时任务方向排查
	b.偶发性增长：首先考虑的是在不影响业务情况下的现场复现，后续往这个方向引导
	c.缓慢持续性增长：都有可能，跳转Step2判断来源
在了解情况过程中，如果客户没有进一步分析的监控手段，推荐阿里云ARMS监控内存和GC情况。
业务有损情况：
1.快速止损：业务有损情况下首先需要推荐快速止损方案
	a.切流下线，通过目前使用的服务发现组件(Nacos,Consul or Eureka) 将问题机器快速下线。如果判断是变更造成的需要灰度回滚。
	b.机器重启或者手动触发FullGC(跳转扩展阅读->Jcmd)。快速回收内存，减少服务影响。
	c.在变更前保留现场


### 1.3 保留现场：
无论是什么机器如果条件允许不建议直接重启或者回滚，可以先保留现场，需要保存如下内容，优先级依次降低
#### 1.3.1. heapdump文件
[﻿ATP帮助文档-生成Java转储文件﻿](https://help.aliyun.com/zh/atp/getting-started/preparations)
生成转储命令
```shell
#jmap命令保存整个Java堆（在你dump的时间不是事故发生点的时候尤其推荐）
jmap -dump:format=b,file=heap.bin <pid> 

#jmap命令只保存Java堆中的存活对象, 包含live选项，会在堆转储前执行一次Full GC
jmap -dump:live,format=b,file=heap.bin <pid>

#jcmd命令保存整个Java堆,Jdk1.7后有效
jcmd <pid> GC.heap_dump filename=heap.bin

#在出现OutOfMemoryError的时候JVM自动生成（推荐）节点剩余内存不足heapdump会生成失败
-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/tmp/heap.bin

#编程的方式生成
使用HotSpotDiagnosticMXBean.dumpHeap()方法

#在出现Full GC前后JVM自动生成，本地快速调试可用
-XX:+HeapDumpBeforeFullGC或 -XX:+HeapDumpAfterFullGC
```
PS: 如果客户侧应用接入ARMS,可以直接用控制界面的dump功能：
﻿[如何创建内存快照_应用实时监控服务(ARMS)-阿里云帮助中心](https://help.aliyun.com/zh/arms/application-monitoring/user-guide/memory-snapshot)


#### 1.3.2 当前JVM的启动参数
获取启动参数
```shell
ps -ef | grep java
```

#### 1.3.3 GC日志
Java GC日志需要在应用启动时设置GC日志打印相关的JVM参数来开启，以下是推荐的参数设置，仅供参考：
```shell
# Java8及以下
-XX:+PrintGCDetails -XX:+PrintGCDateStamps -Xloggc:<path>

# Java9及以上
-Xlog:gc*:<path>:time
```

#### 1.3.4 内存栈
```shell
jstack <pid> > jstack.log #jstack生成，推荐
jcmd <pid> Thread.print > jstack.log #jcmd生成
```

#### 1.3.5 Linux 日志
如果你观察到现象的时候你的JVM进程已经消失了，有概率是被linux oom_killer kill掉了，可以通过以下命令获得相关日志：
```shell
dmesg -T | grep -i kill
```

#### 1.3.6 JAVA日志(有具体的OOM信息最好)
很多时候JVM内存溢出是会打出日志信息。如果有比较明确的日志信息也可以帮助我们快速定位。
比如元空间溢出：java.lang.OutOfMemoryError: Metaspace；
直接内存溢出netty会报OutOfDirectMemoryError: failed to allocate capacity byte(s) of direct memory (used: usedMemory , max: DIRECT_MEMORY_LIMIT )
```shell
#/path/to/your/logfile 替换为 日志地址
grep "java.lang.OutOfMemoryError" /path/to/your/logfile

#如果日志文件是压缩文件（如.log.gz），你可以使用zgrep命令
zgrep "java.lang.OutOfMemoryError" /path/to/your/logfile.gz

#如果是多个文件的话在日志地址上增加通配符
grep "java.lang.OutOfMemoryError" /path/to/your/logs/*.log
```


## 二、判断 JVM 内存问题来源

 确认到底是哪个进程的内存问题

判断是否是JVM内存泄漏：内存占用缓慢增加一定是内存泄漏吗？

分析日志

根据现象初步判断问题所在内存区域

## 三、堆内问题排查
这里给出具体的步骤和树状图。优先使用命令，具体工具因为线上环境限制无法使用的场景，可以使用命令从上到下排查。具体内容详解会链接到正文后面扩展阅读中：
工具
﻿ATP GC分析：分析GC, 同类功能有EasyGC.
﻿ATP 堆分析: 类似MAT的在线平台：内部工具grace产品化, 可以快速使用。
MAT 堆分析：最推荐的堆内存分析工具,可以通过内存快照, 有一定的学习成本，推荐花点时间完整学一遍非常有用。ps: 扩展阅读-》常用第三方工具部分 会列出常用场景和OQL，帮助快速使用
命令，从上到下排查，从整体到局部，从JVM到系统本身
jstat -gcutil \<pid> ：获取JAVA堆，元空间，gc信息，详情参见扩展阅读示例
jmap -heap \<pid> ：Java进程的堆内存详情，看在线情况
jmap -histo \<pid> ： 生成堆中的对象直方图：快速识别哪些类的实例占用了大量的堆内存
arthas的memory命令 ： 查看堆和对外具体信息，详情参见扩展阅读部分Arthas
pmap -x \<pid> | sort -nrk3 | less ： 获取大内存块特别是堆，栈，代码段具体位置和布局


## 四、堆外问题排查


### 4.1 元空间
在JVM启动后或者某一特定时间点，MetaSpace的使用量持续增加，并且每次的垃圾回收（GC）都无法释放这部分空间。即使增大MetaSpace的容量，也无法根本解决这个问题。
简要思路：定位具体类位置
元空间是存储类元数据的位置，元空间问题的排查方式就是去trace类加载，下面是相关命令：
查看类加载情况
```shell
#显示指定进程的类加载器相关的统计信息
Jmap -clstats <pid> :

#监视类加载器的行为，包括加载、卸载的类的数量以及相关的内存消耗。
Jstat -class <pid>

#统计在JVM的类加载中，每一个类的实例数量，并按照数量降序排列。
jcmd <PID> GC.class_stats|awk '{print$13}'|sed  's/\(.*\)\.\(.*\)/\1/g'|sort |uniq -c|sort -nrk1

#arthas的classloader命令玩法比较多，有一定学习成本
arthas的classloader命令
```

追踪类加载，卸载情况：
在调试环境中添加VM参数(在生产环境请谨慎！！！)
-verbose:class 用于同时跟踪类的加载和卸载
-XX:+TraceClassLoading 单独跟踪类的加载
-XX:+TraceClassUnloading 单独跟踪类的卸载
简要思路：关注点，具体见-》案例部分
1.关注 fastjson, beanCopy, Orika, Groovy, 反射，CGLIB 动态代理
2.是否有设置-XX:MaxMetaspaceSize 元空间最大值
### 4.2 DirectMemory和JNI Memory
#### 4.2.1 异常现象：
top发现JAVA实际占用的RES 甚至超过了 -Xmx 的大小，内存使用率不断上升，甚至开始使用 SWAP 内存，同时可能出现 GC 时间飙升，线程被 Block 等现象

#### 4.2.2 排查策略
判断堆外内存泄漏是否和direct memory强相关
使用 NMT（NativeMemoryTracking） 进行分析。在项目中添加 -XX:NativeMemoryTracking=detail JVM参数后重启项目（需要注意的是，打开 NMT 会带来 5%~10% 的性能损耗）。
使用命令 jcmd pid VM.native_memory detail 查看内存分布。重点观察 total 中的 committed，因为 jcmd 命令显示的内存包含堆内内存、Code 区域、通过 Unsafe.allocateMemory 和 DirectByteBuffer 申请的内存，但是不包含其他 Native Code（C 代码）申请的堆外内存。如果 total 中的 committed 和 top 中的 RES 相差不大，则应为主动申请的Direct Memory未释放造成的。

#### 4.2.3 DirectMemory关注点（详见 -》案例部分）
-XX:MaxDirectMemorySize 配置
如果大量DirectMemory没有释放，关注-XX:DisableExplicitGC 配置
通过反射监控NIO 和 Netty 的计数器字段
使用Dio.netty.leakDetectionLevel的netty自带排查功能
﻿
#### 4.2.4 JNI Memory关注点（详见 -》案例部分）
JNI Memory的内存是因为JVM内存调用了Native方法，即C、C++方法，所以需要使用C、C++的思路去解决。C和C++的问题排查也是非常复杂的值得用一个专题去介绍。这里只是简单的结合网上的案例提供两个比较粗糙的方案。
方向1：
1.gpertools分析谁没有释放内存：定位C、C++的函数
2.确认C、C++的函数对应的Java 方法
3.jstack或arthas的stack命令：Java方法对应的调用栈
方向2：
1.pmap定位内存块的分布：查看哪些内存块的Rss、Swap占用大
2.dump出内存块，打印出内存数据：把内存中的数据，打印成字符串，分析是什么数据
方向2具体操作参见pmap指令部分和扩展阅读-》案例部分


### 4.3 栈问题
里粗略讨论两种栈溢出报错：StackoverflowError和OOM：unable to create new native thread。两者都可以粗略归类为栈相关的内存问题，StackoverflowError是当前线程的 JVM 栈帧空间耗尽报错；unable to create new native thread一般为1.内存中耗尽无法为新线程分配空间；2.系统层面线程数超过了限制。再次强调，栈的内存报错最快的定位方式就是在有应用日志的情况下查看日志相关异常关键字相关上下文。

Stackoverflow ：
问题定位：
1.程序直接抛出StackOverflowError异常：直接可以检查 Java 调用栈看是哪个方法触发了溢出。注意，JVM可能不会完全打印所有栈帧，因为栈帧输出数量默认限制为1024（XX:MaxJavaStackTraceDepth=1024）。若需完整栈信息，将此参数设为-1。
2.分析Crash日志： 如果进程崩溃后留下了Crash日志，查看日志中"Current thread"的栈范围和RSP寄存器的值。如果RSP值超出了栈范围，说明是栈溢出导致崩溃。
3.利用核心转储（core dump）分析： 如果没有Crash日志，你需要依赖核心转储文件。在程序运行前设置ulimit -c unlimited来允许核心转储。进程崩溃时会生成core.\<pid>文件，使用jstack $JAVA_HOME/bin/java core. \<pid>来分析栈信息。检查是否有异常长的调用链。注意，使用jstack提取信息可能受到serviceability agent(SA)的bug影响。

根因分析：具体参见[云栖社区-StackOverFlowError 常见原因及解决方法](https://developer.aliyun.com/article/713353)﻿

总结一下：

| 引发 StackOverFlowError 原因                               | 解决方法                                  |
| ------------------------------------------------------ | ------------------------------------- |
| 无限递归循环调用(eg.类之间的循环依赖)                                  | 修bug，使用异常堆栈追踪重复的代码行                   |
| 执行了大量方法导致线程栈空间耗尽                                       | 通过 -Xss 参数增加线程栈内存空间，如 -Xss2m          |
| 方法内声明了海量的局部变量                                          | 通过 -Xss 参数增加线程栈内存空间以容纳更多局部变量          |
| native 代码在栈上分配了较大内存，如 java.net.SocketInputStream.read0 | 检查和优化 native 代码，或通过 -Xss 参数为线程栈分配更多空间 |

OutOfMemoryError: unable to create new native thread
报错原因为1.内存中耗尽无法为新线程分配空间；2.系统层面线程数超过了限制 。解决方法无非是从应用和系统两个层面上“开源节流“
1.应用层面上分析线程，判断是否创建了过多的线程，谁创建的这些线程。线程的变化趋势也可以通过ARMS的监控监控到，线程相关分析可以使用[ATP-线程分析功能](https://help.aliyun.com/zh/atp/getting-started/java-thread-stack-analysis-quick-start)。
2.线程数触限：系统层面上对线程数有相关限制，可以通过ulimit -u 查看，可以适当调大系统线程数。
3.内存耗尽：财大气粗就升配，没有必要就评估一下减少-xss(会有stackflowerror风险)

## 五、常见案例

### 5.1 JNI Memory溢出
#### 5.1.1 Linux使用默认ptmalloc2内存分配器在高并发分配内存时，存在较多内存碎片无法释放
关键词：Linux经典64MB问题，ptmalloc2，glibc，malloc
解决方法：
a.治标：调用malloc_trim手动释放内存
b.治本：更换ptmalloc2为jemalloc或者tcmalloc(两者各有优劣，慎重变更)
关联问题：GZip问题，stream流问题
关联文章：
﻿[【JVM案例篇】堆外内存(JNI Memory)泄漏(Linux经典64M内存块问题) - 掘金](https://juejin.cn/post/7255634554987020343)﻿
一句话描述：文章描述了完整JNI 内存问题的定位和排查思路，推荐
﻿[一次 Java 进程 OOM 的排查分析（glibc 篇） - 掘金](https://juejin.cn/post/6854573220733911048)﻿
一句话描述：相较于前文注重流程化排查，本文更深入原理


#### 5.1.2 在Java中流对象(FileInputStream)、网络连接对象(Socket)一般都关联了原生资源，对象未关闭导致native内存无法释放

关键词：Java.util.zip.Inflater, Gzip, hibernate
解决方法：定位到代码段，升级落后版本依赖，关闭Stream，难点主要在定位和排查。
关联文章：
﻿[一次大量 JVM Native 内存泄露的排查分析（64M 问题） - 掘金](https://juejin.cn/post/7078624931826794503)﻿
一句话描述：hibernate源码中的流没有关闭导致泄漏，排查逻辑值得学习
﻿[一次Java内存占用高的排查案例，解释了我对内存问题的所有疑问](https://zhuanlan.zhihu.com/p/652545321)﻿
一句话描述：Gzip流未关闭问题，作者使用pmap快速定位到了问题代码，值得一提的是：kafka的consumer、provider端处理gzip压缩算法时，都是可能出现JNI Memory内存泄露问题。如果将kafka消息的压缩算法gzip改为其他算法，例如Snappy、LZ4，这些压缩算法可以规避掉JVM的gzip解、压缩使用JNI Memory的问题。


### 5.2 元空间泄露
关键词：fastjson, beanCopy, Orika, Groovy, 反射，CGLIB 动态代理

如前文JVM内存部分元空间部分所描述的，元空间的问题原理比较复杂，但是问题定位比较简单，无论是使用阿里云ATP的GC分析日志，还是直接在线上的监控直接观察到元空间的异常变动。都可以快速定位到元空间问题。经常会出问题的几个点有 Orika 的 classMap、JSON 的 ASMSerializer、Groovy 动态加载类，bean copy等，基本都集中在反射、Javasisit 字节码增强、CGLIB 动态代理、OSGi 自定义类加载器等的技术点上。
题外话：这类问题比较容易发生的场景是老旧系统从java 1.7升级1.8的时候，没有设置MaxMetaspaceSize参数，导致元空间没有限制打爆了pod，所以才暴露出来问题。
关联文章：
#### 5.2.1 FastJson 元空间泄漏相关文章，这类文章在内部和外部都出现过挺多次了，属于比较容易踩的一个坑了：
﻿[一次完整的JVM堆外内存泄漏故障排查记录｜ Java Debug 笔记 - 掘金](https://juejin.cn/post/6865231944390148103)﻿
﻿[记录一次MetaSpace OOM问题排查历程-腾讯云开发者社区-腾讯云](https://cloud.tencent.com/developer/article/2001945)﻿
#### 5.2.2 Groovy 导致的元空间泄漏
﻿[大量类加载器创建导致诡异FullGC | HeapDump性能社区](https://heapdump.cn/article/1924890)﻿
一句话总结：MAT使用的实践，值得一看

#### 5.2.3 beanCopy导致的元空间泄漏
﻿[记一次Bean Copy导致Metaspace OOM - 掘金](https://juejin.cn/post/7295626801287610408)﻿
﻿[又一个beanCopy引发的血案metaspace溢出_beanutil.copyproperties metaspace oomm-CSDN博客](https://blog.csdn.net/a807719447/article/details/123583849)﻿
﻿[为什么阿里代码规约要求避免使用 Apache BeanUtils 进行属性的拷贝 - UCloud云社区](https://www.ucloud.cn/yun/74742.html)﻿
一句话总结：三篇实际内容相似，如果大量使用底层基于反射的apache的BeanUtils和spring的BeanUtils会创建大量的DelegatingClassLoader。导致元空间溢出。推荐使用cglib方式的beancopy

### 5.3 DirectMemory溢出
关键词：Netty, 分布式，RPC
直接内存宕机问题。常见于NIO、Netty等相关组件和使用了这些组件的相关RPC框架。直接内存泄漏的判断不难，基本上会有如下步骤：
1.top发现JAVA实际占用的RES 甚至超过了 -Xmx 的大小，内存使用率不断上升，甚至开始使用 SWAP 内存，同时可能出现 GC 时间飙升，线程被 Block 等现象
2.初步判断堆外内存泄漏是否和direct memory强相关。这里可以使用 NMT（[NativeMemoryTracking](https://docs.oracle.com/javase/8/docs/technotes/guides/troubleshoot/tooldescr007.html)） 进行分析。在项目中添加 -XX:NativeMemoryTracking=detail JVM参数后重启项目（需要注意的是，打开 NMT 会带来 5%~10% 的性能损耗）。使用命令 jcmd pid VM.native_memory detail 查看内存分布。重点观察 total 中的 committed，因为 jcmd 命令显示的内存包含堆内内存、Code 区域、通过 Unsafe.allocateMemory 和 DirectByteBuffer 申请的内存，但是不包含其他 Native Code（C 代码）申请的堆外内存。如果 total 中的 committed 和 top 中的 RES 相差不大，则应为主动申请的Direct Memory未释放造成的，

判断是Direct Memory是比较容易的，锁定具体代码堆栈位置有一定的工作量：
- [Netty堆外内存泄露排查盛宴](https://tech.meituan.com/2018/10/18/netty-direct-memory-screening.html)﻿
一句话总结：美团技术团队的文章，涉及到Idea debug的具体流程，通过反射监控Netty 中 io.netty.util.internal.PlatformDependent#DIRECT_MEMORY_COUNTER。有参考性
- [长连接Netty服务内存泄漏，看我如何一步步捉“虫”解决-腾讯云开发者社区-腾讯云](https://cloud.tencent.com/developer/article/2315550?areaId=106001)﻿
一句话总结：Netty泄漏，文章更新，使用Dio.netty.leakDetectionLevel的netty自带排查功能 JVM参数可以参考。
- [java 堆外内存监控_mob64ca12ddcacc的技术博客_51CTO博客](https://blog.51cto.com/u_16213354/7186633)﻿
一句话总结：通过反射监控NIO 中的 java.nio.Bits#totalCapacity

### 5.4 栈溢出

关键词：递归，调用栈过深，
StackoverflowError 更多的涉及到具体框架的回调和循环调用逻辑, 包括bug或者漏洞。很多组件的github issue中都会提到。比如比较典型的log4j-over-slf4j.jar 和 slf4j-reload4j 在同一个classpath下会递归调用
- [Detected both log4j-over-slf4j.jar AND slf4j-reload4j on the class path, preempting StackOverflowError.](https://www.slf4j.org/codes.html#log4jDelegationLoop)﻿
排查推荐[云栖社区-StackOverFlowError 常见原因及解决方法](https://developer.aliyun.com/article/713353) 非常清晰！
- [Troubleshoot OutOfMemoryError: Unable to Create New Native Thread](https://dzone.com/articles/troubleshoot-outofmemoryerror-unable-to-create-new)﻿
一句话总结：应该是全网最清晰的Unable to Create New Native Thread 的排查文章了，感觉看这篇就够了

### 5.5 OOM-Killer
跳出JVM内存管理后，当实例全局内存或实例内cgroup的内存不足时，Linux 会选择内存占用最多，优先级最低或者最不重要的进程杀死。一般在容器里，主要的进程就是肯定是我们的 JVM ，一旦内存满，第一个杀的就是它，而且还是 kill -TERM (-9)信号，打你一个猝不及防。
1.如果 生产环境容器只有JVM一个进程在跑，JVM 内存参数配置合理，远低于容器内存限制,，还是出现了 OOM Killer 的话，那么恭喜你，大概率是有什么 Native 内存泄漏，请跳转上文排查 JNI Memory 溢出。
2.如果生产环境中有多个进程在跑，比如有JAVA进程，监控日志进程（Filebeat），定时调度脚本等，手动运维变更动作。。。那就需要具体看日志看到底是谁触发oom,oom时的内存分布是什么了
推荐阅读阿里云官网的出现OOM Killer的原因与解决方案 和 为什么应用运行时进程突然消失了？﻿
日志搜索命令一般为
```shell
grep -i 'killed process' /var/log/messages
或者
egrep "oom-killer|total-vm" /var/log/messages
```
日志解读主要关注三段：谁触发了oom-killer； 最后杀死了谁；当时进程的相关信息
- [解读OOM killer机制输出的日志](https://blog.csdn.net/weixin_39247141/article/details/126304526)

## 六、常用排查指令

### 6.1 jcmd
从JDK7开始，jdk提供了一个方便扩展的诊断命令jcmd，用来取代之前比较分散的jdk基础命令，如jps、jstack、jmap、jinfo等，并且jdk添加新的诊断功能，也会通过jcmd提供。

### 6.2 jhat
jhat是用来分析jmap生成dump文件的命令，jhat内置了应用服务器，可以通过网页查看dump文件分析结果，jhat一般是用在离线分析上。
命令格式 : jhat [option][dumpfile]
option参数解释：
-stack false: 关闭对象分配调用堆栈的跟踪
-refs false: 关闭对象引用的跟踪
-port : HTTP服务器端口，默认是7000 -debug : debug级别
-version 分析报告版本

### 6.3 jps
Java版的ps命令，查看java进程及其相关的信息。
命令格式：jps [options] [hostid]
options参数解释：
-l : 显示进程id,显示主类全名或jar路径
-q : 显示进程id
-m : 显示进程id, 显示JVM启动时传递给main()的参数
-v : 显示进程id,显示JVM启动时显示指定的JVM参数
hostid : 主机或其他服务器ip

### 6.4 jinfo
常用指令之一，主要用来查看JVM参数和动态修改部分JVM参数的命令。
命令格式：jinfo [options] \<pid>
options参数解释：
no options 输出所有的系统属性和参数
-flag 打印指定名称的参数
-flag [+|-] 打开或关闭参数
-flag = 设置参数
-flags 打印所有参数
-sysprops 打印系统配置

### 6.5 jstat
常用指令之一，主要用来查看JVM运行时的状态信息，包括内存状态、垃圾回收等。
命令格式：jstat [option] VMID [interval] [count ]
VMID是进程id，interval是打印间隔时间（毫秒），count是打印次数（默认一直打印）。
option参数解释：
-class class loader的行为统计
-compiler HotSpt JIT即时编译器行为统计
-gc 垃圾回收堆的行为统计
-gccapacity 各个垃圾回收代容量(young,old,perm)和他们相应的空间统计
-gcutil 垃圾回收统计概述
-gccause 垃圾收集统计概述（同-gcutil），附加最近两次垃圾回收事件的原因
-gcnew 新生代行为统计
-gcnewcapacity 新生代与其相应的内存空间的统计
-gcold 年老代和永生代行为统计
-gcoldcapacity 年老代行为统计
-printcompilation HotSpot编译方法统计
查看GC堆容量使用情况 jstat -gc 2708 200 10

### 6.6 jstack
查看JVM线程快照的命令，线程快照是当前JVM线程正在执行的方法堆栈集合，定位线程出现长时间卡顿的原因，例如死锁，死循环等。
命令格式：jstack [options]
option参数解释：
-F 当使用jstack 无响应时，强制输出线程堆栈。
-m 同时输出java堆栈和c/c++堆栈信息(混合模式)
-l 除了输出堆栈信息外,还显示关于锁的附加信息，如死锁。
场景一：cpu占用过高问题排查
1）使用Process Explorer工具找到cpu占用率较高的线程
2）在thread卡中找到cpu占用高的线程id
3）线程id转换成16进制
4）使用jstack -l 查看进程的线程快照 根据16进制id找到对应线程
5）线程快照中找到指定线程,并分析代码

### 6.7 jmap
jmap可以生成 java 程序的 dump 文件， 也可以查看堆内对象示例的统计信息、查看 ClassLoader 的信息以及 finalizer 队列
命令格式：jmap [option] (连接正在执行的进程) 不加参数默认打印所有
option参数解释：
-heap 打印java heap摘要
-histo[:live] 打印堆中的java对象统计信息
-clstats 打印类加载器统计信息
-finalizerinfo 打印在f-queue中等待执行finalizer方法的对象
-dump: 生成java堆的dump文件

## 七、常用第三方工具库

### 7.1 ATP
作为阿里云开箱即用的内存分析产品强力推荐！！
- [什么是应用诊断分析平台ATP_应用问题诊断分析平台(ATP)-阿里云帮助中心](https://help.aliyun.com/zh/atp/user-guide/overview-of-analysis-views-1)﻿
ATP 的GC日志分析可以基于时间轴给出相关的建议。比如之前元空间泄漏的问题，会通过日志给出建议。
ATP本身的功能在官方的帮助文档里有对应的说明，非常清楚了。这里列一下在时机使用中遇到的注意点：
1.使用ATP Java堆分析-》分析垃圾对象模式 有点bug，解析的时候会卡死，猜测是和后端用的机器内存不够。还是推荐使用MAT。
2.之前ATP不支持基于OQL的分析，但是前两天又用ATP的时候发现新增了好多功能，也支持了OQL, 新增了好多新的功能包括ByteBuffer，JVM等，非常非常牛批！

### 7.2 ARMS
阿里云 APM 产品在前文多次提及。一个精确且全面的监控产品可以帮助第一时间定位问题。作为ARMS的用户，客观来说，功能性和易用程度比Skywalking真的好很多。多提一句，ARMS也可以接入非阿里云上部署的服务，整个售后体系也非常完善。

### 7.3 MAT
MAT 作为最常用最流行的内存分析工具，网上的入门文章非常多，下面三篇是我觉得中文社区中相对比较全面的文档。这里补充记录我使用时的一些场景。
- [JVM故障分析及性能优化系列之六：JVM Heap Dump（堆转储文件）的生成和MAT的使用 | HeapDump性能社区](https://heapdump.cn/article/2836272)﻿
- [MAT从入门到精通](https://zhuanlan.zhihu.com/p/57347496)﻿
- [JVM 内存分析工具 MAT 的深度讲解与实践——入门篇](https://juejin.cn/post/6908665391136899079)(推荐！)
场景一:老年代缓慢增加，MAT查看老年代对象
JVM 频繁full gc却无法清除老年代，当Full GC也无法回收足够的内存空间，JVM将抛出OutOfMemoryError错误，到了这步问题转变为分析java 进程中的老年代实例有哪些，是什么线程或者类实例长期在old generation而无法GC。在这里就可以用到之前转码的heapdump文件。第一步是通过vjmap获取老年代的地址范围，然后MAT OQL工具在MAT 界面找到老年代对象。
场景二: 后端服务定期出现长时间young GC,MAT需分析Unreachable对象
JVM定期或者突发性大量GC, MAT分析却抓不到对象。这里有个比较tricky的点是，这个场景并不是有内存泄漏, 而是有可回收对象大量创建才触发GC导致服务受损。这时候需要分析Unreachable对象。
1.jmap命令需要保存整个Java堆：jmap -dump:format=b,file=heap.bin \<pid>
2.MAT默认是只分析reachable对象实例，所以在"Preferences=>Memory Analyzer"中勾选"Keep Unreachable Objects"，删除索引文件Dump同路径下的所有".index"，即可看到所有的对象
3.对于可疑对象可以用OQL查询对象，然后

### 7.4 Jprofile
Jprofile的教程参考官方文档[https://www.ej-technologies.com/resources/jprofiler/help_zh_CN/doc/main/profiling.html](https://www.ej-technologies.com/resources/jprofiler/help_zh_CN/doc/main/profiling.html)

### 7.5 gperftools
主要用于排查堆外问题
要找到内存问题，要使用google的`gperftools`，我们主要用到它的 Heap Profiler，功能很强大。[https://github.com/gperftools/gperftools](https://github.com/gperftools/gperftools)
它的启动方式有点特别，安装成功之后，你只需要输出两个环境变量即可。


