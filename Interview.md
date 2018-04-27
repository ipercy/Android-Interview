## LruCache原理

LRU(Least Recently Used)，近期最少使用的算法，它的核心思想是当缓存满时，会优先淘汰那些近期最少使用的缓存对象。采用LRU算法的缓存有两种：LruCache和DiskLruCache，分别用于实现内存缓存和硬盘缓存，其核心思想都是LRU缓存算法。

LruCache是个泛型类，主要算法原理是**把最近使用的对象用强引用**（即我们平常使用的对象引用方式）存储在 **LinkedHashMap** 中。当缓存满时，把最近最少使用的对象从内存中移除，并提供了get和put方法来完成缓存的获取和添加操作。

![LRU算法](https://upload-images.jianshu.io/upload_images/3985563-33560a9500e72780.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)

这个队列是由LinkedHashMap来维护的，而LinkedHashMap是由**数组+双向链表**的数据结构来实现的。其中双向链表的结构可以实现访问顺序和插入顺序，使得LinkedHashMap中的<key,value>对按照一定顺序排列起来。



## 三级缓存

三级缓存策略：通过网络、本地、内存三级缓存图片，来减少不必要的网络交互，避免浪费流量

- 网络缓存：通过Cache-Control的设置进行网络缓存
- 内存缓存：内存缓存采用LruCache
- 磁盘缓存：使用的是DiskLruCache



## HashMap原理

在HashMap中有两个很重要的参数，容量(Capacity)和负载因子(Load factor)

简单的说，Capacity就是buckets的数目，Load factor就是buckets填满程度的最大比例，默认值为0.75。如果对迭代性能要求很高的话不要把`capacity`设置过大，也不要把`load factor`设置过小。当bucket填充的数目（即hashmap中元素的个数）大于`capacity*load factor`时就需要调整buckets的数目为当前的2倍。



put函数大致的思路为：

1. 对key的hashCode()做hash，然后再计算index;
2. 如果没碰撞直接放到bucket里；
3. 如果碰撞了，以链表的形式存在bucket后；
4. 如果碰撞导致链表过长(大于等于TREEIFY_THRESHOLD)，就把链表转换成红黑树；
5. 如果节点已经存在就替换old value(保证key的唯一性)
6. 如果bucket满了(超过load factor*current capacity)，就要resize。

get函数大致思路如下：

1. bucket里的第一个节点，直接命中；
2. 如果有冲突，则通过key.equals(k)去查找对应的entry
   若为树，则在树中通过key.equals(k)查找，O($log_{2}^{n}$)；
   若为链表，则在链表中通过key.equals(k)查找，O(n)。

在get和put的过程中，计算下标时，先对hashCode进行hash操作，然后再通过hash值进一步计算下标，如下图所示：

![计算hash过程](https://cloud.githubusercontent.com/assets/1736354/6957712/293b52fc-d932-11e4-854d-cb47be67949a.png)

hash函数的作用是高16bit不变，低16bit和高16bit做了一个异或

在设计hash函数时，因为目前的table长度n为2的幂，而计算下标的时候，是这样实现的(使用`&`位操作，而非`%`求余)：

```
(n - 1) & hash
```

设计者认为这方法很容易发生碰撞。为什么这么说呢？不妨思考一下，在n - 1为15(0x1111)时，其实散列真正生效的只是低4bit的有效位，当然容易碰撞了。

因此，设计者想了一个顾全大局的方法(综合考虑了速度、作用、质量)，就是把高16bit和低16bit异或了一下。设计者还解释到因为现在大多数的hashCode的分布已经很不错了，就算是发生了碰撞也用O($log_{2}^{n}$)的tree去做了。仅仅异或一下，既减少了系统的开销，也不会造成的因为高位没有参与下标的计算(table长度比较小时)，从而引起的碰撞。

在获取HashMap的元素时，基本分两步：

1. 首先根据hashCode()做hash，然后确定bucket的index；
2. 如果bucket的节点的key不是我们需要的，则通过keys.equals()在链中找。

在**Java 8之前的实现中是用链表**解决冲突的，在产生碰撞的情况下，进行get时，两步的时间复杂度是O(1)+O(n)。因此，当碰撞很厉害的时候n很大，O(n)的速度显然是影响速度的。

因此在**Java 8中，利用红黑树替换链表**，这样复杂度就变成了O(1)+O($log_{2}^{n}$)了，这样在n很大的时候，能够比较理想的解决这个问题



当put时，如果发现目前的bucket占用程度已经超过了Load Factor所希望的比例，那么就会发生resize。在resize的过程，简单的说就是把bucket扩充为2倍，之后重新计算index，把节点再放到新的bucket中。所以，元素的位置要么是在原位置，要么是在原位置再移动2次幂的位置。

例如我们从16扩展为32时，具体的变化如下所示：

![扩展为32](https://cloud.githubusercontent.com/assets/1736354/6958256/ceb6e6ac-d93b-11e4-98e7-c5a5a07da8c4.png)

因此元素在重新计算hash之后，因为n变为2倍，那么n-1的mask范围在高位多1bit(红色)，因此新的index就会发生这样的变化：

![index变化](https://cloud.githubusercontent.com/assets/1736354/6958301/519be432-d93c-11e4-85bb-dff0a03af9d3.png)

因此，我们在扩充HashMap的时候，不需要重新计算hash，只需要看看原来的hash值新增的那个bit是1还是0就好了，是0的话索引没变，是1的话索引变成“原索引+oldCap”。


## HashMap与Hashtable的区别

HashMap和Hashtable都实现了Map接口。主要的区别有：线程安全性，同步(synchronization)，以及速度。

HashMap几乎可以等价于Hashtable，除了HashMap是非synchronized的，并可以接受null(HashMap可以接受为null的键值(key)和值(value)，而Hashtable则不行)。也就是说Hashtable是线程安全的，多个线程可以共享一个Hashtable。Java 5提供了**ConcurrentHashMap**，它是HashTable的替代，比HashTable的扩展性更好。

HashMap可以通过下面的语句进行同步：

```java
Map m = Collections.synchronizeMap(hashMap);
```



## ConcurrentHashMap和Hashtable的区别

当Hashtable的大小增加到一定的时候，性能会急剧下降，因为迭代时需要被锁定很长的时间。因为ConcurrentHashMap引入了分割(segmentation)，不论它变得多么大，仅仅需要锁定map的某个部分，而其它的线程不需要等到迭代完成才能访问map。简而言之，在迭代的过程中，ConcurrentHashMap仅仅锁定map的某个部分，而Hashtable则会锁定整个map。

## SparseArray

使用int[]数组存放key，避免了HashMap中基本数据类型需要装箱的步骤，其次不使用额外的结构体（Entry)，单个元素的存储成本下降。


- SparseArray的key为int，value为Object。
- 相比与HashMap，其采用 **时间换空间** 的方式，使用更少的内存来提高手机APP的运行效率(HashMap中当table数组中内容达到总容量0.75时，则扩展为当前容量的两倍)

## ArrayMap

SparseArray键中始终是原始类型，旨在消除自动装箱的问题，而ArrayMap不能避免自动装箱问题，在其他方面的操作原理是相似的。



## ArrayList和LinkedList区别

1. ArrayList是实现了基于可改变大小数组的数据结构，LinkedList基于双向链表的数据结构。 
2. 对于随机访问get和set，ArrayList绝对优于LinkedList，因为LinkedList要移动指针。
3. 对于新增和删除操作add和remove，LinkedList比较占优势，因为ArrayList要移动数据。



## CopyOnWriteArrayList

遍历List的同时操作List，比如说删除其中的元素会抛出`java.util.ConcurrentModificationException`

ArrayList是非线程安全的，Vector是线程安全的，即使把ArrayList换成Vector还是不可以线程安全地遍历，因为从Vector源码可以发现它的很多方法都加上了synchronized来进行线程同步，例如add()、remove()、set()、get()，但是Vector内部的synchronized方法无法控制到遍历操作，所以即使是线程安全的Vector也无法做到线程安全地遍历。

CopyOnWriteArrayList类最大的特点就是，在对其实例进行修改操作（add/remove等）会拷贝一份新的List并且在新的List上进行修改，最后将原List的引用指向新的List。这样也就没有了ConcurrentModificationException错误。



## ThreadLocal工作原理

ThreadLocal提供了线程本地变量，它可以保证访问到的变量属于当前线程，每个线程都保存有一个变量副本，每个线程的变量都不同。ThreadLocal相当于提供了一种线程隔离，将变量与线程相绑定。

每个线程内部都有一个名字为threadLocals的成员变量，该变量类型为HashMap，其中key为我们定义的ThreadLocal变量的this引用，value则为我们set时候的值，每个线程的本地变量是存到线程自己的内存变量threadLocals里面的，如果当前线程一直不消失那么这些本地变量会一直存在，所以可能会造成内存溢出，所以使用完毕后要记得调用ThreadLocal的remove方法删除对应线程的threadLocals中的本地变量。



## SharedPreference 多进程

MODE\_MULTI\_PROCESS，跨进程模式，如果项目有多个进程使用同一个Preference，需要使用该模式，但是也已经废弃了,**建议使用ContentProvider替代**。

- commit() 是**同步提交到内存后再同步提交到磁盘**上，如果 commit() 之前还有没结束的异步任务（包括 apply() 的提交），就会一直阻塞到前面的提交都完成，才进行提交。
- apply() 是**立即提交到内存后异步提交到磁盘**上。
- commit() 有返回值，而 apply() 没有返回值。
- 存在内存与磁盘数据不同步的情况，多进程共享需要注意数据安全。


## SurfaceView
View类如果需要更新视图，必须我们主动的去调用invalidate()或者postInvalidate()方法来再走一次onDraw()完成更新。但是呢，Android系统规定屏幕的刷新间隔为16ms，如果这个View在16ms内更新完毕了，就不会卡顿，但是如果逻辑操作太多，16ms内没有更新完毕，剩下的操作就会丢到下一个16ms里去完成，这样就会造成UI线程的阻塞，造成View的运动过程掉帧，自然就会卡顿了。
所以这些原因也就促使了SurfaceView的存在。毕竟，如果是一个游戏，它有可能相当频繁的有更新画面的需求。

**SuraceView的主要优势**

1、SurfaceView的刷新处于主动，有利于频繁的更新画面。

2、SurfaceView的绘制在子线程进行，避免了UI线程的阻塞。

3、SurfaceView在底层实现了一个双缓冲机制，效率大大提升。

但是SurfaceView也是View派生而来的。

双缓冲技术是把要处理的图片在内存中处理好之后，再将其显示在屏幕上。双缓冲主要是为了解决 **反复局部刷屏带来的闪烁**。把要画的东西先画到一个内存区域里，然后整体的一次性画出来



## Android广播分类

- 无序广播

  > 没有顺序的广播，广播的接收方没有严格的顺序可言，不可中断
- 有序广播

  > 在注册时可指定优先级，优先级高的广播接收者优先收到广播，优先级以一个整数来标识，数值越大优先级越高。可中断，可再修饰
- 粘滞广播

  > 发出的广播会滞留，注册时间可晚于发送时间，其他功能与无序广播相同。在Android6.0中已经被标记为过时，它有不安全(任何App都能访问), 没有保护 (任何App都能修改)等问题
- 本地广播

  > 本地广播只有本应用内通过`LocalBroadcastManager.getInstance(this).registerReceiver(BroadcastReceiver receiver, IntentFilter filter)`方法注册的广播接收者能收到，具有更高的安全性，效率也更高

## LocalBroadcastReceiver
相对 BroadcastReceiver，它只能用于应用内通信，安全性更好，同时拥有更高的运行效率，不是走Binder机制。**核心实现实际还是 Handler**，只是利用到了 IntentFilter 的 match 功能



## ClassLoader加载类的原理

ClassLoader使用的是**双亲委托模型**来搜索类的，每个ClassLoader实例都有一个父类加载器的引用（不是继承的关系，是一个包含的关系），虚拟机内置的类加载器（Bootstrap ClassLoader）本身没有父类加载器，但可以用作其它ClassLoader实例的父类加载器。当一个ClassLoader实例需要加载某个类时，它会试图亲自搜索某个类之前，先把这个任务委托给它的父类加载器，这个过程是由上至下依次检查的，首先由最顶层的类加载器Bootstrap ClassLoader试图加载，如果没加载到，则把任务转交给Extension ClassLoader试图加载，如果也没加载到，则转交给App ClassLoader 进行加载，如果它也没有加载得到的话，则返回给委托的发起者，由它到指定的文件系统或网络等URL中加载该类。如果它们都没有加载到这个类时，则抛出ClassNotFoundException异常。否则将这个找到的类生成一个类的定义，并将它加载到内存当中，最后返回这个类在内存中的Class实例对象。

**为什么要使用双亲委托这种模型呢**？

因为这样可以**避免重复加载**，当父亲已经加载了该类的时候，就没有必要子ClassLoader再加载一次。考虑到安全因素，我们试想一下，如果不使用这种委托模式，那我们就可以随时使用自定义的String来动态替代java核心api中定义的类型，这样会存在非常大的安全隐患

JVM在判定两个class是否相同时，_不仅要判断两个类名是否相同，而且要判断是否由同一个类加载器实例加载的_。只有两者同时满足的情况下，JVM才认为这两个class是相同的

![ClassLoader的体系架构](http://hi.csdn.net/attachment/201202/25/0_13301699801ybH.gif)

Android中的类加载器：PathClassLoader 和 DexClassLoader 

PathClassLoader 在应用启动时创建，从 data/app/… 安装目录下加载 apk 文件。BootClassLoader 是 PathClassLoader 的父加载器，其在系统启动时创建，在 App 启动时会将该对象传进来。

对比 PathClassLoader 只能加载已经安装应用的 dex 或 apk 文件，DexClassLoader 则没有此限制，可以从 SD 卡上加载包含 class.dex 的 .jar 和 .apk 文件，这也是插件化和热修复的基础，在不需要安装应用的情况下，完成需要使用的 dex 的加载。

![Android类加载器](https://blog.dreamtobe.cn/img/android_dynamic_dex.png)



## Java运行时数据区域

![Java运行时数据区域](https://upload-images.jianshu.io/upload_images/2614605-246286b040ad10c1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/578)

- 程序计数器

  > 程序计数器占用较小的内存空间，可以看做是**当前线程所执行的字节码的行号指示器**。
  >
  > 由于Java虚拟机的多线程是通过线程轮流切换并分配处理器执行时间的方式来实现的，在任何一个确定的时刻，一个处理器（对于多核处理器来说就是一个内核）都只会执行一条线程中的指令。因此，**为了线程切换后能够恢复到正确的执行位置**，每条线程都需要有一个独立的程序计数器。

  > 如果线程正在执行Java方法，则计数器记录的是正在执行的虚拟机字节码指令的地址；如果正在执行的是Native方法，则这个计数器则为空(Undefined)。

- Java虚拟机栈

  > 与程序计数器一样，Java虚拟机栈也是线程私有的，而且生命周期与线程相同，每个Java方法在执行的时候都会创建一个栈帧（Stack Frame）用于**存储局部变量表、操作数栈、动态链接、方法出口等信息**。

- 本地方法栈

  > 本地方法栈的作用与虚拟机栈作用是非常类似的，区别在于虚拟机栈为虚拟机执行Java方法（也就是字节码）服务，而本地方法栈则为虚拟机使用到的Native方法服务。

- Java堆

  > Java堆（Heap）是Java虚拟机所管理的内存中最大的一块，Java堆是被所有线程共享的一块内存区域，在虚拟机启动时创建。该内存区域唯一的目的就是**存放对象实例**，**Java对象实例以及数组都在堆上分配**（随着JIT编译器发展等技术成熟，所有对象分配在堆上也渐渐不是那么“绝对”了）。
  >
  > Java堆是垃圾收集器管理的主要区域，因此Java堆也常被称为“GC堆”，由于现在收集器基于分代收集算法，Java堆还可以细分为：新生代和老年代；再细致一点的有Eden空间、From Survivor空间、To Survivor空间等。
  >
  > 根据Java虚拟机规范的规定，Java堆可以处于物理上不连续的内存空间中，只要逻辑上是连续的即可

- 方法区

  > 方法区与Java堆一样，是各个线程共享的内存区域，用于**存储已被虚拟机加载的类信息、常量、静态变量、即时编译器编译后的代码等数据**。很多人愿意把方法区称为“永久代”（Permanent Generation），本质上两者并不等价。

- 运行时常量池

  > **运行时常量池（Runtime Constant Pool）是方法区的一部分**。Class文件中除了有关类的版本、字段、方法、接口等描述信息外，还有一项信息是**常量池（Constant Pool Table）**，用于存放编译期生成的各种字面量和符号引用，这部分内容将在类加载后进入方法区的运行时常量池中存放。




## Java内存分配策略

- 对象优先在Eden分配

  > 大多数情况下，对象在新生代Eden区中分配。当Eden区没有足够空间进行分配时，虚拟机将发起一次 Minor GC

- 大对象直接进入老年代

  > 所谓大对象是指，需要大量连续内存空间的Java对象，最典型的大对象就是那种很长的字符串以及数组。

- 长期存活的对象将进入老年代

  > 既然虚拟机采用了分代收集的思想来管理内存，那么内存回收时就必须能识别哪些对象应放在新生代，哪些对象应放在老年代。为做到这一点，**虚拟机给每个对象定义了一个对象年龄（Age）计数器**。如果对象在Eden出生并经过第一次Minor GC后仍然存活，并且能被Survivor容纳的话，将被移动到Survivor空间中，并且对象年龄设为1。对象在Survivor区中每熬过一次Minor GC，年龄就增加1岁，当它的年龄增加到一定程度（默认15岁），就会被晋升到老年代中。对象晋升老年代的年龄阈值，可通过参数`-XX:MaxTenuringThreshold`设置

- 动态对象年龄判定

  > 为了能更好地适应不同程序的内存状况，虚拟机并不是永远地要求对象的年龄必须达到了MaxTenuringThreshold才能晋升老年代，如果在Survivor空间中相同年龄所有对象大小的总和大于Survivor空间的一半，年龄大于或等于该年龄的对象就可以直接进入老年代，无须等到MaxTenuringThreshold中要求的年龄。



Java 程序运行时的内存分配策略有三种,分别是静态存储区（也称方法区）、栈区和堆区。

- 静态存储区（方法区）：主要存放静态数据、全局 static 数据和常量。这块内存在程序编译时就已经分配好，并且在程序整个运行期间都存在。
- 栈区 ：当方法被执行时，方法体内的局部变量（其中包括基础数据类型、对象的引用）都在栈上创建，并在方法执行结束时这些局部变量所持有的内存将会自动被释放。因为栈内存分配运算内置于处理器的指令集中，效率很高，但是分配的内存容量有限。
- 堆区 ： 又称动态内存分配，通常就是指在程序运行时直接 new 出来的内存，也就是对象的实例。这部分内存在不使用时将会由 Java 垃圾回收器来负责回收。

**栈与堆的区别**

```java
public class Sample {
    int s1 = 0;
    Sample mSample1 = new Sample();

    public void method() {
        int s2 = 1;
        Sample mSample2 = new Sample();
    }
}

Sample mSample3 = new Sample();
```

Sample 类的局部变量 s2 和引用变量 mSample2 都是存在于栈中，但 mSample2 指向的对象是存在于堆上的。mSample3 指向的对象实体存放在堆上，包括这个对象的所有成员变量 s1 和 mSample1，而它自己存在于栈中。

结论：

- 局部变量的基本数据类型和引用存储于栈中，引用的对象实体存储于堆中。—— 因为它们属于方法中的变量，生命周期随方法而结束。
- 成员变量全部存储于堆中（包括基本数据类型，引用和引用的对象实体）—— 因为它们属于类，类对象终究是要被new出来使用的。




## Java判断对象是否存活

- 引用计数法（Reference Counting）

  > 给对象添加一个引用计数器，每当有一个地方引用它时，计数器值就加1；当引用失效时，计数器就减1；任何时刻计数器为0的对象就是不可能再被使用的。但它很难解决**对象之间相互循环引用**的问题。

- 可达性分析算法（Reachability Analysis）

  > 基本思想是通过一系列的称为“GC Roots”的对象作为起始点，从这些节点开始向下搜索，搜索所走过的路径称为引用链（Reference Chain）,当一个对象到GC Roots没有任何引用链相连（用图论的话来说，就是**从GC Roots到这个对象不可达**）时，则证明此对象是不可用的。




可作为GC Roots的对象有：

1. 虚拟机栈（栈帧中的本地变量表）中引用的对象；

2. 方法区中的类静态属性引用的对象；

3. 方法区中常量引用的对象；

4. 本地方法栈中JNI（即一般说的Native方法）中引用的对象

   ​

## Java垃圾回收算法

- Mark-Sweep Collector(标记-清除收集器）

  > 标记清除收集器停止所有的工作，从根扫描每个活跃的对象，然后标记扫描过的对象，标记完成以后，清除那些没有被标记的对象。

  > 缺点是耗时长，内存碎片多

- Copying Collector(复制收集器）

  > 将内存分为两块，标记完成开始回收时，将一块内存中保留的对象全部复制到另一块空闲内存中。

  > 缺点是需要额外的空间消耗

- Mark-Compact Collector(标记-整理收集器）

  > 标记整理收集器汲取了标记清除和复制收集器的优点，它分两个阶段执行，在第一个阶段，首先扫描所有活跃的对象，并标记所有活跃的对象，第二个阶段首先清除未标记的对象，然后将活跃的对象复制到堆的底部。

- 分代收集算法

  > 当代商业虚拟机的垃圾收集都采用“分代收集（Generational Collection）”算法，该算法并没有什么新的思想，只是根据对象存活周期的不同将内存划分为几块。**一般是把Java堆分为新生代和老年代，这样就可以根据各个年代的特点采用最适合的收集算法**。




## Java四种引用

- 强引用（Strong Reference）最常用的引用类型，如`Object obj = new Object(); `。只要强引用存在则GC时必定不被回收。
- 软引用（Soft Reference）用于描述还有用但非必需对象，当堆将发生OOM（Out Of Memory）时则会回收软引用所指向的内存空间，若回收后依然空间不足才会抛出OOM。
- 弱引用（Weak Reference）发生GC时必定回收弱引用指向的内存空间。和软引用加入队列的时机相同
- 虚引用（Phantom Reference)又称为幽灵引用或幻影引用，虚引用既不会影响对象的生命周期，也无法通过虚引用来获取对象实例，仅用于在发生GC时接收一个系统通知。




## Java各种GC比较

- Minor GC：通常是指对新生代的回收。指发生在新生代的垃圾收集动作，因为 Java 对象大多都具备朝生夕灭的特性，所以 Minor GC 非常频繁，一般回收速度也比较快
- Major GC：通常是指对老年代的回收。
- Full GC：Major GC除并发gc外均需对整个堆进行扫描和回收。指发生在老年代的 GC，出现了 Major GC，经常会伴随至少一次的 Minor GC（但非绝对的，在 ParallelScavenge 收集器的收集策略里就有直接进行 Major GC 的策略选择过程） 。Major GC 的速度一般会比 Minor GC 慢 10倍以上。




## Java类加载机制

![JVM类加载过程](https://upload-images.jianshu.io/upload_images/272719-14daa5893c05f62a.PNG?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)

类加载的过程包括了加载、验证、准备、解析、初始化五个阶段。在这五个阶段中，加载、验证、准备和初始化这四个阶段发生的顺序是确定的，而**解析阶段则不一定**，它在某些情况下可以在初始化阶段之后开始，这是为了支持Java语言的运行时绑定（也称为动态绑定或晚期绑定）

一、加载：

在加载阶段，虚拟机需要完成以下三件事情：

1. 通过一个类的全限定名来获取定义此类的二进制字节流。
2. 将这个字节流所代表的静态存储结构转化为方法区的运行时数据结构。
3. 在内存中生成一个代表这个类的java.lang.Class对象，作为方法区这个类的各种数据的访问入口。

二、验证：

验证是连接阶段的第一步，这一阶段的目的是为了确保Class文件的字节流中包含的信息符合当前虚拟机的要求，并且不会危害虚拟机自身的安全。从整体上看，验证阶段大致上会完成下面4个阶段的校验动作：文件格式验证、元数据验证、字节码验证、符号引用验证。

三、准备： 

准备阶段是正式为类变量分配内存并设置类变量初始值的阶段，这些变量所使用的内存都将在方法区中进行分配。这时候进行内存分配的**仅包括类变量（被static修饰的变量），而不包括实例变量**，实例变量将会在对象实例化时随着对象一起分配在Java堆中。

四、解析：

解析阶段是虚拟机将常量池内的符号引用替换为直接引用的过程。

五、初始化：

类初始化阶段是类加载过程的最后一步，前面的类加载过程中，除了在加载阶段用户应用程序可以通过自定义类加载器参与之外，其余动作完全由虚拟机主导和控制。到了初始化阶段，才真正开始执行类中定义的Java程序代码（或者说是字节码）。



## App沙箱化

 Android从Linux继承了已经深入人心的**类Unix进程隔离机制与最小权限原则**，Android沙箱的核心机制基于以下几个概念：<u>标准的Linux进程隔离、大多数进程拥有唯一的用户ID（UID），以及严格限制文件系统权限</u>。

沙箱系统的原理主要基于Linux系统的UID/GID机制。Android对传统的Linux的UID/GID机制进行了修改。在 Linux 中，一个用户 ID 识别一个给定用户;在 Android 上，一个用户 ID 识别一个应用程序。



## APK安装过程

![APK安装过程](http://solart.cc/images/Install_apk.png)

1. 将apk文件复制到data/app目录
2. 解析apk信息
3. dexopt操作,存放于`data/dalvik-cache`,
4. 更新权限信息
5. 完成安装,发送Intent.ACTION\_PACKAGE_ADDED广播

上述的dexopt操作，对于dalvik虚拟机,dexopt就是优化操作,而对于art虚拟机,dexopt执行的则是dex2oat操作,既将.dex文件翻译成oat文件。



## App启动流程

![App启动流程](https://upload-images.jianshu.io/upload_images/2057980-9c537daee4ef6932.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)



1. 通过 Launcher 启动应用时，点击应用图标后，Launcher 调用 `startActivity` 启动应用。
2. Launcher Activity 最终调用 `Instrumentation` 的 `execStartActivity` 来启动应用。
3. `Instrumentation` 调用 `ActivityManagerProxy` (`ActivityManagerService`  在应用进程的一个代理对象) 对象的 `startActivity` 方法启动 `Activity`。
4. 到目前为止所有过程都在 Launcher 进程里面执行，接下来 `ActivityManagerProxy` 对象跨进程调用 `ActivityManagerService` (运行在 `system_server` 进程)的 `startActivity` 方法启动应用。
5. `ActivityManagerService` 的 `startActivity` 方法经过一系列调用，最后调用  `zygoteSendArgsAndGetResult` 通过 `socket` 发送给 `zygote` 进程，`zygote` 进程会孵化出新的应用进程。
6. `zygote` 进程孵化出新的应用进程后，会执行 `ActivityThread` 类的 `main` 方法。在该方法里会先准备好 `Looper` 和消息队列，然后调用 `attach` 方法将应用进程绑定到 `ActivityManagerService`，然后进入 `loop` 循环，不断地读取消息队列里的消息，并分发消息。
7. `ActivityManagerService` 保存应用进程的一个代理对象，然后 `ActivityManagerService` 通过代理对象通知应用进程创建入口 `Activity` 的实例，并执行它的生命周期函数。



## 统计启动时长

- 冷启动：当启动应用时，后台没有该应用的进程，这时系统会重新创建一个新的进程分配给该应用，这个启动方式就是冷启动。冷启动因为系统会重新创建一个新的进程分配给它，所以会先创建和初始化 `Application` 类，再创建和初始化 `MainActivity` 类，最后显示在界面上。
- 热启动：当启动应用时，后台已有该应用的进程（例：按back键、home键，应用虽然会退出，但是该应用的进程是依然会保留在后台，可进入任务列表查看），所以在已有进程的情况下，这种启动会从已有的进程中来启动应用，这个方式叫热启动。热启动因为会从已有的进程中来启动，所以热启动就不会走 `Application` 这步了，而是直接走 `MainActivity`，所以热启动的过程不必创建和初始化 `Application`，因为一个应用从新进程的创建到进程的销毁，`Application` 只会初始化一次。
- 首次启动：首次启动严格来说也是冷启动，之所以把首次启动单独列出来，一般来说，首次启动时间会比非首次启动要久，首次启动会做一些系统初始化工作，如缓存目录的生成，数据库的建立，SharedPreference的初始化，如果存在多 dex 和插件的情况下，首次启动会有一些特殊需要处理的逻辑，而且对启动速度有很大的影响，所以首次启动的速度非常重要，毕竟影响用户对 App 的第一印象。

本地调试时查看启动时间的命令

```
adb shell am start -w packagename/activity
```

类似的输出：

```
$ adb shell am start -W com.speed.test/com.speed.test.HomeActivity Starting: Intent { act=android.intent.action.MAIN cat=[android.intent.category.LAUNCHER] cmp=com.speed.test/.HomeActivity } Status: ok Activity: com.speed.test/.HomeActivity ThisTime: 496 TotalTime: 496 WaitTime: 503 Complete
```

- `WaitTime` 返回从 `startActivity` 到应用第一帧完全显示这段时间. 就是总的耗时，包括前一个应用 `Activity` pause 的时间和新应用启动的时间；
- `ThisTime` 表示一连串启动 `Activity` 的最后一个 `Activity` 的启动耗时；
- `TotalTime` 表示新应用启动的耗时，包括新进程的启动和 `Activity` 的启动，但不包括前一个应用`Activity` pause的耗时。

线上记录启动时间：

**起始时间点**

如果记录冷启动启动时间一般可以在 `Application.attachBaseContext() `开始的位置记录起始时间点

如果记录热启动启动时间点可以在 `Activity.onRestart()` 中记录起始时间点。

**结束时间点**

在 `Activity.onWindowFocusChanged` 记录应用启动的结束时间点，不过需要注意的是该函数，在 Activity焦点发生变化时就会触发，所以要做好判断



## Android开机过程

- BootLoder引导,然后加载Linux内核.
- 0号进程init启动.加载init.rc配置文件,配置文件有个命令启动了zygote进程
- zygote开始fork出SystemServer进程
- SystemServer加载各种JNI库,然后init1,init2方法,init2方法中开启了新线程ServerThread.
- 在SystemServer中会创建一个socket客户端，后续AMS（ActivityManagerService）会通过此客户端和zygote通信
- ServerThread的run方法中开启了AMS,还孵化新进程ServiceManager,加载注册了一大堆的服务,最后一句话进入loop 死循环
- run方法的SystemReady调用resumeTopActivityLocked打开锁屏界面

## Dalvik和ART

Dalvik是基于寄存器的，而JVM是基于栈的。

在Dalvik下，应用每次运行的时候，字节码都需要通过即时编译器（just in time ，JIT）转换为机器码，这会拖慢应用的运行效率，而在ART 环境中，应用在第一次安装的时候，字节码就会预先编译成机器码，使其成为真正的本地应用。这个过程叫做预编译（AOT,Ahead-Of-Time）。这样的话，应用的启动(首次)和执行都会变得更加快速。



## Android IPC进程间通信

Binder是Android中的一种跨进程通信方式（还有Socket、ContentProvider等等）。

Binder是ServiceManager连接各种Manager（ActivityManager、WindowManager等等）和相应的ManagerService的桥梁。

![Binder机制](http://hi.csdn.net/attachment/201107/19/0_13110996490rZN.gif)

![Binder是什么](http://upload-images.jianshu.io/upload_images/944365-45db4df339348b9b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**各种IPC的差异以及选择**

![各种IPC](http://hujiaweibujidao.github.io/images/androidart_ipc.png)



## Serializable和Parcelable

Serializable是Java中的序列化接口，其使用起来简单但是开销很大，序列化和反序列过程需要大量的I/O操作。但Parcelable是Android中的序列化方式，因此更适合用在Android平台上，缺点是用起来稍微麻烦，但是它效率很高。



## Bundle

要实现IPC跨进程通信，主要包含三方面的内容：Serializable接口、Parcelable接口以及Binder。Serializable和Parcelable接口可以完成对象的序列化过程，当我们需要通过Intent和Binder传输数据时就需要使用Serializable或Parcelable。

四大组件中的三大组件（Activity、Service、Receiver）都是支持在Intent中传递Bundle数据的，**由于Bundle实现了Parcelable接口**，所以它可以很方便的在不同进程间传输。



## StringBuffer与StringBuilder的区别

- String 字符串常量
- StringBuffer 字符串变量（线程安全）
- StringBuilder 字符串变量（非线程安全）

简要的说， String 类型和 StringBuffer 类型的主要性能区别其实在于 **String 是不可变的对象**, 因此在每次对 String 类型进行改变的时候其实都等同于生成了一个新的 String 对象，然后将指针指向新的 String对象，所以经常改变内容的字符串最好不要用 String

使用策略：

（1）基本原则：如果要操作少量的数据，用String ；单线程操作大量数据，用StringBuilder ；多线程操作大量数据，用StringBuffer。

（2）不要使用String类的"+"来进行**频繁的拼接**，因为那样的性能极差的，应该使用StringBuffer或StringBuilder类，这在Java的优化上是一条比较重要的原则。



## Synchronized与Lock锁的区别        

- Synchronized：在资源竞争不是很激烈的情况下，偶尔会有同步的情形下，synchronized是很合适的。原因在于，编译程序通常会尽可能的进行优化synchronized，另外可读性非常好，不管用没用过5.0多线程包的程序员都能理解。 
- ReentrantLock:**ReentrantLock提供了多样化的同步**，比如有时间限制的同步，可以被Interrupt的同步（synchronized的同步是不能Interrupt的）等。在资源竞争不激烈的情形下，性能稍微比synchronized差一点。但是当同步非常激烈的时候，synchronized的性能一下子能下降好几十倍。而ReentrantLock却还能维持常态。




## Java创建线程三种方式

- 创建一个类继承Thread
- 通过Runnable接口创建线程类
- 使用Callable和FutureTask创建线程

采用实现Runnable、Callable接口的方式创建多线程时，优势是：线程类只是实现了Runnable接口或Callable接口，还可以继承其他类。

在这种方式下，多个线程可以共享同一个target对象，所以非常适合多个相同线程来处理同一份资源的情况，从而可以将CPU、代码和数据分开，形成清晰的模型，较好地体现了面向对象的思想。

使用继承Thread类的方式创建多线程时优势是：编写简单，如果需要访问当前线程，则无需使用Thread.currentThread()方法，直接使用this即可获得当前线程。



## Java线程池

Java通过Executors提供四种线程池，Executors 提供了一系列工厂方法用于创建线程池，返回的线程池都实现了 ExecutorService 接口，分别为：

- `newCachedThreadPool` 创建一个可缓存线程池，如果线程池长度超过处理需要，可灵活回收空闲线程，若无可回收，则新建线程。
- `newFixedThreadPool` 创建一个定长线程池，可控制线程最大并发数，超出的线程会在队列中等待。
- `newScheduledThreadPool` 创建一个定长线程池，支持定时及周期性任务执行。
- `newSingleThreadExecutor` 创建一个单线程化的线程池，它只会用唯一的工作线程来执行任务，保证所有任务按照指定顺序(FIFO, LIFO, 优先级)执行。

自定义线程池，可以用`ThreadPoolExecutor `类创建，它有多个构造方法来创建线程池，用该类很容易实现自定义的线程池

```java
public ThreadPoolExecutor(int corePoolSize, int maximumPoolSize, long keepAliveTime, TimeUnit unit, BlockingQueue<Runnable> workQueue, ThreadFactory threadFactory, RejectedExecutionHandler handler)
```

1. `corePoolSize` 核心线程数大小，当线程数 < corePoolSize ，会创建线程执行 runnable
2. `maximumPoolSize` 最大线程数， 当线程数 >= corePoolSize的时候，会把 runnable 放入 workQueue中
3. `keepAliveTime`  保持存活时间，当线程数大于corePoolSize的空闲线程能保持的最大时间。
4. `unit` 时间单位
5. `workQueue` 保存任务的阻塞队列
6. `threadFactory` 创建线程的工厂
7. `handler` 拒绝策略



**任务执行顺序**

1、当线程数小于 `corePoolSize`时，创建线程执行任务。

2、当线程数大于等于 `corePoolSize`并且 `workQueue` 没有满时，放入`workQueue`中

3、线程数大于等于 `corePoolSize`并且当 `workQueue` 满时，新任务新建线程运行，线程总数要小于 `maximumPoolSize`

4、当线程总数等于 `maximumPoolSize` 并且 `workQueue` 满了的时候执行 `handler` 的 `rejectedExecution`。也就是拒绝策略。

**四个拒绝策略**

ThreadPoolExecutor默认有四个拒绝策略：

1、`ThreadPoolExecutor.AbortPolicy()`   直接抛出异常`RejectedExecutionException`

2、`ThreadPoolExecutor.CallerRunsPolicy()`    直接调用run方法并且阻塞执行

3、`ThreadPoolExecutor.DiscardPolicy()`   直接丢弃后来的任务

4、`ThreadPoolExecutor.DiscardOldestPolicy()`  丢弃在队列中队首的任务

当然可以自己继承RejectedExecutionHandler来写拒绝策略



## Java注解

注解分为三类：

- 标准 Annotation：包括 Override, Deprecated, SuppressWarnings，是java自带的几个注解，他们由编译器来识别，不会进行编译，不影响代码运行。

- 元 Annotation：`@Retention`, `@Target`, `@Inherited`, `@Documented`，它们是用来定义 Annotation 的 Annotation。也就是当我们要自定义注解时，需要使用它们。

  > @Target用来表示这个注解可以使用在哪些地方。比如：类、方法、属性、接口等等。这里ElementType.TYPE 表示这个注解可以用来修饰：Class, interface or enum declaration。

- 自定义 Annotation：根据需要，自定义的Annotation。



- @Retention(RetentionPolicy.SOURCE)：该注解仅用于在源码阶段时处理，但在编译成class文件或运行中以后，APT就没有办法对他进行处理了。
- @Retention(RetentionPolicy.CLASS)：该注解用于源码、类文件阶段。就是我们编写java文件和编译后产生的class文件。
- @Retention(RetentionPolicy.RUNTIME)：该注解用于源码、类文件和运行时阶段。



- 运行时注解：在代码中通过注解进行标记，运行时通过**反射**寻找标记进行某种处理，因此会影响性能。
- 编译时注解：可以理解成**代码生成**。在编译时对注解做处理，通过注解，获取必要信息，在项目中生成代码，运行时调用，和直接运行手写代码没有任何区别。而更准确的叫法：**APT - Annotation Processing Tool**



## 多线程断点续传原理

断点续传原理：在本地下载过程中要使用数据库实时存储到底存储到文件的哪个位置了，这样点击开始继续传递时，才能通过HTTP的GET请求中的`setRequestProperty()`方法可以告诉服务器，数据从哪里开始，到哪里结束。同时在本地的文件写入时，`RandomAccessFile`的`seek()`方法也支持在文件中的任意位置进行写入操作。

而多线程断点续传便是在单线程的断点续传上延伸的，而多线程断点续传是把整个文件分割成几个部分，每个部分由一条线程执行下载，而每一条下载线程都要实现断点续传功能。为了实现文件分割功能，我们需要使用到HttpURLConnection的另外一个方法：`public int getContentLength()`

当请求成功时，可以通过该方法获取到文件的总长度。

`每一条线程下载大小 = fileLength / THREAD_NUM`



## OKhttp处理缓存

OkHttp默认对Http缓存进行了支持，只要服务端返回的Response中含有缓存策略，OkHttp就会通过CacheInterceptor拦截器对其进行缓存。但是OkHttp默认情况下构造的HTTP请求中并没有加Cache-Control，即便服务器支持了，我们还是不能正常使用缓存数据。所以需要对OkHttp的缓存过程进行干预，使其满足我们的需求。



## 横竖屏切换时Activity生命周期

1.如果自己没有配置`android:ConfigChanges`，这时默认让系统处理，就会重建Activity，此时Activity的生命周期会走一遍

onPause->onSaveInstanceState->onStop->onDestroy->onCreate->onStart->onRestoreInstanceState->onResume

在onStop之前回调`onSaveInstanceState`保存数据，在重新创建Activity的时候在onStart之后回调`onRestoreInstanceState`

2.如果设置  `android:configChanges="orientation|keyboardHidden|screenSize">`，此时Activity的生命周期不会重走一遍，Activity不会重建，只会回调`onConfigurationChanged`方法。



## 进程保活

- 提供进程优先级，降低进程被杀死的概率

方法一：监控手机锁屏解锁事件，在屏幕锁屏时启动1个像素的Activity，在用户解锁时将 Activity 销毁掉。

方法二：启动前台service，可以使用startForeground()将service放到前台状态。这样在低内存时被kill的几率会低一些。

方法三：提升service优先级：在AndroidManifest.xml文件中对于intent-filter可以通过`android:priority = "1000"`这个属性设置最高优先级，1000是最高值，如果数字越小则优先级越低，同时适用于广播。

- 在进程被杀死后，进行拉活

方法一：注册高频率广播接收器，唤起进程。如网络变化，解锁屏幕，开机等

方法二：双进程相互唤起。

方法三：依靠系统唤起。

方法四：onDestroy方法里重启service：service+broadcast 方式，就是当service走onDestory的时候，发送一个自定义的广播，当收到广播的时候，重新启动service；

- 依靠第三方

根据终端不同，在小米手机（包括 MIUI）接入小米推送、华为手机接入华为推送；其他手机可以考虑接入腾讯信鸽或极光推送与小米推送做 A/B Test。



## Context相关问题

- Activity和Service以及Application的Context是不一样的,**Activity继承自ContextThemeWraper**.其他的继承自ContextWrapper.
- **每一个Activity和Service以及Application的Context都是一个新的ContextImpl对象**
- getApplication()用来获取Application实例的，但是这个方法只有在Activity和Service中才能调用的到。那么也许在绝大多数情况下我们都是在Activity或者Service中使用Application的，但是如果在一些其它的场景，比如BroadcastReceiver中也想获得Application的实例，这时就可以借助getApplicationContext()方法.
  getApplicationContext()比getApplication()方法的作用域会更广一些，任何一个Context的实例，只要调用getApplicationContext()方法都可以拿到我们的Application对象。
- Context的数量等于Activity的个数 + Service的个数 + 1，这个1为Application.
  那Broadcast Receiver，Content Provider呢？Broadcast Receiver，Content Provider并不是Context的子类，他们所持有的Context都是其他地方传过去的，所以并不计入Context总数。




## 非UI线程更新UI

真要实现其实也是可以的，当访问UI时，ViewRootImpl会调用`checkThread`方法去检查当前访问UI的线程是哪个，如果不是UI线程则会抛出异常
执行onCreate方法的那个时候ViewRootImpl还没创建，无法去检查当前线程.ViewRootImpl的创建在onResume方法回调之后.

非UI线程是可以刷新UI的，前提是它要拥有自己的ViewRoot,即更新UI的线程和创建ViewRoot是同一个,或者在执行`checkThread()`前更新UI.



## 应用被强杀

判断方法：在Application中定义一个static常量，赋值为-1，在欢迎界面改为0，如果被强杀，application重新初始化，在父类Activity判断该常量的值。

解决方法：如果在每一个Activity的onCreate里判断是否被强杀，冗余了，封装到Activity的父类中，如果被强杀，跳转回主界面，如果没有被强杀，执行Activity的初始化操作，给主界面传递intent参数，主界面会调用onNewIntent方法，在onNewIntent跳转到欢迎页面，重新来一遍流程。



## ANR问题

1. KeyDispatchTimeout(5 seconds) --主要是类型按键或触摸事件在特定时间内无响应
2. BroadcastTimeout(10 seconds) --BroadcastReceiver在特定时间内无法处理完成
3. ServiceTimeout(20 secends) --小概率事件 Service在特定的时间内无法处理完成

如何避免：

1. UI线程尽量只做跟UI相关的工作
2. 耗时的操作(比如数据库操作，I/O，连接网络或者别的有可能阻塞UI线程的操作)把它放在单独的线程处理
3. 尽量用Handler来处理UI线程和别的线程之间的交互

排查方法：

1. 首先分析log
2. 从trace.txt文件查看调用stack，adb pull data/anr/traces.txt ./mytraces.txt



## MultiDex原理

单个dex里面不能有超过65536个方法。原因在于Android会把每一个类的方法id检索起来，存在一个链表结构里面。但是这个**链表的长度是用一个short类型**来保存的，short占两个字节（保存$-2^{15}​$~$2^{15}-1​$，即-32768~32767），最大保存的数量就是65536。新版本的Android系统中修复了这个问题。

Dex拆分步骤分为：

1）自动扫描整个工程代码得到main-dex-list； 

2）根据main-dex-list对整个工程编译后的所有class进行拆分，将主、从dex的class文件分开； 

3）用dx工具对主、从dex的class文件分别打包成 .dex文件，并放在apk的合适目录。

加载方式：

从 apk 中提取出所有的从 dex（classes2.dex，classes3.dex，…），通过反射将classes2.dex等注入到当前ClassLoader的 DexPathList 的 Element[] 数组的后面。这个过程要尽可能的早，所以一般是在Application的attachBaseContext()方法中。一些热修复技术，就是通过一定的方式把修复后的dex插入到DexPathList的Element[]数组**前面**，实现了修复后的class抢先加载。



## 约瑟夫环

问题描述：n个人（编号1~n)，从1开始报数，报到m的退出，剩下的人继续从1开始报数。求胜利者的编号。

解法一：建立一个有N个元素的循环链表，然后从链表头开始遍历并记数，如果计数i==m(i初始为1)踢出元素，继续循环，当当前元素与下一元素相同时退出循环

解法二：转换为数学问题

递推公式

```
f[1]=0;

f[i]=(f[i-1]+m)%i;  (i>1)
```

```java
public static int lastRemaining(int n, int m){
    if(n < 1 || m < 1){
        return -1;
    }
    int last = 0;
    for(int i = 2; i <= n; i++){
        last = (last + m) % i;
    }
    return last;
}
```



## 单例模式

常见的单例模式写法有懒汉式和饿汉式两种写法，但懒汉式在每次调用getInstance都进行同步，造成不必要的同步开销。可以将其改造成Double Check Lock（DCL）

```Java
public class Singleton {
    private static Singleton sInstance = null;
    private Singleton(){}

    public static Singleton getInstance(){
        if (sInstance == null){
            synchronized (Singleton.class){
                if (sInstance == null){
                    sInstance = new Singleton();
                }
            }
        }
        return sInstance;
    }
}
```

第一层判空主要是**为了避免不必要的同步**，也就是说单例对象初始化后调用getInstance不进行同步锁，也就是为了解决懒汉式存在的最大问题。

另外`sInstance = new Singleton()`这句代码最终会编译成多条汇编指令，大致做了3件事件：

1. 给Singleton的实例分配内存
2. 调用Singleton()的构造函数，初始化成员字段
3. 将sInstance对象指向分配的内存空间（此时sInstance就不是null了）



由于Java编译器允许处理器乱序执行，2和3的先后顺序不能保证，也就导致了DCL失效，修复方法是将sInstance的定义改为

```java
private volatile static Singleton sInstance = null;
```

就可以保证sInstance对象每次都是从主内存中读取。

除了上述方法，还有静态内部类单例模式、枚举单例、使用容器实现单例等等。



## Volatile

被volatile修饰的共享变量，就具有了以下两点特性：

1 . 保证了不同线程对该变量操作的内存可见性;

2 . 禁止指令重排序

更多细节如JMM可参考[这篇文章](https://juejin.im/post/5a2b53b7f265da432a7b821c)