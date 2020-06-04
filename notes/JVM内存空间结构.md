# JVM 内存空间结构
1. 虚拟机栈: 栈帧(Stack Frame), 主要存放局部变量表, 操作数栈, 动态链接和方法出口等   线程私有
2. 程序计数器 (Program Counter), 记录当前线程的行号指示器  线程私有
3. 本地方法栈: 主要用于处理本地方法
4. 堆(Heap): JVM管理的最大一块内存空间   所有线程共享
    > 与堆相关的重要概念是垃圾收集器. 几乎所有的GC都采用的分代收集算法, 所以堆空间也基于这点进行了相应的划分: 新生代与老年代. Eden空间, From Survivor空间与To Survivor空间.
5. 方法区(Method Area): JVM规范, 用于存储元信息(常量池, 字段描述, 方法描述). 从JDK 1.8开始已经彻底废除了永久代(Permanent Generation), 使用元空间(Meta Space).    线程共享
---
* 运行时常量池: 方法区的一部分, 存放编译期生成的各种字面量和符号引用, 也可以存储由符号引用解析出的直接引用, 运行期间也可以将新的常量放入池中, 如: String类的intern()方法.  
* 直接内存(Direct Memory), 与Java NIO密切相关, NIO可以使用Native函数库直接分配堆外内存, 通过Java堆上的DirectByteBuffer对象来操作直接内存以提高性能.

### Java对象的创建过程
* new关键字创建对象的3个步骤:
    1. 在堆内存中创建出类的实例
    2. 为对象的实例变量赋初值
    3. 将对象的引用返回

* 对象在内存中的布局: 
    1. 对象头(object header)   包括两类信息
        > markword: 用于存储对象自身的运行时数据, 如哈希码（HashCode） 、 GC分代年龄、 锁状态标志等  8个字节
        > class pointer: 类型指针, 即指向该对象的Class对象;     使用-XX:+UseCompressedClassPointers参数后(默认) 4个字节
        > Java数组的对象头中还必须有一块用于记录数组长度的数据.
    2. 实例数据(instance data)     存储包括从父类继承的字段数据
    3. 对齐(padding)    (可选)  总大小对齐为8字节的倍数

* 堆内存的形式
    * 指针碰撞: 堆中的空间通过一个指针进行分割, 一侧是已经被占用的空间, 另一侧是未被占用空间.
    * 空闲列表: 堆内存中已被使用空间与未被使用空间交织在一起. 虚拟机通过一个列表来记录哪些空间是可以使用的, 哪些空间是已被使用的. 接下来找出可以容纳新创建对象且未被使用的空间存放该对象, 同时修改列表上的记录. 

* 栈空间引用访问对象的方式:
    1. 使用句柄的方式
    > 堆中划分一块内存作为句柄池, 引用指向句柄, 句柄分两个指针, 一个指针指向实例数据, 另一个指针指向类型数据.
    2. 使用直接指针的方式.  
    > 引用直接存储实例数据的地址, 实例数据中存储对应的类型数据地址. 好处是, 少了一个指针定位开销; 缺点是, 对象数据被移动时(GC后如果进行内存重排), 栈中对象的引用也需要同步更新.

### 内存溢出错误
1. 堆内存溢出  java.lang.OutOfMemoryError
    * -Xms5m  : 初始堆内存5M
    * -Xmx5m  : 允许分配的最大堆内存5M
    * -XX:+HeapDumpOnOutOfMemoryError : 将OOM异常转储
2. 虚拟机栈内存溢出  java.lang.StackOverflowError
    * -Xss100k : 设置栈内存大小

### Metaspace 元空间
> 元空间是与堆不相连的本地内存区域, 现在元空间的最大可分配空间就是系统可用内存空间, 用户可以为元空间设置一个可用空间最大值，如果不进行设置，JVM会自动根据类的元数据大小动态增加元空间的容量。

> -XX:MaxMetaspaceSize=20m : 设置元空间最大内存, 默认值是21MB

1. 元空间内存管理
> 元空间的内存管理由元空间虚拟机来完成.  
> 在元空间中，类和其元数据的生命周期和其对应的类加载器是相同的. 每一个类加载器的存储区域都称作一个'元空间', 当一个类加载器被垃圾回收器标记为不再存活，其对应的元空间会被回收。

2. 元空间调优工具

    ‑XX:MinMetaspaceFreeRatio 和‑XX:MaxMetaspaceFreeRatio 可设置元空间空闲比例的最大值和最小值

    * jmap -clstats PID  打印类加载器数据(PID为当前jvm进程)
    * jstat -gc PID  打印元空间内存信息
    * jps -l   打印当前运行的所有java进程信息
3. jcmd 命令(JDK 1.7开始增加的命令)
    * jcmd  查看当前运行的所有java进程
    * jcmd pid VM.flags     打印进程的JVM启动参数
    * jcmd pid help     列出当前运行的java进程可以执行的jcmd指令
    * jcmd pid help JFR.dump    查看JFR.dump等具体指令的用法
    * jcmd pid PerfCounter.print    查看JVM性能相关的参数
    * jcmd pid VM.uptime    查看进程启动时长
    * jcmd pid GC.class_histogram   查看系统中类的统计信息
    * jcmd pid Thread.print     查看指定进程所有线程堆栈信息
    * jcmd pid GC.heap_dump filename    导出heap dump文件(.hprof), 导出的文件可以通过jvisualvm查看
    * jcmd pid VM.system_properties     查看JVM的属性信息
    * jcmd pid VM.version   查看目标JVM进程的版本信息
    * jcmd pid VM.command_line  查看JVM启动的命令行参数信息
4. jstack pid   查看或导出指定进程中所有线程的堆栈信息

5. jmc : Java Mission Control   (集大成的可视化工具)
6. jhat .hprof文件  : 查看heap dump文件