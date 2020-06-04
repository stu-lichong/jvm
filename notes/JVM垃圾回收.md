# JVM垃圾回收

## 垃圾判断算法

1. 引用计数算法(Reference Counting)
    * 思想: 给对象添加一个引用计数器, 当有一个地方引用它, 计数器加1, 当引用失效, 计数器减1, 任何时刻计数器为0的对象就是不可能再被使用的.  
    * 缺点: 无法解决对象循环引用的问题. 
2. 根搜索算法(GC Roots Tracing)
    * 思想: 通过一系列被称为"GC Roots"的对象作为起始进行向下搜索, 当一个对象到GC Roots没有任何引用链(Reference Chain)相连, 则证明此对象是不可用的.  
    * 在Java语言中GC Roots主要包括:
        * VM栈中的引用的对象
        * 方法区中的静态属性和常量引用的对象
        * 本地方法栈中引用的对象
        * JVM内部的引用, 如基本数据类型对应的Class对象, 一些异常类对象, 类加载器对象等.

## 方法区的垃圾回收---主要回收废弃常量和无用类
1. 类回收需要满足如下3个条件
    * 该类所有的实例都已经被GC, 即JVM中不存在该Class的任何实例
    * 加载该类的ClassLoader已经被GC(类和加载它的类加载器是双向引用的)
    * 该类对应的java.lang.Class对象没有在任何地方被引用
2. 在大量使用反射, 动态代理, cglib, 动态生成jsp以及OSGi这类频繁自定义类加载器的场景, 都需要JVM具备类卸载的支持, 以保证方法区不会溢出.

## 垃圾收集名词
* Minor GC(Young GC) : 垃圾收集目标是新生代.
* Major GC(Old GC) : 垃圾收集目标只是老年代, 目前只有CMS会有单独收集老年代的行为.
* Mixed GC : 混合搜集, 收集整个新生代以及老年代, 目前只有G1会有这种行为
* Full GC : 对整个Java堆和方法区的垃圾收集

## JVM常见GC算法
1. 标记-清除算法(Mark-Sweep)
    > 首先标记出所有需要回收的对象, 然后再回收所有需要回收的对象  
    > 效率问题, 需要扫描所有对象, 标记和清除效率都不高; 空间问题, 标记清理后产生大量不连续内存碎片, GC次数越多, 碎片越严重.

2. 标记-整理算法(Mark-Compact)
    > 先标记, 然后令所有存活的对象一端移动, 再直接清理掉这端边界以外的内存.  
    > 没有内存碎片, 但需要耗费更多的时间进行compact  

3. 复制算法(Copying)
    > 将可用内存分为两块, 每次只使用其中的一块, 当半区内存用完了, 仅将存活的对象复制到另外一块上面, 然后将原半区内存一次性清理掉  
    > 无内存碎片, 实现简单, 运行高效; 内存缩小为原来的一半, 代价高昂.  
    > 适合生命周期比较短的对象, 这样每次GC都能回收大部分对象, 复制开销比较小.  
    > 商业虚拟机中采用这种算法来回收*新生代*: 将内存分为一块较大的eden空间和两块较小的survivor(From/To Survivor)空间.  
---
4. 分代收集算法(Generational Collecting) --- 商业虚拟机的垃圾收集都采用这种算法
    > 一般将堆分作*新生代*和*老年代*, 根据各个年代的特定采用最合适的收集算法, 如新生代采用复制收集算法, 只需要付出少量存活对象的复制成本就可以完成收集.  
    > 新生代(Young Generation), 老年代(Old Generation, Tenured)

## 垃圾回收器(Garbage Collector) -- GC的具体实现
> Hotspot JVM提供多种垃圾回收器, 我们需要根据具体应用的需求采用不同的回收器.  
* 垃圾收集器中的并行和并发
    * 并行(Parallel): 指多个收集器的线程同时工作, 但是用户线程处于等待状态.
    * 并发(Concurrent): 指收集器在工作的同时, 可以允许用户线程工作. 但是在GC的关键步骤上用户线程还是要停顿, 如在标记垃圾的时候. 
1. Serial收集器
    > 单线程收集器, 收集时会暂停所有工作线程(Stop The World, STW).    
    > 采用复制算法, 是Hotspot允许在Client模式的默认新生代收集器.  
    > 没有线程切换的额外开销, 简单实用.  
2. ParNew收集器 --- Serial的多线程版本
    > 虚拟机运行在Server模式的默认新生代收集器, 在单CPU环境中, ParNew并不会比Serial收集器效果好.  
    > -XX:ParallelGCThreads : 控制GC线程数的多少, 需结合具体的CPU核心数.
3. Parallel Scavenge(PS)收集器
    > 使用复制算法的多线程收集器, 以吞吐量最大化(GC时间占总运行时间最小)为目标, 允许较长时间的STW换取总吞吐量最大, 适合在后台运算而不需要太多交互的分析任务.
---

4. Serial Old收集器
    > 单线程收集器, 采用标记-整理算法, 是老年代的收集器.   
5. Parallel Old收集器
    > 老年代版本吞吐量优先收集器, PS收集器的老年代版本, 使用多线程和标记整理算法.
    
6. CMS(Concurrent Mark Sweep)收集器---并发收集器
    > 以*最短停顿时间*为目标的老年代收集器, 但是并不能达到总GC时间最短, 使用标记-清除算法, 搭配新生代收集器ParNew使用.  
    > 使用-XX:+UseConcMarkSweepGC打开

---

-XX:+UseSerialGC    : 使用Serial + Serial Old的收集器组合
-XX:+UseParNewGC    : 使用ParNew + Serial Old收集器
-XX:+UseConcMarkSweepGC     : 使用ParNew + CMS + Serial Old收集器, Serial Old是Concurrent Mode Failure失败后的后备收集器
-XX:+UseParallelGC  : 使用Parallel Scavenge + Serial Old收集器
-XX:+UseParallelOldGC   : 使用Parallel Scavenge + Parallel Old收集器

---

## JVM GC相关参数

-verbose:gc    : 显示虚拟机GC运行的详细信息
-Xms20m  : 堆的初始内存为20m
-Xmx20m  : 堆的最大分配内存20m
-Xmn10m  : 指定新生代内存为10m
-XX:+PrintGCDetails     : 打印GC工作后内存的变化信息
-XX:SurvivorRatio=8     : 指定新生代中Eden, From Survivor, To Survivor空间比为8:1:1

java -XX:+PrintCommandLineFlags -version    : 打印JVM启动参数和JDK版本信息

-XX:PretenureSizeThreshold=1024 : 设置对象超过1024字节时直接在老年代分配内存
-XX:+UseSerialGC  :  设置采用串行收集器对新生代和老年代进行GC
-XX:+UseParallelOldGC : 设置采用并行收集器对新生代和老年代进行GC

-XX:MaxTenuringThreshold=5  : 在可以自动调节对象晋升(Promote)到老年代阈值的GC中, 设置该阈值的最大值. 该参数的默认值为15, CMS中默认值为6, G1中默认为15(该数值由4个bit表示, 所以最大值为15).
> 经历多次GC后, 存活的对象会在From Survivor和To Survivor之间来回存放, GC算法会计算每个存活对象的年龄, 如果达到某个年龄后发现总大小已经大于Survivor空间的50%, 那么JVM会自动调整阈值, 让这些存活的对象尽快完成晋升.   

-XX:TargetSurvivorRatio=60  : To Survivor空间占用超过60%, 就自动调节对象晋升到老年代的阈值.
-XX:+PrintTenuringDistribution  : 打印不同年龄的对象在To Survivor空间占的内存大小.
-XX:+PrintGCDateStamps  : 打印GC垃圾回收的时间戳
-XX:+UseParNewGC    : 新生代使用ParNew进行GC
-XX:+UseConcMarkSweepGC     : 老年代使用CMS进行GC
-XX:MaxTenuringThreshold=3  : 对象晋升到老年代的年龄阈值的最大值

## CMS收集器(Concurrent Mark Sweep)
基于"标记-清除"算法实现, 以获取*最短GC停顿时间*为目标, 多数应用于B/S系统的服务器端上. 整个过程分为4个步骤:
1. 初始标记(CMS initial mark)  会STW

2. 并发标记(CMS concurrent mark)

3. 重新标记(CMS remark)     会STW

4. 并发清除(CMS concurrent sweep)

CMS通过将大量工作分散到并发处理阶段来减少STW时间. 整个过程中耗时最长的并发标记和并发清除阶段, GC线程都可以与用户线程一起工作. 因此, 总的来看CMS的GC过程是与用户线程一起并发执行的.  

* CMS的缺点
    - 对CPU资源非常敏感. 并发阶段占用了一部分线程, 导致并发设计的应用程序变慢, 降低总吞吐量.  
    - CMS无法处理"浮动垃圾"(Floating Garbage). CMS运行期间用户线程继续运行, 伴随有新的垃圾对象(浮动垃圾)不断产生, 需要下次GC时回收.  
    - 可能出现"Concurrent Mode Failure". CMS运行期间预留的内存无法满足并行的用户程序需要, 出现并发模式失败, 导致另一次Full GC的产生.  
    - 收集结束时产生大量内存碎片. 出现大对象需要分配内存时无法找到足够大的连续空间的问题, 导致不得不提前进行一次Full GC.  

## G1(Garbage First Collector)

* 垃圾收集器追求的目标: 高响应能力, 高吞吐量  
    > 响应能力指一个应用或系统能多快对一个数据请求进行响应; 强调在一个较短的时间内进行响应, 不能容忍长时间的业务线程暂停.  
    > 吞吐量强调在一个较长的时间范围内最大化应用的工作量(总停顿时间最短), 不考虑应用的响应速度, 因此可以容忍一次较长的停顿时间. 

g1收集器是一个面向服务端的垃圾收集器, 适用于多核处理器, 大内存容量的服务端系统. 满足短时间gc停顿的同时达到较高的吞吐量. JDK 7以上版本适用.  

---
1. g1收集器堆结构划分
    * heap被划分为一个个相等的不连续的内存区域(region), 每个region都有一个分代的角色: eden, survivor, old
    * 对每个角色的数量并没有强制的限定, 也就是说每种分代内存的大小可以动态变化
    * G1最大的特定就是高效的执行回收, 优先执行那些大量对象可回收的region
    * G1使用gc停顿可预测的模型, 尽量满足用户设定的gc停顿时间, 根据用户设定的目标时间自动选择哪些region要清除, 一次清除多少个region
    * G1从多个region中复制存活对象, 然后集中放入一个region中, 同时整理 清除内存(复制算法)

---
2. G1对比其他GC
    > G1收集器会完全取代CMS. 对比使用CMS, G1只使用copying算法, 回收内存后会马上进行合并空闲内存工作, 不会造成内存碎片; G1中eden/survivor/old区不再固定, 内存使用上更灵活; G1会在Young GC中使用, 而CMS只能在old区使用; 此外, G1提供可预测的GC暂停时间, 并且允许用户指定期望的暂停时间.      
    > 对比Parallel Scavenge(基于copying的新生代收集器), Parallel Old(基于标记整理的老年代收集器), 会对整个区域做整理导致gc停顿时间较长, 而G1只是特定整理几个region.

---
3. G1重要概念  

    3.1 分区(Region) : G1将整个堆分成相同大小(1MB-32MB)的分区.

* 每个分区都可能是年轻代或老年代, 但是在某个时刻只能属于某个代, 年轻代, 幸存区, 老年代这些概念还在, 成为逻辑上的概念, 方便复用之前分代框架逻辑.
* G1会优先回收垃圾对象特别多的分区, 就可以花费较少的时间来回收这些分区的垃圾, 即首先收集垃圾最多的分区(Garbage First).
* 仍然是在新生代满了时, 对整个新生代进行回收(Young GC). 也可同时对新生代, 老年代进行回收(Mixed GC). 
* G1还是一种带压缩的收集器, 在回收老年代分区时, 是将存活对象从一个分区拷贝到另一个可用分区, 实现了局部压缩.

    3.2 收集集合(CSet): 一组可被回收的分区的集合. CSet中的分区可以来自eden空间, survivor空间, 老年代.

    3.3 记忆集(RSet): G1中RSet记录其他region中的对象引用本region中对象的关系(记录别的region引用我的对象). RSet的价值在于使得垃圾收集器只需要扫描RSet即可找到谁引用了当前分区的对象.  
    > G1是在points-out的Card Table之上再加上一层构成points-into RSet: 每个region会记录下哪些别的region有指向自己的指针, 而这些指针分别在哪些card范围内.  
    > 这个RSet其实是一个Hash Table, key是别的region的起始地址, value是一个集合, 元素是card table的索引, 表示别的region中哪些卡页里有引用指向了本region.  

    3.4 SATB(Snapshot At The Begining), 是G1在*并发标记*阶段使用的增量式标记算法, 并发线程在同一时刻只扫描一个分区.  

---
4. G1的GC模式
    > G1提供两种GC模式: Young GC和Mixed GC, 两种都是完全Stop The World的  
* Young GC : CSet中包含*所有*新生代里的Region. 可通过控制新生代的Region个数, 即新生代内存大小, 来控制Young GC的时间开销.  
    > Young GC在eden充满时触发, 回收之后属于eden的所有region全部变成空白, 不属于任何一个分区(eden, survivor, old)  
    > G1中的RSet使用point-into来解决. 即记录哪些分区引用了当前分区的对象, 这样仅仅将那些分区中的对象当作根来扫描. 由于新生代中的对象每次GC都会被扫描, 就没必要记录新生代region之间的引用, 只需要记录老年代到新生代之间的引用.  

* Mixed GC : CSet中包含*所有*新生代Region, 外加根据global concurrent marking统计得出收集收益高的*若干*老年代Region. 在用户指定的开销目标范围内尽可能选择收益高的老年代Region.  
    > Mixed GC不是Full GC, 他只能回收部分老年代的Region, 如果Mixed GC实在无法跟上程序分配内存的速度, 导致老年代填满, 就会使用Serial Old GC(Full GC)来收集整个GC堆. 所以, 本质上G1是不提供Full GC的.  

* Mixed GC发生时机?  由参数控制      
    > G1HeapWastePercent: 在global concurrent marking结束之后, 我们可以知道老年代Region中有多少空间要被回收, 在每次Young GC之后和再次发生Mixed GC之前, 会检查垃圾占比是否达到此参数, 只有达到了, 下次才会发生Mixed GC.  
    > G1MixedGCLiveThresholdPercent : 老年代region中存活对象的占比只有在低于此参数值, 才会被选入CSet.  
    > G1MixedGCCountTarget : 一次global concurrent marking之后, 最多执行Mixed GC的次数.   
    > G1OldCSetRegionThresholdPercent : 一次Mixed GC中被选入CSet的最多老年代region的数量.  

* global concurrent marking : 执行过程类似于CMS, 主要是在进行Mixed GC前, 为Mixed GC提供标记服务, 并不是一次GC过程的必须环节. 初始标记--STW; 并发标记; 重新标记--STW; 清理(清除空Region加入free list).  

* G1会在Young GC和Mixed GC之间不断地切换运行, 同时定期地做全局并发标记, 在赶不上对象创建速度的情况下使用Full GC.  

* G1其他参数  
    -XX:G1HeapRegionSize=n  : 设置region大小  
    -XX:MaxGCPauseMillis    : 设置G1收集目标时间, 默认200ms  
    -XX:G1NewSizePercent    : 新生代最小值, 默认5%  
    -XX:G1MaxNewSizePercent     : 新生代最大值, 默认60%  

---
5. Humongous区域  
    > 如果一个对象占用的空间达到或是超过region容量50%以上, G1划分 一个Humongous区用来存放巨型对象. 如果一个H区装不下, G1会寻找连续的H分区来存储. 为了找到连续的H区, 有时候不得不启动Full GC.  

---
6. 三色标记算法---并发标记  
    将对象分成三种类型:  
    * 黑色: 根对象, 或者该对象与它的子对象(field)都被标记过.  
    * 灰色: 对象本身被标记, 其所有子对象还未完全被扫描.---标记过程的中间状态    
    * 白色: 未被标记的对象, 扫描完所有对象后, 最终所有可达对象都变成黑色, 白色的为不可达对象, 即垃圾对象.  

- 由于并发标记阶段, 扫描线程与应用线程并发执行, 三色标记算法可能会产生对象丢失问题(漏标, 被GC后导致应用崩溃). 解决方式---G1使用SATB   

- 并发标记发生对象丢失的充分必要条件(按顺序满足以下两个条件): 
    - 赋值器插入了一条或多条从黑色对象到白色对象的新引用
    - 赋值器删除了全部灰色对象到该白色对象的直接或间接引用

---
7. SATB(Snapshot At The Begining)原始快照 --- 破坏第二个条件
    * 在*开始标记*的时候生成一个快照图
    * 在并发标记时所有被改变的对象入队(在write barrier里把所有旧的引用所指向的对象都变为非白的, 即可能是非垃圾对象)
    * Write Barrier 写屏障, 就是对引用字段进行赋值时做了额外处理. 通过Write Barrier就可以了解到哪些引用对象发生了什么样的变化.  
    * 存在的问题: 可能存在浮动垃圾, 将在下次被收集.
    ---
    * SATB 详解
    > 对于三色标记算法在并发标记阶段产生漏标记问题, SATB在并发标记阶段中, 对于从grey对象将要移除的目标引用对象标记为grey, 并发扫描结束后, 再以记录过的grey对象为根, 重新扫描一此. 即无论引用关系是否删除, 都会按照扫描开始时的对象图快照来进行搜索.   
    > 并发标记过程中, 程序继续运行会持续产生新对象, G1为每个region设计了两个名为TAMS(Top at Mark Start)的指针, 将region中一部分空间划分出来用于并发阶段的新对象分配, 这些新对象是被隐式标记的, 即默认它们是存活的.  
    > 漏标与误标: 误标没关系, 顶多造成浮动垃圾, 在下次GC回收; 漏标后果是致命的, 回收了本应该存活的对象, 影响程序正确性.  

---
8. 停顿预测模型  
    G1突出特点是通过一个停顿预测模型根据用户配置的停顿时间来选择CSet的大小, 从而达到用户期待的应用程序暂停时间.     
    * -XX:MaxGCPauseMillis=x参数来设置. 设置的时间越短意味着每次收集的CSet越小, 导致垃圾逐步累积, 最终不得不进行Full GC; 停顿时间过长, 影响程序对外的响应时间. 

---
9. G1最佳实践
    * 不断调优暂停时间, 逐步调整到最佳状态.  
    * 不要设置新生代和老年代的大小. G1收集器会通过改变代的大小来调整对象晋升的速度以及年龄, 从而达到设置的暂停时间目标. 我们只需要设置整个对内存的大小, 剩下的交给G1自己去分配各个代的大小即可.  
    * 关注Evacuation Failure(回收失败). 堆空间垃圾太多导致无法完成Region之间的拷贝, 不得不退化成Full GC.   
