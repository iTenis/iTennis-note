# 1 Java虚拟机

## 1.1 Java内存区域与内存溢出异常

- 线程隔离私有数据区：

  - 程序计数器
    - **存储**当前线程所执行的字节码的行号指示器，从而使字节码指示器根据程序计数器选取下一条需要执行的字节码指令
    - 各个线程间计数器互不影响，独立存储
    - 唯一没有规定任何OOM的异常情况的区域

  - Java虚拟机栈
    - **存储**局部变量表、操作数栈、动态连接、方法出口等信息
    - 局部变量表存放了编译器可知的各种Java虚拟机的基本数据类型、对象引用和returnAddress类型(下一条字节码指令的地址)
    - 存在栈深度溢出和栈扩展失败异常(StackOverflowError、OutOfMemoryError)
  - 本地方法栈
    - **存储**虚拟机使用到的本地(Native)方法服务
    - 存在栈深度溢出和栈扩展失败异常(StackOverflowError、OutOfMemoryError)

- 所有线程共享数据区：

  - Java堆
    - 虚拟机所管理内存中最大的区域，虚拟机启动时创建
    - **存储**对象实例和数组
    - Java堆是垃圾收集器管理的内存区域，也被称为“GC堆”
    - Java堆可以处于物理不连续的内存空间，但在逻辑上应该是连续的
    - Java堆即可以实现固定大小，也可以实现动态扩展，通常通过"-Xmx和-Xms"设置
    - 存在栈扩展失败异常(OutOfMemoryError)
  - 方法区
    - **存储**已被虚拟机加载的类型信息、常量、静态变量、即时编译后的代码缓存等数据
    - JDK1.7之前采用永久代实现方法区，JDK1.7之后废弃永久代采用直接内存实现方法区
    - 存在栈扩展失败异常(OutOfMemoryError)

## 1.2 Hotspot虚拟机对象

### 1.2.1 对象创建过程

当虚拟机遇到new字节码指令，对象的创建便开始了：

1. 检查这个指令的参数是否能在常量池中定位到一个类的符号引用，并且检查这个符号引用代表的类是否已被加载、解析和初始化过。如果没有，那就必须先进行相应的类加载过程
2. 虚拟机为新生对象分配内存，内存分配的方式由Java堆是否规整决定，Java堆是否规整又由采用的垃圾回收器是否采用带有空间压缩整理的能力决定。
   - 分配方式一：**指针碰撞**，内存是规整的
   - 分配方式一：**空闲列表**，使用的内存和未使用的内存是交错的
   - 存在的问题：对象的频繁创建中，虽然仅仅修改一个指针所指向的位置，在并发的情况下也不是线程安全的，可能出现正在给对象A分配内存，指针还没来得及修改，对象B又同时使用了原来的指针来分配内存的情况,可选解决方案:
     - 方案一：对分配内存空间的动作进行同步，实际上虚拟机采用的是CAS+失败重试的方式来保证更新操作的原子性
     - 方案二：把内存分配的动作按照线程划分在不同的空间进行，即每个线程在Java堆中预先分配内存，称为本地线程分配缓冲（Thread Local Allocation Buffer，TLAB），只有本地缓冲区使用完了，分配新的缓冲区时才需要同步锁定。虚拟机是否采用通过-XX:+UseTLAB启用
3. 分配到的内存空间初始化零值(不包括对象头)，如果使用TLAB的话，提前在TLAB分配时初始化零值，保证对象不赋值也能访问到对应数据类型所对应的零值
4. 设置对象头，类的元数据信息、对象的哈希码(实际对象的哈希码会延后到真正调用hashCode()方法时才会计算)、对象的GC分代年龄等信息
5. 虚拟机角度，一个新的对象已经产生，Java程序角度，需要经过构造函数，即Class文件中<init>()方法执行，一个真正可用的对象才算完全被构造出来。

### 1.2.2 对象访问方式

1. 使用**句柄**访问对象
   - reference中存储稳定的句柄地址，对象的移动不会只会改变句柄池中的实例数据指针，不会改变reference地址
2. 通过**直接指针**访问对象：
   - 减少一次额外转发，速度更快

### 1.2.3 Java堆溢出

Java内存溢出异常情况解决思路：

1. 通过-XX:+HeapDumpOnOutOfMemoryError指定虚拟机出现内存溢出异常时候Dump出当前的内存堆转储快照
2. 通过内存映像分析工具对Dump出来的堆转储快照进行分析
3. 确认内存溢出导致OOM是内存泄漏还是内存溢出
   - 内存泄漏：进一步通过工具查看泄漏对象到GC Roots的引用链，找到泄漏对象是通过怎样的引用路径、与哪些GC Roots关联，才导致垃圾收集器无法回收，一般可以准确的定位到对象创建的位置，进而找出产生内存泄漏的代码具体位置
   - 内存溢出：说明对象确实存在且必须存活，那就需要调整Java虚拟机的堆参数(-Xmx和-Xms)，再从代码上检查是否存在某些对象的生命周期过长、持有状态时间过长、存储结构不合理等情况，尽量减少内存消耗

## 1.3 垃圾收集器与内存分配策略

### 1.3.1 判断对象是否存活

1. 引用计数算法
   - **定义**：对象被引用，计数器加一；引用失效，计算器减一；任何时刻计数器为零的对象就是不可能在被使用到的
   - **优点**：实现简单，效率高
   - **问题**：对象之间相互引用的问题导致无法回收
2. 可达性分析算法
   - **定义**：通过一系列称为“GC Roots”的根对象作为起始节点集，从这些节点开始，根据引用关系向下搜索，搜索过程所走过的路径称为“引用链”，如果某个对象到GC Roots间没有任何引用链相连，或者用图论的话就是从GC Roots到这个对象不可达时，证明此对象会被判定为不能再被使用的
   - 判定为不可达对象需要至少经历**两次标记**过程：第一次判定不可达标记后会放入到F-Queue队列中，虚拟机会自动建立一个低调度优先级的Finalizer线程去执行它们的finalize()方法，对象的finalize()方法只会被系统自动调用一次，如果失败则对象将不会被回收， 所以需要谨慎重写或者不要重写该方法
   - GC Roots对象包含：
     -  在虚拟机栈中引用的对象
     - 在方法区中类静态属性引用 的对象
     - 在方法区中常量引用的对象
     - 在本地方法栈中JNI(Native方法)引用的对象
     - Java虚拟机内部的引用
     - 所有被同步锁(Synchronized)持有的对象
     - 反映Java虚拟机内部情况的JMXBean、JVMTI中注册的回调、本地代码缓存等
     - 垃圾回收器“临时性”加入的对象

3. 引用
   - 强引用：只要强引用关系存在，垃圾回收器就永远不会回收掉引用的对象，类似new关键字实例的对象
   - 软引用：在系统将要发生内存溢出前，会把这些对象列进回收范围之中进行第二次回收，如果这次回收还没有足够的内存，才会抛出内存溢出异常
   - 弱引用：无论当前内存是否够用，都会回收掉只被弱引用关联的对象
   - 虚引用：只是为了能够在这个对象被回收时收到一个系统通知。

### 1.3.2 垃圾回收算法

垃圾回收算法可以划分为“引用计数式垃圾手机”和“追踪式垃圾收集”，也被称为“直接垃圾收集”和“间接垃圾收集”

1. 分代收集理论

   - 弱分代假说：绝大多数的对象都是朝生夕灭的
   - 强分代假说：熬过多次垃圾收集过程的对象就越难以消亡
   - 跨代引用假说：跨代引用相对于同代引用来说仅占极少数，存在相互引用关系的两个对象，应该倾向于同时生存或者消亡(“记忆集”，“卡表”方式解决)，通过记忆集来实现GC Roots扫描范围问题。通过写屏障来保障卡表赋值。
     - 记忆集：一种用于记录从非收集区域指向收集区域的指针集合的抽象数据结构
     - 卡表：是记忆集的一种具体实现，定义了记忆集的记录精度与堆内存的映射关系等。

2. 标记-清除算法

   - 执行效率不稳定
   - 内存空间碎片化问题

3. 标记-复制算法

   - 实现简单，运行高效
   - 可用内存减少为原来的一半

4. 标记-整理算法

5. 并发标记过程中的解决方案

   - 增量更新
   - 原始快照

### 1.3.3 垃圾收集器

垃圾收集器一般关注两个方面：**吞吐量**、**低延迟**。吞吐量是处理器用于运行用户代码的时间与处理器总消耗的时间的比值。

1. Serial收集器

2. ParNew收集器

3. Parallel Scavenge收集器

4. Serial Old收集器

5. Parallel Old收集器

6. CMS收集器

7. G1收集器

   1. G1收集器采用Mixed GC模式
   2. 把连续的Java堆划分为多个相同大小的独立区域(Region)，和特殊区域Humongous，用来存储大对象。大小超过Region一半就判定为大对象，可以-XX:G1HeapRegionSize动态指定，如果超过大对象容量，则使用连续N个Humongous Region存放
   3. Region作为单次回收的最小单元，每次收集都是Region大小的整数倍，避免整个Java堆中进行全区域的垃圾收集
   4. 面向全堆进行回收，针对回收集(CSet)，衡量标准不再是分代，而是哪块内存存放的垃圾数量最多，回收效益最大
   5. 利用卡表实现跨代指针，每个Region中包含一个卡表

8. ZGC收集器

   - 通过使用读屏障、染色指针和内存多重映射等技术实现并发标记整理算法
   - 采用Region内存布局
     - 小型Region(2M，存放小于256k对象)、
     - 中型Region(32M，存放大于256k对象并且小于4M)、
     - 大型Region(容量不固定，可以动态变化，但必须是2的整数倍)
   - 通过扫描全部Region，省去使用记忆集的维护成本
   - 自愈：重分配集中的存活对象复制到新的Region上，此时用户线程并发访问位于重分配集中的对象，这次访问会被预置的内存屏障所拦截，然后立即根据regina上的转发表记录将访问转发到新复制的对象上，并同时修正更新该引用的值，使其直接指向新对象，这种称为指针的自愈能力。

### 1.3.4 动态对象年龄判断

1. 经过第一次Minor GC后仍然存活，并且能被Survivor容纳的话，设置对象的年龄为1，没熬过一次Minor GC，年龄加一，当年龄达到一定程度(默认15)，就会被晋升到老年代中。通过-XX:MaxTenuringThreshold设置最大年龄阀值
2. 如果在Survivor空间中相同年龄所有对象大小的总和大约Survivor空间的一半，年龄大于或等于该年龄的对象可以直接进入老年代
3. Full GC触发时机：
   - 在发生Minor GC之前，虚拟机必须先检查老年代最大可用的连续空间是否大于新生代所有对象总空间，如果成立：检查-XX:HandlePromotionFailure参数是否允许担保失败，如果允许：检查老年代最大可用的连续空间是否大于历次晋升到老年代对象的平均大小，大于则进行Minor GC，否则，进行Full GC。