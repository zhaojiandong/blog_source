title: JVM垃圾收集算法
date: 2015-08-09 23:55:08
tags: jvm GC
---

# JVM垃圾收集

## 1. 判断对象是否存活
- 引用计数算法
对象添加一个引用计数器，每个地方引用它，计数器值加+1；当引用失效，计算器值减1；任何时刻计数器为0的对象不可能被使用。引用计数算法实现简单，高效。
缺点：引用计数算法，很难解决相互引用的问题。
``` java
 objA.instance = B;
 objB.instance = C;
```


- **可达性分析算法**
 主流商用算法，通过一些列的"GC roots" 作为对象的起点，从这些节点开始向下搜索，锁走过的路径成为引用链(reference Chain),当一个对象到GC root没有任何引用链项链，则证明对象不可用。
在Java语言中，可以作为回收改对象 GC Roots的对象：
    虚拟机栈中引用的对象
    方法区中类静态属性引用对象
    方法区中常量引用的对象
    本地房发展中JNI(一般说的Native方法)引用对象
- **再谈引用**
  强引用：代码中普遍存在的，object obj = new Object();
  软引用：有用但非必须的对象
  弱引用：非必须的对象，当GC工作室，都会被回收
  虚引用：称为幽灵引用或欢迎引用，最弱的一种引用

------------------------
## 2. 对象生存还是死亡
 **对象判断死亡需要经历两次标记过程：**
1. 如果对象在进行可达性分析后发现没有与GC roots相连的引用链，那它将会第一次标记并进行一个三选，删选的条件是此对象是否有必要执行finalize（）方法。当对象没有覆盖finalize()方法，或者finalize()方法已经被虚拟机调用过，虚拟机将这两种方法视为“没有必要执行”。如果这个对象有必要执行finalize()方法，那么这个对象会被放置在一个叫F-Queue的队列中，并在稍后由一个虚拟机自动建立的，低优先级的Finalizer线程去执行它。这里所谓的“执行”及时虚拟机会触发这个方法，并承诺会等它运行结束。
2.  第二次小规模的标记，如果对象要在finaliz()中成功拯救自己——只要重新与引用链上的任何一个对象建立关联即可，譬如自己(this关键字)复制给某个变量或对象的成员变量，那第二次标记时它将被移除“即将回收”的集合；如果对象这时候没有逃脱，那基本上它就真的要被回收了。

- **回收方法区**
方法区也就是永久代也需要回收，两部分内容：废弃常量和无用的类。
判断常量是否需要回收：Java堆中常量回收没有对象引用这个常量，也没有其他地方引用这个字面量。

判定类是否需要回收:
1. 该类所有的实例都已经被回收，也就是Java堆中不存在该类的任何实例
2. 加载该类的ClassLoader已经被回收
3. 该类对应的java.lang.Class对象没有任何的地方被引用，无法再任何地方通过反射访问该类的方法

-----------------------------------------------------------------------------------------

## 3.垃圾收集算法

- **标记-清除算法(Mark-Sweep)**
 分成两个过程，首先标记出所有需要回收的对象，在标记完成后统一回收所有被标记的对象。缺点是：1效率问题；2空间问题，大量不连续的内存碎片

- **复制算法**：
复制算法可以解决效率问题，标记号需要回收的对象，复制到内存的另一块中去。缺点：空间利用率问题。
现在的商业虚拟机中，都是“朝生夕死”的对象，可以不按照１:１比例来划分内存空间，而将内存分成一块较大的Eden空间，2块较小的Survivor空间。当回收时，copy Eden空间和1块from Survivor空间中存活的对象到另一块to survivor空间中。

- **标记-整理(Mark-Compact)**
标记过程和Mark-Sweep一样，整理是将存活的对象向一端移动，然后直接清理掉端边界以外的内存。

---------------------------------------------------------

### 3.1 分代收集算法
- 新生代：每次收集都有大批对象死去，只有少量存活，复制算法，只需复制少量存活的对象。所以新生代大多数是stop-the-world收集器。
- 老年代：对象存活率高，没有额外空间，必须使用“标记-清理”或“标记-整理”
- Eden:绝大部分对象存在Eden区域。当Java创建非常大的对象时JVM会分配在老年代而非新生代。每次收集完Eden空间总是空的。
- Survivor空间：在垃圾收集过程设有被当做垃圾收集的对象存放该区域，也就是Survivor域的对象至少经历一次回收。Eden和1个Survivor空间中存活的对象被复制到另一个Survivor空间，同时清理掉Eden和Survivor空间。
![Alt text](http://7xlamb.com1.z0.glb.clouddn.com/GC分代)

- 枚举根节点：
JVM停顿下来后，并不需要一个不漏地检查完所有的引用，JVM通过一组OopMap的数据结构来直接得知哪些地方存放着对象引用。
在类加载完成时，JVM把对象在内存中偏移量计算出来，保存在OopMap中。这样GC在扫描引用时，可以直接得知引用关系了。
- 安全点(safepoint)：
在OopMap协助下，JVM可以快速准确地完成GC Root枚举，但一个很现实的问题随之而来：可能导致引用关系变化，或者说OopMap内容变化的指令很多，如果为每一条指令生成对应的OopMap，那将会需要大量的额外空间。实际上，JVM的确没有为没一条指令生成OopMap，只有特定位置记录了这些信息，这些位置成为安全点(sagfePoint)。方法调用、循环跳转、异常跳转等功能的指令才会生成SafePoint。
**如何让所有线程在GC时停顿下来：**
1. 抢先式中断：GC发生时，所有线程全部中断，如果发现有线程中断的地方不在safepoint上，就恢复该线程，让它跑到safepoint上。
2. 主动式中断：当GC需要中断线程时，不要直接对线程操作，仅仅简单的设置一个标志，各线程执行时主动去轮询这个标志，发现中断标志时就主动中断挂起。

- 各种垃圾收集器的图
    ![Alt text](http://7xlamb.com1.z0.glb.clouddn.com/垃圾收集器的图.png)

- **Serial收集器**
 单线程，收集的时候停止用户线程，模式stop-the-world
 ![Alt text](http://7xlamb.com1.z0.glb.clouddn.com/jvmSerial收集器.png)
新生代：采用复制算法，复制eden空间中和from Survivor空间的对象，到to Survivor空间中。
老年代：采用标记整理算法
简单高效，是JVM默认的新生代收集器。

- **ParNew收集器**
 ParNew收集器其实Serial收集器的多线程版本，除了多线程收集及外，stop-the-world模式。多核处理器，可以充分利用CPU资源，但是在单核中比Serial收集器性能差，因为需要线程切换的开销。
![Alt text](http://7xlamb.com1.z0.glb.clouddn.com/jvmParNew收集器.png)



- **Parallel Scavenge 收集器**
 吞吐量优先收集器，Parallel Scavenge提供两个参数精确控制吞吐量，分别是控制最大来及停顿时间的-XX:MaxGCPauseMillis参数，以及直接设置吞吐量大小的 -XX：GCTimeRatio参数。吞吐量=运行用户代码时间/（运行用户代码时间+垃圾收集时间）
![Alt text](http://7xlamb.com1.z0.glb.clouddn.com/jvmParallel%20Scavenge%20收集器.png)


- Serial Old 收集器
Seri Old收集器是Serial收集器的来年版本，它同样是一个多线程收集器，使用“mark-compact”标记-整理算法。可以和Parallel Scavenge收集器搭配使用；还可以作为CMS收集器的后备预案。
 ![Alt text](http://7xlamb.com1.z0.glb.clouddn.com/jvmSerial%20Old%20收集器.png)


- **CMS收集器**
CMS(concurrent-Mark-Sweep) 是一种以获取最短回收停顿时间为目标的收集器。基于标记-清除算法
**CMS回收策略可以设置阀值触发GC，不一定都是Full GC，而 Full GC 需要stop the world**
过程：
初始标记：标记一下GC roots能直接关联的对象，stop-the-world
并发标记：和用户程序线程一起并发执行
重新标记：为了修正并发标记期间因用户程序继续运行而导致标记残生变动那一部分对象，比初始标记时间长，但远比并发标记时间短，认识stop-the-world模式
并发清除：所有不再被应用的对象将从堆里清除掉。
并发重置：收集器做一些收尾的工作，以便下一次GC周期能有一个干净的状态。
![Alt text](http://7xlamb.com1.z0.glb.clouddn.com/jvm收集器运行示意图.png)
优点：CMS并发收集，低停顿
缺点：1 CMS收集器对CPU敏感，并发的收集器对资源都敏感，会占用用户线程数，推荐1/cpu数量
2 CMS收集器无法回收浮动垃圾，即并发清除过程中产生的垃圾在标记过后，CMS无法再当次收集中处理掉他们，只好留待下一次GC再清理
3 基于标记-清除算法意味着会产生大量空间碎片，空间碎片过多，将会给大对象分配带来麻烦。会提前触发一次Full GC。+UseCMSCompactAtFullCollection开关参数，用于在CMS收集器顶不住要进行FullGC时开启内存碎片整理过程
![Alt text](http://7xlamb.com1.z0.glb.clouddn.com/jvmCMS和Serial%20收集器区别.png)
    CMS和Serial 收集器区别



- **G1收集器**
G1(Garbage-first)收集器是当今最前沿的成果，JDK1.7中试用,7u4
![Alt text](http://7xlamb.com1.z0.glb.clouddn.com/jvmG1收集器.png)
**特点：**
1. 并发与并行：
2. 分代收集
3. 空间整合：
4. 可预测的停顿：
G1收集器，Java堆的内存布局就与其他收集器有很大差别，他将整个Java堆划分成多个大小相等的独立区域(region)，虽然还保留新生代和老年代的概念，但新生代和老年代不再是物理上的隔离，它们都是一部分Region(不需要连续)的集合。
G1收集器中，Region之间的对象引用以及其他收集器中新生代与老年代之间对象引用，虚拟机都使用Remembered Set来避免全堆的扫描。G1中每个Region都有一个与之对象的Remembered Set，JVM发现程序对Reference类型的数据进行写操作时，会产生一个Write Barrier暂时中断写操作，检查Reference引用的对象是否处于不同的Region之中，如果是，边通过CardTable把引用信息记录到被引用的对象的所属的region的Remembered Set之中。当进行内存回收时，GC根节点的美剧范围中加入Remembered Set 即可保证部队全堆扫描也不会遗漏。
**G1收集器大致过程：**
1. 初始标记(Initial Marking)：比较GC Roots能直接关联到的对象。
2. 并发标记(Concurrent Marking)：耗时长，和用户线程并发执行
3. 最终标记(Final marking)：为了修正并发标记期间因用户线程继续运作而导致标记产生变化的那部分记录，重新标记
4. 筛选回收(Live Data Counting and Evacution)：回收阶段首先对各个Region的回收价值和成本进行排序，根据用户期望的停顿时间来指定回收计划，这一过程可以和用户并发。

---------------------------------------------

## 4. 关于Full GC

> 当老年代对象大小达到阈值时，便会触发一次Full GC（Full GC一般会伴随一次minor GC），Full GC很消耗内存，把 老年代和新生代 里面大部分垃圾回收掉。这个时候用户线程都会被block住（stop-the-world）。
老年代都会采用 标记-整理-压缩，虽是一个耗时操作，但为了提高老年代空间的利用率，减少Full GC次数，是必要的。




## 5.理解GC日志

开启GC日志：
-XX:+PrintGC 或者 -verbose:gc  开启了简单GC日志模式
-XX:PrintGCDetails，就开启了详细GC日志模式。在这种模式下，日志格式和所使用的GC算法有关
-XX:+PrintGCTimeStamps可以将时间和日期也加到GC日志中。表示自JVM启动至今的时间戳会被添加到每一行中
-XX:+PrintGCDateStamps，每一行就添加上了绝对的日期和时间
-Xloggc:也可以输出到指定的文件。需要注意这个参数隐式的设置了参数-XX:+PrintGC和-XX:+PrintGCTimeStamps，但为了以防在新版本的JVM中有任何变化，我仍建议显示的设置这些参数

GC 表示垃圾收集类型，Full GC需要Stop-the-world
[DefNew [PsYongGen 表示表示收集器区域和收集器名称

``` java
33.125: [GC [DefNew: 3324K->152K(3712K), 0.0025925 secs] 3324K->152K(11904K), 0.0031680 secs]  
100.667: [Full GC [Tenured: 0K->210K(10240K), 0.0149142 secs] 4603K->210K(19456K), [Perm : 2999K->2999K(21248K)], 0.0150007 secs] [Times: user=0.01 sys=0.00, real=0.02 secs]
```

最前面的数字“33.125：”和“100.667：”代表了GC发生的时间，这个数字的含义是从Java虚拟机启动以来经过的秒数。
    GC日志开头的“［GC”和“［Full GC”说明了这次垃圾收集的停顿类型，而不是用来区分新生代GC还是老年代GC的。如果有“Full”，说明这次GC是发生了Stop-The-World的，例如下面这段新生代收集器ParNew的日志也会出现“［Full GC”（这一般是因为出现了分配担保失败之类的问题，所以才导致STW）。如果是调用System.gc()方法所触发的收集，那么在这里将显示［Full GC (System)

``` java
    [Full GC 283.736: [ParNew: 261599K->261599K(261952K), 0.0000288 secs]
```


接下来的“［DefNew”、“［Tenured”、“［Perm”表示GC发生的区域，这里显示的区域名称与使用的GC收集器是密切相关的，例如上面样例所使用的Serial收集器中的新生代名为“Default New Generation”，所以显示的是“［DefNew”。如果是ParNew收集器，新生代名称就会变为“［ParNew”，意为“Parallel New Generation”。如果采用Parallel Scavenge收集器，那它配套的新生代称为“PSYoungGen”，老年代和永久代同理，名称也是由收集器决定的。
    后面方括号内部的“3324K->152K(3712K)”含义是“GC前该内存区域已使用容量-> GC后该内存区域已使用容量 (该内存区域总容量)”。而在方括号之外的“3324K->152K(11904K)”表示“GC前Java堆已使用容量 -> GC后Java堆已使用容量 (Java堆总容量)”。
    再往后，“0.0025925 secs”表示该内存区域GC所占用的时间，单位是秒。有的收集器会给出更具体的时间数据，如“［Times： user=0.01 sys=0.00， real=0.02 secs］”，这里面的user、sys和real与Linux的time命令所输出的时间含义一致，分别代表用户态消耗的CPU时间、内核态消耗的CPU事件和操作从开始到结束所经过的墙钟时间（Wall Clock Time）。CPU时间与墙钟时间的区别是，墙钟时间包括各种非运算的等待耗时，例如等待磁盘I/O、等待线程阻塞，而CPU时间不包括这些耗时，但当系统有多CPU或者多核的话，多线程操作会叠加这些CPU时间，所以读者看到user或sys时间超过real时间是完全正常的。

###  5.1 Full GC触发的4种条件

1. 老年代空间不足： outofmemoryError:java heap space
2. 永久代空间满：存放class信息，当系统通过反射加载的类过多时需要清理 outofmemoryError:permanet generation
3. CMS GC时出现promotion failed和concurrent mode failure可能触发Full GC，survivor空间不足
4. 统计得到Minor GC晋升到老年代的平均大小大于老年代剩余空间

------------------------------------------------

## 6. 内存分配与回收策略
1. 对象优先分配在Eden空间
   当Eden空间不足时，JVM就发起一次MinorGC。
2. 大对象直接进入老年代
 大对象是指需要大量连续空间的java对象，大对象对JVM是坏消息，特别是“朝生夕死”的大对象，-XX：PrintenureSizeThreshold参数，另大于这个设置值得对象直接进入老年代。
3. 长期存活的对象将进入老年代
 Jvm采用分代收集思想 -XX:MaxTenuringThreshold=1，当GC发生1次时对象进行老年代。
4. 动态对象年龄判定
JVM不只依赖MaxTenuringThreshold晋升老年代，如果Survivor空间中相同年龄所有对象总和大于Survivor空间一半，年龄大于或等于该年龄的对象可以直接进入老年代，无需等到MaxTenuringThreshold中要求的年龄。
5. 空间分配担保
在发生MOniorGC之前，JVM 会检查老年代最大可用的连续空间是否大于新生代所有对象总空间，如果这个条件成立，那么MinorGC可以确保是安全的。如果不成立，则JVM会查看HandlePromotionFailure设置值是否允许担保失败。如果允许，会继续检查老年代最大可用的连续空间是否大于历次晋升到老年代对象的对象的平均大小，如果大于，将尝试着进行一次MinorGC，尽管这次MinorGC是有风险的；如果小于,或者HandlePromotionfailure设置不允许冒险，那时也要改为进行一次FullGC。
