# 1 内存区域与内存溢出异常

## 1.1 运行时数据区域
Java 虚拟机所管理得内存包括以下几个运行时数据区域，如图1所示。
![图1 Java 虚拟机运行时数据区](https://upload-images.jianshu.io/upload_images/24520697-56b2235ea13ccad6.PNG?imageMogr2/auto-orient/strip%7CimageView2/2/w/500)
### 程序计数器(Program Counter Register)
程序计数器是一块较小的内存空间，它可以看作是当前执行的字节码指令的行号指示器。为了线程切换后能恢复到正确的执行位置，每条线程都需要一个独立的程序计数器，属于**线程私有的**。此内存区域是唯一一个在《Java虚拟机规范》中没有规定任何 OutOfMemoryError 情况的区域。
### 虚拟机栈(VM Stack)
虚拟机栈也是**线程私有**的，它的生命周期与线程一致。每个方法被调用时，Java 虚拟机都会同步创建一个**栈帧(Stack Frame)**来存储**局部变量表、操作数栈、动态连接、方法出口**等信息。

局部变量表用于存放**编译期间**可知的**基本数据类型**、**对象引用**、**returnAddress**(指向了一个字节码指令的地址)。这些数据类型存储基本单位用**局部变量槽**(Slot)来表示，局部变量槽占用多大的空间(32位、64位或更多)需要看具体的虚拟机实现。局部变量表里有多少个局部变量槽在编译期间就可确定。

如果线程请求的栈深度大于虚拟机规定深度，将抛出 **StackOverflowError** 异常；如果 Java 虚拟机栈容量可以动态扩展，当栈容量扩展时无法申请到足够的内存会抛出 **OutOfMemoryError** 异常。
### 本地方法栈(Native Method Stack)
该区域和虚拟机栈基本一致，不同的是本地方法栈是为**本地方法**服务的。和虚拟机栈一样，本地方法栈也会在栈深度溢出和栈扩展失败时分别抛出 **StackOverflowError** 和 **OutOfMemoryError** 异常。
### Java 堆(Heap)
Java 堆是虚拟机管理的**最大**的一块内存，被所有线程共享，在虚拟机启动时创建。此区域的唯一目的是存放对象实例，Java 里几乎所有的对象都在这儿分配内存。由于**栈上分配**、**标量替换**优化手段的出现，一切Java 对象实例都在堆中分配已经变得不是那么绝对了。Java 堆在**物理上可以不连续**，但在**逻辑上是连续**的。为了提升对象分配时的效率，Java 堆中可以划分出多个线程私有的**分配缓冲区**(Thread Local Allocation Buffer, TLAB)。

Java 堆是最主要的垃圾回收区域，为了便于垃圾回收，很多垃圾收集器将 Java 堆划分为这几个区域：**新生代、老年代、永久代、Eden 区、From Survivor 空间、To Survivor 区**。

如果在 Java 堆中没有内存完成实例分配，并且堆也无法再扩展时，Java 虚拟机将会抛出 **OutOfMemoryError** 异常。
### 1.5 方法区(Method Area)
方法区与Java堆一样，是**线程共享**的区域，用于存储已经被虚拟机加载的**类型信息、常量、静态变量、即时编译器编译后的代码缓存**等数据。这个区域的内存回收目标主要是针对**常量池**的回收和对类型的卸载。若方法区无法满足新的内存分配需求时，会抛出 **OutOfMemoryError** 异常。
### 1.6 运行时常量池(Runtime Constant Pool)
运行时常量池是方法区的一部分。Class 文件除了有类的版本、字段、方法、接口等描述信息外，还有一项信息是**常量池表(Constant Pool Table)**，用于存放编译期间生成的各种**字面量**和**符号引用**，这部分内容将再加载后存放在方法区的运行时常量池中。运行时常量池除了保存 Class 文件中描述的符号引用外，还会把符号引用翻译出来的直接引用也存储在运行时常量池中。运行时常量池还具有**动态性**，并非只有 Class 文件中常量池的内容才能进入运行时常量池，运行期间也可以将新的常量放入池中，如 String 类的 intern() 方法。

当常量池无法申请到内存时会抛出 OutOfMemoryError 异常。
## 1.2 HotSpot 虚拟机对象
下面将以 HotSpot 虚拟机为例说明对象在堆中是如和分配的。
### 对象的创建
有 2 种方式来为对象分配内存：**指针碰撞(Bump The Pointer)、空闲列表(Free List)**。选择哪种方式要看 Java 堆是否规整。
- 指针碰撞：用过的内存放在一边，空闲内存放在另一边，中间放一个指针作为分界点，那么为对象分配内存实际上就是将指针移向空闲内存。

- 空闲列表：虚拟机维护一张表，该表记录了哪些内存可用。这种方法在 Java 堆不规则时使用。

内存分配完之后，虚拟机会为将分配的内存(不包括对象头)初始化为 **0 值**。接下来，虚拟机会对对象头做必要的设置：**属于哪个类的实例，如何找到类的元数据信息，对象的哈希码、对象的 GC 分代年龄**。

这时候，从虚拟机的角度来看，对象已经产生。但是从 Java 层面来看，对象创建才刚刚开始，即 Class 文件种的 <init> () 方法还没有执行。一般来说，虚拟机遇到 new 指令后会接着执行 <init>() 方法，按照程序员的意图对对象进行初始化。
### 对象的内存布局
对象在内存中的布局分为 3 个部分：**对象头(Header)、实例数据(Instance Data)和对齐填充(Padding)**。
- 对象头：包含两类信息。一类是**对象自身的运行是数据(官方称之为 Mark Word)**，如哈希码、GC 分代年龄、锁状态标志、线程持有锁、偏向线程 ID、偏向时间戳等。另一类是**类型指针**,并不是所有虚拟机都必须实现在对象上上保留类型指针，换句话说，**查找对象的元数据信息并不一定要经过对象本身**。如果对象是数组，那么对象头还会包括**数组长度**信息。

- 实例数据：对象正真存储的**有效信息**，即程序代码里面所定义的各种类型的**字段内容**，无论是从父类继承下来的，还是在子类中定义的字段都必须记录起来。
- 对齐填充：并不是必然存在的，也没有什么特别的意义，仅仅起着占位的作用。
###  对象的访问定位
主流的访问方式有**句柄**和**直接指针**两种方式。
- 句柄：Java 堆中划分出一块内存作为句柄池，reference 中存储的就是对象的**句柄地址**，而句柄中包含了**对象实例数据**和**类型数据**，如图 2 所示。
- 直接指针：reference 存储的直接就是对象地址，如图 3 所示。

![图 2 通过句柄访问对象](https://upload-images.jianshu.io/upload_images/24520697-e2cd401c83b60a7f.PNG?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![图 3 通过直接指针访问对象](https://upload-images.jianshu.io/upload_images/24520697-e9c2a8bba8dc611d.PNG?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这两种访问方式有着各自的优势，句柄访问方式最大的好处就是 reference 中存储的是稳定的句柄地址，在对象被移动时只需要改变句柄中的实例数据指针。直接指针访问方式最大的好处就是速度快。
## 1.3 OutOfMemoryError 异常
除了**程序计数器**外，虚拟机的其他几个运行时区域都会发生 **OOM 异常**。
### Java 堆溢出
将堆的大小设置为 20MB ，不可扩展(最小值 **-Xms** 和 最大值 **-Xmx** 设置为一样即可避免堆自动扩展)。参数 **-XX:+HeapDumpOnOutOfMemoryError** 可以让虚拟机在出现 OOM 异常时 Dump 出当前的内存堆转储快照。
```java
/**
* VM Args: -Xms20m -Xmx20m -XX:+HeapDumpOnOutOfMemoryError
*/
public class HeapOOM {
    static class OOMObject{
    }

    public static void main(String[] args) {
        List<OOMObject> list = new ArrayList<OOMObject>();
        while(true)
            list.add(new OOMObject());
    }
}
```
要解决这个内存区域的异常，常规的处理方法是首先通过内存映射分析工具对Dump 出来的堆转储快照进行分析。确定导致 OOM 异常的对象是否是必要的，也就是判断是**内存泄漏(Memory Leak)**还是**内存溢出(Memory Overflow)**。

如果是内存泄漏，可进一步通过工具查看泄漏对象到 GC Roots 的引用链，找到泄漏对象是通过怎样的引用路径、与哪些 GC Roots 相关联，才导致垃圾收集器无法回收它们，根据泄漏对象的类型信息以及它到 GC Roots 引用链的信息，一般可以比较准确地定位到这些对象创建的位置，进而找出产生内存泄漏的代码的具体位置。

如果不是内存泄漏，换句话说就是内存中的对象确实都是必须存活的，那就应当检查 Java 虚拟机的堆参数（-Xmx与-Xms）设置，与机器的内存对比，看看是否还有向上调整的空间。再从代码上检查是否存在某些对象生命周期过长、持有状态时间过长、存储结构设计不合理等情况，尽量减少程序运行期的内存消耗。

### 虚拟机栈和本地方法栈溢出
由于 HotSpot 虚拟机中并不区分虚拟机栈和本地方法栈，因此栈容量只能由 **-Xss** 参数来设定。关于虚拟机栈和本地方法栈，有两种异常：
1. 如果线程请求的栈深度大于虚拟机所允许的最大栈深度，将抛出 StackOverflowError 异常。
2. 如果虚拟机的栈内存允许动态扩展，当扩展栈容量无法申请到足够的内存时，会抛出 OutOfMemoryError 异常。

HotSopt 虚拟机不支持栈容量动态扩展，所以除非因在创建线程时而无法申请到内存而抛出 OutOfMemoryError 异常，否则不会因扩展而导致 OOM 异常，只会因栈容量无法容纳新的栈帧而抛 StackOverflowError 异常。
### 方法区和运行时常量池溢出
HotSpot 虚拟机从 JDK7 开始逐步“去除永久代”的计划，并在 JDK8 中完全使用**元空间**来代替永久代。自 JDK7 开始，原本放在永久代中的字符串常量池移至 Java 堆中。

String:intern() 是一个本地方法，它的作用是如果字符串常量池中已经包含一个等于此 String 对象的字符串，则返回池中该字符串对象的引用。否则会将 String 对象中的字符串添加到字符串常量池中，并返回引用。在JDK 6或更早之前的HotSpot虚拟机中，常量池都是分配在永久代中，我们可以通过-XX：PermSize和-XX：MaxPermSize限制永久代的大小，即可间接限制其中常量池的容量。使用 JDK6 运行如下代码

![](https://upload-images.jianshu.io/upload_images/24520697-d218f292a3c8a803.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

运行结果

![](https://upload-images.jianshu.io/upload_images/24520697-1c1e5518a1c79fa8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


运行结果中可以看到，运行时常量池溢出时，在OutOfMemoryError异常后面跟随的提示信息是“PermGen space”，说明运行时常量池的确是属于方法区（即JDK 6的HotSpot虚拟机中的永久代）的一部分。

### 本机直接内存溢出
-XX:MaxDirectMemorySize 参数设置直接内存的大小，直接内存溢出的一个明显特征是 Heap Dump 文件看不出有什么异常情况，如果内存溢出之后产生的 Dump 文件很小，而程序中又使用了 DirectMemory，那有很大概率是直接内存溢出。

# 2 垃圾收集器与内存分配策略

## 2.1 如何判断对象已死

### 引用计数算法

该算法是这样的：在对象中添加一个引用计数器，每当有一个地方引用它时，计数器加 1，引用失效时，计数器减 1，任何时刻计数器为 0 的对象就是不可能再被使用。但是引用计数算法无法解决对象相互引用的问题。例如有两个相互引用的对象，并且它们不会再被使用了，但是由于这两个对象还被对方引用着，因此无法对它们进行回收。

### 可达性分析算法

该算法的基本思路就是通过一系列称为 **GC Roots** 的根对象作为起始节点集，从这些节点开始，根据引用关系向下搜索，搜索过程所走过的路径称为**引用链**，如果某个对象到 GC Roots 间没有任何引用链，则证明此对象是不可能再被使用的。

如图所示，对象 object 5、object 6、object 7 虽然互有关联，但是它们到 GC Roots 是不可达的，因此它们将会被判定为可回收的对象。

![](https://upload-images.jianshu.io/upload_images/24520697-6942bd8abb0b3c59.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可作为GC Roots的对象包括以下几种：

- 在虚拟机栈（栈帧中的本地变量表）中引用的对象
- 在方法区中类静态属性引用的对象
- 在方法区中常量引用的对象
- 在本地方法栈中 JNI（即通常所说的 Native 方法）引用的对象。
- Java虚拟机内部的引用，如基本数据类型对应的 Class 对象，一些常驻的异常对象（比如
  NullPointExcepiton、OutOfMemoryError）等，还有系统类加载器。
- 所有被同步锁（synchronized关键字）持有的对象
- 反映Java虚拟机内部情况的JMXBean、JVMTI中注册的回调、本地代码缓存等。

除了这些固定的 GC Roots 集合以外，根据用户所选用的垃圾收集器以及当前回收的内存区域不
同，还可以有其他对象“临时性”地加入，共同构成完整 GC Roots 集合。比如，在针对新生代的垃圾收集时，该区域里的对象完全有可能被位于堆中其他区域的对象所引用，这时候就需要将这些关联区域的对象也一并加入 GC Roots 集合中去。

### Java 中的引用

JDK1.2 之后，Java 对引用的概念进行了扩充，将引用分为**强引用**，**软引用**，**弱引用**，**虚引用**，这 4 种引用强度逐渐减弱。

- 强引用：类似`Object obj=new Object()`这种引用关系。无论任何情况下，只要强引用关系还存在，垃圾收集器就永远不会回收掉被引用的对象。
- 软引用：用来描述一些还有用，但非必须的对象。只被软引用关联着的对象，在系统将要发生内存溢出异常前，会把这些对象列进回收范围之中进行第二次回收，如果这次回收还没有足够的内存，才会抛出内存溢出异常。
- 弱引用：也是用来描述那些非必须对象，但是它的强度比软引用更弱一些。被弱引用关联的对象只能生存到下一次垃圾收集发生为止。当垃圾收集器开始工作，无论当前内存是否足够，都会回收掉只被弱引用关联的对象。
- 虚引用：最弱的一种引用关系。一个对象是否有虚引用的存在，完全不会对其生存时间构成影响，也无法通过虚引用来取得一个对象实例。为一个对象设置虚引用关联的唯一目的只是为了能在这个对象被收集器回收时收到一个系统通知。

### 两次标记

即使在可达性分析算法中判定为不可达的对象，也不是“非死不可”的。要真正宣告一个对象死亡，至少要经历两次标记过程：如果对象在进行可达性分析后发现没有与 GC Roots 相连接的引用链，那它将会被第一次标记。筛选的条件是此对象是否有必要执行`finalize()`方法。假如对象没有覆盖`finalize()`方法，或者`finalize()`方法已经被虚拟机调用过，那么虚拟机将这两种情况都视为“没有必要执行”。

如果这个对象被判定为确有必要执行`finalize()`方法，那么该对象将会被放置在一个名为 F-Queue 的队列之中，并在稍后由一条由虚拟机自动建立的、低调度优先级的 Finalizer 线程去执行它们的`finalize()`方法。`finalize()`方法是对象逃脱死亡命运的最后一次机会，稍后收集器将对 F-Queue 中的对象进行第二次小规模的标记，如果对象要在finalize()中成功拯救自己——只要重新与引用链上的任何一个对象建立关联即可，譬如把自己（this关键字）赋值给某个类变量或者对象的成员变量，那在第二次标记时它将被移出“即将回收”的集合。

### 回收方法区条件

方法区的垃圾收集主要回收两部分内容：废弃的常量和不再使用的类型。回收废弃常量与回收
Java堆中的对象非常类似。判定一个类型是否属于“不再被使用的类”的条件就比较苛刻了。需要同时满足下面三个条件：

- 该类所有的实例都已经被回收
- 加载该类的类加载器已经被回收
- 该类对应的 java.lang.Class 对象没有在任何地方被引用

Java 虚拟机被允许对满足上述三个条件的无用类进行回收，这里说的仅仅是“被允许”，而并不是
和对象一样，没有引用了就必然会回收。

在大量使用反射、动态代理、CGLib等字节码框架，动态生成JSP以及OSGi这类频繁自定义类加载器的场景中，通常都需要Java虚拟机具备类型卸载的能力，以保证不会对方法区造成过大的内存压力。

## 2.2 垃圾收集算法

### 分代收集

部分收集(Partitial GC)指不是对整个 Java 堆的垃圾收集，其中分为：

- 新生代收集(Minor GC/Young GC)：针对新生代的垃圾收集。
- 老年代收集(Major GC/Old GC)：针对老年代的垃圾收集。
- 混合收集(Mixed GC)：收集整个新生代和部分老年代。

整堆收集(Full GC)：收集整个 Java 堆和方法区。

### 标记-清除算法

首先标记出所有要回收的对象，之后统一回收。主要有两个缺点：1）如果有大量的要回收的对象，那么就必须进行大量的标记和清除操作，执行效率随对象的增加而降低；2）垃圾回收后会产生大量的不连续的空间碎片，很难为大对象分配内存。

![](https://upload-images.jianshu.io/upload_images/24520697-4885d710cb2a0535.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 标记-复制算法

该算法将内存分为 2 块，每次只使用其中 1 块。当这个一块内存用完了，就将还存活的对象复制另一块上面，然后再将使用过的内存空间一次清理掉。

![](https://upload-images.jianshu.io/upload_images/24520697-f8b10b880c434536.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

现在商用的 Java 虚拟机大多都优先采用了这种收集算法去回收新生代。IBM 提出，新生代的对象大多数都是“朝生夕灭的”。因此又出现了一种更优的半区复制算法，称为 **Apple 式回收**。它将新生代划分为1个 **Eden** 区和 2 个 **Survivor** 区，HotSpot虚拟机划分比例为 8:1:1。每次分配内存只使用 Eden 和其中 1 块 Survivor 空间。发生垃圾收集时，将 Eden 和 Survivor 中仍然存活的对象复制到另一块 Survivor 空间上，然后直接清理掉 Eden 和 使用过的 Survivor 空间。Apple 式回收还有一个“**逃生门**"设计，当 Survivor 空间不足以容纳一次 Mininor GC 之后存活的对象时，就需要依赖其他区域(老年代)进行分配担保。

### 标记-整理算法

标记-复制算法在对象存活率较高时就要进行较多的复制操作，不适合老年代。标记-整理算法是针对**老年代**提出的，该算法首先进行标记，然后将存活的对象移到内存空间的一端，最后直接清理掉边界以外的内存。标记清除算法和标记整理算法的本质区别是前者是**非移动式**的回收算法，后者是**移动式**的。

![](https://upload-images.jianshu.io/upload_images/24520697-ec955abe36618c19.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

HotSpo t虚拟机里面关注吞吐量的 Parallel Scavenge收集器是基于标记-整理算法的，而关注延迟的 CMS 收集器则是基于标记-清除算法的。

## 2.3 HotSpot 的算法细节实现

2.1、2.2节从理论原理上介绍了常见的对象存活判定算法和垃圾收集算法，Java虚拟机实现这些算法时，必须对算法的执行效率有严格的考量，才能保证虚拟机高效运行。

### 根节点枚举

迄今为止，所有收集器在根节点枚举这一步骤时都是必须**暂停用户线程**的，根节点枚举始终还是必须在一个能保障一致性的快照中才得以进行。这里“一致性”的意思是整个枚举期间执行子系统
看起来就像被冻结在某个时间点上，不会出现分析过程中，根节点集合的对象引用关系还在不断变化的情况。若这点不能满足的话，分析结果准确性也就无法保证。这是导致垃圾收集过程必须停顿所有用户线程的其中一个重要原因。

目前主流 Java 虚拟机使用的都是准确式垃圾收集，当用户线程停顿下来之后，并不需要一个不漏地检查完所有执行上下文和全局的引用位置，虚拟机有办法直接得到哪些地方存放着对象引用的。HotSpot 是使用一组称为 OopMap 的数据结构来达到这个目的。一旦类加载动作完成的时候，HotSpot就会把对象内什么偏移量上是什么类型的数据计算出来

OopMap 记录了栈上本地变量到堆上对象的引用关系。在源代码里面每个变量都是有类型的，但是编译之后的代码就只有变量在栈上的位置了。oopMap 就是一个附加的信息，告诉你栈上哪个位置本来是个什么东西。

### 安全点

在OopMap的协助下，HotSpot可以快速准确地完成GC Roots枚举，但一个很现实的问题随之而
来：可能导致引用关系变化，或者说导致OopMap内容变化的指令非常多，如果为每一条指令都生成对应的OopMap，那将会需要大量的额外存储空间。

HotSpot 没有为每条指令都生成 OopMap，只是在“特定的位置”记录了这些信息，这些位置被称为**安全点**。安全点决定了用户程序必须执行到达安全点后才能够暂停下来开始垃圾收集。

在垃圾收集时让所有线程都跑到最近的安全点有 2 种方式：抢占式中断和主动式中断。

- 抢占式中断：不需要线程的执行代码主动去配合，在垃圾收集发生时，系统首先把所有用户线程全部中断，如果发现有用户线程中断的地方不在安全点上，就恢复这条线程执行，让它一会再重新中断，直到跑到安全点上。现在几乎没有虚拟机实现采用抢先式中断来暂停线程响应 GC 事件。
- 当垃圾收集需要中断线程的时候，不直接对线程操作，仅仅简单地设置一个标志位，各个线程执行过程时会不停地主动去轮询这个标志，一旦发现中断标志为真时就自己在最近的安全点上主动中断挂起。

### 安全区域

用户线程处于 Sleep 状态或者 Blocked 状态，这时候线程无法响应虚拟机的中断请求，不能再走到安全的地方去中断挂起自己，虚拟机也显然不可能持续等待线程重新被激活分配处理器时间。对于这种情况，就必须引入安全区域来解决。

安全区域是指能够确保在某一段代码片段之中，引用关系不会发生变化，因此，在这个区域中任
意地方开始垃圾收集都是安全的。我们也可以把安全区域看作被扩展拉伸了的安全点。

当用户线程执行到安全区域里面的代码时，首先会标识自己已经进入了安全区域，那样当这段时
间里虚拟机要发起垃圾收集时就不必去管这些已声明自己在安全区域内的线程了。当线程要离开安全区域时，它要检查虚拟机是否已经完成了根节点枚举（或者垃圾收集过程中其他需要暂停用户线程的阶段），如果完成了，那线程就当作没事发生过，继续执行；否则它就必须一直等待，直到收到可以离开安全区域的信号为止。

### 记忆集与卡表

为解决对象跨代引用所带来的问题，垃圾收集器在新生代中建立了名为**记忆集**的数据结构，用以避免把整个老年代加进 GC Roots 扫描范围。

记忆集是一种用于记录从非收集区域指向收集区域的指针集合的抽象数据结构。最简单的实现可以用非收集区域中所有含跨代引用的对象数组来实现这个数据结构，这种记录全部含跨代引用对象的实现方案，无论是空间占用还是维护成本都相当高昂。

目前最常用的一种记忆集实现形式是用一种称为**卡表**的方式去实现，它的每个记录精确到一块内存区域，该区域内有对象含有跨代指针。卡表就是记忆集的一种具体实现，它定义了记忆集的记录精度、与堆内存的映射关系等。

基于卡表的设计，通常将堆空间划分为一系列2次幂大小的卡页。卡表，用于标记卡页的状态，每个卡表项对应一个卡页。HotSpot 的卡页大小为 512 字节，卡表被实现为一个简单的字节数组，即卡表的每个标记项为 1 个字节。

一个卡页的内存中通常包含不止一个对象，只要卡页内有一个（或更多）对象的字段存在着跨代
指针，那就将对应卡表的数组元素的值标识为1，称为这个元素变脏（Dirty），没有则标识为 0。在垃圾收集发生时，只要筛选出卡表中变脏的元素，就能轻易得出哪些卡页内存块中包含跨代指针，把它们加入GC Roots中一并扫描。

![](https://upload-images.jianshu.io/upload_images/24520697-b3b77d623a42484a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 写屏障

在 HotSpot 虚拟机里是通过写屏障，这里的写屏障与解决并发乱序中的内存屏障是不一样的。写屏障可以看作在虚拟机层面对“引用类型字段赋值”这个动作的 AOP 切面，在引用对象赋值时会产生一个环形通知，供程序执行额外的动作，也就是说赋值的前后都在写屏障的覆盖范畴内。在赋值前的部分的写屏障叫作写前屏障，在赋值后的则叫作写后屏![](https://upload-images.jianshu.io/upload_images/24520697-4bcad3db048e09ba.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

应用写屏障后，虚拟机就会为所有赋值操作生成相应的指令，一旦收集器在写屏障中增加了更新卡表操作，无论更新的是不是老年代对新生代对象的引用，每次只要对引用进行更新，就会产生额外的开销，不过这个开销与Minor GC时扫描整个老年代的代价相比还是低得多的。

### 并发的可达性分析

在 JVM 进行可达性分析时，一般其他的 java 用户线程是没有停止的，它们还在辛勤的劳动。那么此时如果用户线程改变了引用关系。所谓并发，在于用户线程和垃圾回收线程同时运行。首先 JVM 会将所有可以作为 GC Roots 的对象设置成 GC Roots，这一过程是需要 STW 的（stop the world，该过程要冻结用户线程的运行，不过这一过程耗时很短，几乎可以忽略不计）。之后便是要开始往下扫描了。我们引入三色标记：

- 白色，表示对象尚未被垃圾收集器访问过
- 黑色，表示对象已经访问过，并且其所有引用的对象也访问过（即该节点被访问过，该节点的所有子节点也被访问过）
- 灰色，表示该对象被访问过，但是其所有引用的对象都没有被访问过（即该节点被访问过，该节点的所有子节点却没有被访问过）

![](https://upload-images.jianshu.io/upload_images/24520697-32c5dd18669b4e6b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

出现这个问题的原因在于二个条件：

- 一是添加了一个新的引用
- 二是删除了旧的引用

对于破坏添加新引用这个条件，我们称之为增量更新。黑色对象插入新的指向白色对象的引用关系时，就将这个新插入的引用记录下来，等我们的扫描（遍历）结束之后，在以我们记录中的黑色对象为 GC Roots，再扫描一遍。

对于破坏删除旧引用这个条件，我们称之为原始快照。其做法是：当灰色对象要删除指向白色对象的引用关系时，就将这个要删除的引用记录下来，等本次扫描结束之后，再将这些记录过的引用关系中的灰色对象为根，再重新扫描一次。

## 2.4 垃圾收集器

### Serial 收集器

这个收集器是一个单线程工作的新生代收集器，但它的“单线程”的意义并不仅仅是说明它只会使用一个处理器或一条收集线程去完成垃圾收集工作，更重要的是强调在它进行垃圾收集时，必须暂停其他所有工作线程，直到它收集结束。

![](https://upload-images.jianshu.io/upload_images/24520697-7b8c4674f64e2d55.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### ParNew 收集器

ParNew 收集器实质上是 Serial 收集器的多线程并行版本，与 Serial 收集器相比并没有太多创新之处。除了 Serial 收集器外，目前只有它能与 CMS 收集器配合工作。

![](https://upload-images.jianshu.io/upload_images/24520697-52e889072cf57316.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### Parallel Scavenge收集器

Parallel Scavenge 收集器也是一款新生代收集器，它同样是基于标记-复制算法实现的收集器，也是能够并行收集的多线程收集器。Parallel Scavenge 收集器的目标是达到一个可控制的吞吐
量。

![](https://upload-images.jianshu.io/upload_images/24520697-bb43749c03681085.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### Serial Old 收集器

Serial Old 是 Serial 收集器的老年代版本，它同样是一个单线程收集器，使用标记-整理算法。

Serial/Serial Old收集器运行示意图：

![](https://upload-images.jianshu.io/upload_images/24520697-3f02b70152b1973f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### Parallel Old 收集器

Parallel Old是Parallel Scavenge收集器的老年代版本，支持多线程并发收集，基于标记-整理算法实现。

Parallel Scavenge/Parallel Old收集器运行示意图：

![](https://upload-images.jianshu.io/upload_images/24520697-ba7625e153dc4132.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### CMS 收集器

这款收集器是 HotSpot 虚拟机中第一款真正意义上支持并发的垃圾收集器，它首次实现了让垃圾收集线程与用户线程（基本上）同时工作。回收区域在老年代。

回收过程分为四个步骤：

1. 初始标记
2. 并发标记
3. 重新标记
4. 并发清除

其中初始标记、重新标记这两个步骤仍然需要“Stop The World”。初始标记仅仅只是标记一下GC
Roots能直接关联到的对象，速度很快；并发标记阶段就是从GC Roots的直接关联对象开始遍历整个对象图的过程，这个过程耗时较长但是不需要停顿用户线程，可以与垃圾收集线程一起并发运行；而重新标记阶段则是为了修正并发标记期间，因用户程序继续运作而导致标记产生变动的那一部分对象的标记记录。最后是并发清除阶段，清理删除掉标记阶段判断的已经死亡的对象，由于不需要移动存活对象，所以这个阶段也是可以与用户线程同时并发的。

![](https://upload-images.jianshu.io/upload_images/24520697-33da09dec398fed8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### Garbage First 收集器

G1不再坚持固定大小以及固定数量的分代区域划分，而是把连续的Java堆划分为多个大小相等的独立区域（Region），每一个Region都可以根据需要，扮演新生代的Eden空间、Survivor空间，或者老年代空间。

Region中还有一类特殊的 Humongous 区域，专门用来存储大对象。G1 认为只要大小超过了一个 Region 容量一半的对象即可判定为大对象。每个 Region 的大小可以通过参数 -XX：G1HeapRegionSize 设定，取值范围为1MB～32MB，且应为2的N次幂。而对于那些超过了整个Region容量的超级大对象，将会被存放在N个连续的Humongous Region之中，G1的大多数行为都把Humongous Region作为老年代的一部分来进行看待。

### 未完待续

## 2.5 低延迟垃圾收集器

## 2.6 选择合适的垃圾收集器

## 2.7 实战：内存分配与回收策略

对象的内存分配，从概念上讲，应该都是在堆上分配（而实际上也有可能经过即时编译后被拆散
为标量类型并间接地在栈上分配）。在经典分代的设计下，新生对象通常会分配在新生代中，少数情况下（例如对象大小超过一定阈值）也可能会直接分配在老年代。

### 对象优先在 Eden 分配

大多数情况下，对象在新生代Eden区中分配。当Eden区没有足够空间进行分配时，虚拟机将发起一次Minor GC。

HotSpot虚拟机提供了-XX：+PrintGCDetails这个收集器日志参数，告诉虚拟机在发生垃圾收集行为时打印内存回收日志。

在下面代码`testAllocation()`方法中，尝试分配三个2MB大小和一个4MB大小的对象，在运行时通过-Xms20M、-Xmx20M、-Xmn10M这三个参数限制了Java堆大小为20MB，不可扩展，其中10MB分配给新生代，剩下的10MB分配给老年代。-XX：Survivor-Ratio=8决定了新生代中Eden区与一个Survivor区的空间比例是8:1。

```java
private static final int _1MB = 1024 * 1024;
//VM参数：-verbose:gc -Xms20M -Xmx20M -Xmn10M -XX:+PrintGCDetails -XX:SurvivorRatio=8
public static void testAllocation() {
    byte[] allocation, allocation2, allocation3, allocation4;
    allocation1 = new byte[2 * 1_MB];
    allocation2 = new byte[2 * 1_MB];
    allocation3 = new byte[2 * 1_MB];
    // 给 allocation4 分配内存时会出现一次 Minor GC
    allocation4 = new byte[2 * 1_MB];
}
```

给 allocation4 分配内存时会出现一次 Minor GC。。产生这次垃圾收集的原因是为 allocation4分配内存时，发现 Eden 已经被占用了 6MB，剩余空间已不足以分配 allocation4 所需的4MB内存，因此发生Minor GC。垃圾收集期间虚拟机又发现已有的三个 2MB 且还存活的对象全部无法放入Survivor空间（Survivor空间只有1MB大小），所以只好通过分配担保机制提前转移到老年代去。

这次收集结束后，4MB 的 allocation4 对象顺利分配在 Eden 中。因此程序执行完的结果是 Eden 占用 4MB（被allocation4占用），Survivor空闲，老年代被占用 6MB（被allocation1、2、3占用）。

### 大对象直接进入老年代

大对象就是指需要大量连续内存空间的Java对象，最典型的大对象便是那种很长的字符串，或者元素数量很庞大的数组。在Java虚拟机中要避免大对象的原因是，在分配空间时，它容易导致内存明明还有不少空间时就提前触发垃圾收集，以获取足够的连续空间才能安置好它们，而当复制对象时，大对象就意味着高额的内存复制开销。HotSpot 虚拟机提供了 -XX：PretenureSizeThreshold 参数，指定大于该设置值的对象直接在老年代分配。

执行下面代码的`testPretenureSizeThreshold()`方法后，Eden空间几乎没有被使用，而老年代的 10MB 空间被使用了 40%，也就是 4MB 的 allocation 对象直接就分配在老年代中，是因为 -XX：PretenureSizeThreshold 被设置为3MB，超过3MB的对象都会直接在老年代进行分配。

```java
private static final int _1MB = 1024 * 1024;
// VM参数：-verbose:gc -Xms20M -Xmx20M -Xmn10M -XX:+PrintGCDetails -XX:SurvivorRatio=8
// -XX:PretenureSizeThreshold=3145728
public static void testPretenureSizeThreshold() {
    byte[] allocation;
    // 直接分配在老年代中
    allocation = new byte[4 * _1MB];
}
```

> 注意：-XX：PretenureSizeThreshold 参数只对 Serial 和 ParNew 两款新生代收集器有效，HotSpot 的其他新生代收集器，如 Parallel Scavenge 并不支持这个参数。如果必须使用此参数进行调优，可考虑 ParNew 加 CMS 的收集器组合。

### 长期存活的对象进入老年代

虚拟机给每个对象定义了一个对象年龄计数器，存储在对象头中。对象通常在Eden区里诞生，如果经过第一次 Minor GC 后仍然存活，并且能被 Survivor 容纳的话，该对象会被移动到Survivor 空间中，并且将其对象年龄设为 1 岁。对象在 Survivor 区中每熬过一次 Minor GC，年龄就增加 1 岁，当它的年龄增加到一定程度（默认为15），就会被晋升到老年代中。对象晋升老年代的年龄阈值，可以通过参数 -XX：MaxTenuringThreshold 设置。

### 动态对象年龄判定

HotSpot 虚拟机并不是永远要求对象的年龄必须达到阈值才能晋升老年代，如果在 Survivor 空间中相同年龄所有对象大小的总和大于 Survivor 空间的一半，年龄大于或等于该年龄的对象就可以直接进入老年代。

执行下面代码，并将设置-XX：MaxTenuring-Threshold=15，发现运行结果中 Survivor 占用仍然为 0%，而老年代比预期增加了 6%，也就是说 allocation1、allocation2 对象都直接进入了老年代，并没有等到 15 岁的临界年龄。因为这两个对象加起来已经到达了 512KB，并且它们是同年龄的，满足同年对象达到Survivor空间一半的规则。我们只要注释掉其中一个对象的 new 操作，就会发现另外一个就不会晋升到老年代了。

```java
private static final int _1MB = 1024 * 1024;
// VM参数：-verbose:gc -Xms20M -Xmx20M -Xmn10M -XX:+PrintGCDetails -XX:SurvivorRatio=8
// -XX:MaxTenuringThreshold=15
// -XX:+PrintTenuringDistribution
public static void testTenuringThreshold2() {
    byte[] allocation1, allocation2, acllocation3, allocation4;
    // allocation1+allocation2大于survivo空间一半
    allocation1 = new byte[_1MB / 4];
	allocation2 = new byte[_1MB / 4];
    allocation3 = new byte[4 * _1MB];
    allocation4 = new byte[4 * _1MB];
    allocation4 = null;
    allocation4 = new byte[4 * _1MB];
}
```

### 空间分配担保

在发生 Minor GC 之前，虚拟机必须先检查老年代最大可用的连续空间是否大于新生代所有对象总空间，如果这个条件成立，那这一次 Minor GC 可以确保是安全的。如果不成立，则虚拟机会先查看 -XX：HandlePromotionFailure 参数的设置值是否允许担保失败（Handle Promotion Failure）；如果允许，那会继续检查老年代最大可用的连续空间是否大于历次晋升到老年代对象的平均大小，如果大于，将尝试进行一次 Minor GC，尽管这次Minor GC是有风险的；如果小于，或者 -XX：HandlePromotionFailure 设置不允许冒险，那这时就要改为进行一次 Full GC。

取历史平均值来比较其实仍然是一种赌概率的解决办法，也就是说假如某次 Minor GC 存活后的对突增，远远高于历史平均值的话，依然会导致担保失败。如果出现了担保失败，那就只好老老实实地重新发起一次 Full GC，这样停顿时间就很长了。虽然担保失败时绕的圈子是最大的，但通常情况下都还是会将 -XX：HandlePromotionFailure 开关打开，避免 Full GC 过于频繁。

JDK 6 Update 24 之后的规则变为只要老年代的连续空间大于新生代对象总大小或者历次晋升的平均大小，就会进行 Minor GC，否则将进行Full GC。-XX：HandlePromotionFailure参数不会再影响到虚拟机的空间分配担保策略。

# 3 虚拟机性能监控、故障处理工具

## 3.1 基础故障处理工具

### jps：虚拟机进程状况工具

可以列出正在运行的虚拟机进程，并显示虚拟机执行主类（Main Class，main()函数所在的类）名称以及这些进程的本地虚拟机唯一 ID（LVMID，Local Virtual Machine Identifier）。

jps 命令格式：

```
jps [options] [hostid]
```

例如：

```
jps -l
2388 D:\Develop\glassfish\bin\..\modules\admin-cli.jar
2764 com.sun.enterprise.glassfish.bootstrap.ASMain
3788 sun.tools.jps.Jps
```

jps 主要选项：

![](https://upload-images.jianshu.io/upload_images/24520697-1398af6cb960b0d9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### jstat：虚拟机统计信息监视工具

用于监视虚拟机各种运行状态信息的命令行工具，它可以显示本地或者远程虚拟机进程中的类加载、内存、垃圾收集、即时编译等运行时数据。

命令格式：

```
jstat [ option vmid [interval[s|ms] [count]] ]
```

参数interval和count代表查询间隔和次数，如果省略这2个参数，说明只查询一次。假设需要每250毫秒查询一次进程2764垃圾收集状况，一共查询20次，那命令应当是：

```
jstat -gc 2764 250 20
```

选项option代表用户希望查询的虚拟机信息，主要分为三类：类加载、垃圾收集、运行期编译状况。

![](https://upload-images.jianshu.io/upload_images/24520697-09471ee6eff02d39.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### jinfo：Java 配置信息工具

jinfo 的作用是实时查看和调整虚拟机各项参数。使用 jps 命令的 -v 参数可以查看虚拟机启动时显式指定的参数列表，但如果想知道未被显式指定的参数的系统默认值，除了去找资料外，就只能使用 jinfo 的- flag 选项进行查询了。

命令格式：

```
jinfo [ option ] pid
```

执行样例：查询 CMSInitiatingOccupancyFraction 参数值

```
jinfo -flag CMSInitiatingOccupancyFraction 1444
-XX:CMSInitiatingOccupancyFraction=85
```

### jmap：Java 内存映像工具

jmap 命令用于生成堆转储快照（一般称为 heapdump 或 dump 文件）。还可以查询finalize执行队列、Java堆和方法区的详细信息，如空间使用率、当前用的是哪种收集器等。

命令格式：

```
jmap [ option ] vmid
```

![](https://upload-images.jianshu.io/upload_images/24520697-67b4b3f61c6252a2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

使用 jmap 生成 dump 文件：

```
jmap -dump:format=b,file=eclipse.bin 3500
Dumping heap to C:\Users\IcyFenix\eclipse.bin ...
Heap dump file created
```

### jhat：虚拟机堆转储快照分析工具

JDK 提供 jhat 命令与 jmap 搭配使用，来分析 jmap 生成的堆转储快照。jhat内置了一个微型的HTTP/Web服务器，生成堆转储快照的分析结果后，可以在浏览器中查看。不过实事求是地说，在实际工作中，除非手上真的没有别的工具可用，否则多数人是不会直接使用 jhat 命令来分析堆转储快照文件的，主要原因有两个方面。一是一般不会在部署应用程序的服务器上直接分析堆转储快照，即使可以这样做，也会尽量将堆转储快照文件复制到其他机器上进行分析，因为分析工作是一个耗时而且极为耗费硬件资源的过程，既然都要在其他机器上进行，就没有必要再受命令行工具的限制了。另外一个原因是 jhat 的分析功能相对来说比较简陋。

### jstack：Java 堆栈跟踪工具

jstack 命令用于生成虚拟机当前时刻的线程快照（一般称为 threaddump 或者 javacore 文件）。线程快照就是当前虚拟机内每一条线程正在执行的方法堆栈的集合，生成线程快照的目的通常是定位线程出现长时间停顿的原因，如线程间死锁、死循环、请求外部资源导致的长时间挂起等，都是导致线程长时间停顿的常见原因。

jstack命令格式：

```
jstack [ option ] vmid
```

![](https://upload-images.jianshu.io/upload_images/24520697-aa39c0a8cd8c828d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 基础工具总结

除了上述 6 种工具外，还有很多其他命令工具。

基础工具：用于支持基本的程序创建和运行

![](https://upload-images.jianshu.io/upload_images/24520697-ae15a9fd16a765f7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

安全：用于程序签名、设置安全测试等

![](https://upload-images.jianshu.io/upload_images/24520697-7c62fd18ecdf31f1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

国际化：用于创建本地语言文件

![](https://upload-images.jianshu.io/upload_images/24520697-1ed3e0f6145c7857.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

远程方法调用：用于跨Web或网络的服务交互

![](https://upload-images.jianshu.io/upload_images/24520697-e41fc2eb3fcdfdef.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Java IDL与RMI-IIOP：在JDK 11中结束了十余年的CORBA支持，这些工具不再提供

![](https://upload-images.jianshu.io/upload_images/24520697-bb28afd1a96d842c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

部署工具：用于程序打包、发布和部署（见表4-10）

![](https://upload-images.jianshu.io/upload_images/24520697-f62750bd1bba69ca.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Java Web Start

![](https://upload-images.jianshu.io/upload_images/24520697-50499d03d8c08f8f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

性能监控和故障处理：用于监控分析Java虚拟机运行信息，排查问题

![](https://upload-images.jianshu.io/upload_images/24520697-6e57d3a4a20e58c3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

WebService工具：与CORBA一起在JDK 11中被移除

![](https://upload-images.jianshu.io/upload_images/24520697-7ab89a7d941de5d7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

REPL和脚本工具

![](https://upload-images.jianshu.io/upload_images/24520697-90472091b9af5b77.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 未完待续

# 4 调优案例分析与实战

# 5 类文件

## 5.1 类文件结构

任何一个Class文件都对应着唯一的一个类或接口的定义信息，但是反过来说，类或接口并不一定都得定义在文件里（譬如类或接口也可以动态生成，直接送入类加载器中）

Class文件是一组以8个字节为基础单位的二进制流，各个数据项目严格按照顺序紧凑地排列在文件之中，中间没有添加任何分隔符，当遇到需要占用8个字节以上空间的数据项时，则会按照高位在前的方式分割成若干个8个字节进行存储。

Class文件格式采用无符号数和表这两种数据类型来存储数据。

- 无符号数属于基本数据类型，以u1、u2、u4、u8来分别代表1个字节、2个字节、4个字节和8个字节的无符号数。无符号数可以用来描述数字、索引引用、数量值或者字符串的编码。
- 表是由多个无符号数或者其他表作为数据项构成的复合数据类型，为了便于区分，所有表的命名都习惯性地以“_info”结尾，整个Class文件本质上也可以视作是一张表。

类文件结构：

![](https://upload-images.jianshu.io/upload_images/24520697-b32c9bb4a2aa97a6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 魔数与 Class 文件的版本

每个 Class 文件的头 4 个字节被称为魔数，它的唯一作用是确定这个文件是否为一个能被虚拟机接受的 Class 文件，Class文件的魔数值为0xCAFEBABE。

紧接着魔数的4个字节存储的是 Class 文件的版本号：第 5 和第 6 个字节是次版本号，第 7 和第 8 个字节是主版本号。

### 常量池

紧接着主、次版本号之后的是常量池入口，常量池可以比喻为 Class 文件里的资源仓库。

由于常量池中常量的数量是不固定的，所以在常量池的入口需要放置一项 u2 类型的数据，代表常量池容量计数值，这个容量计数是从 1 而不是 0 开始的。第 0 项常量空出来是有特殊考虑的，这样做的目的在于，如果后面某些指向常量池的索引值的数据在特定情况下需要表达“不引用任何一个常量池项目”的含义，可以把索引值设置为0来表示。

常量池中主要存放两大类常量：字面量和符号引用。字面量比较接近于 Java 语言层面的常量概念，如文本字符串、被声明为 final 的常量值等。而符号引用则属于编译原理方面的概念，主要包括下面几类常量：

- 被模块导出或者开放的包
- 类和接口的全限定名
- 字段的名称和描述符
- 方法的名称和描述符
- 方法句柄和方法类型
- 动态调用点和动态常量

在 Class 文件中不会保存各个方法、字段最终在内存中的布局信息，这些字段、方法的符号引用不经过虚拟机在运行期转换的话是无法得到真正的内存入口地址，也就无法直接被虚拟机使用的。当虚拟机做类加载时，将会从常量池获得对应的符号引用，再在类创建时或运行时解析、翻译到具体的内存地址之中。

常量池中每一项常量都是一个表，表结构起始的第一位是个 u1 类型的标志位，，代表着当前常量属于哪种常量类型。

常量池的项目类型：

![](https://upload-images.jianshu.io/upload_images/24520697-fd076188fb72bff8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

CONSTANT_Class_info的结构：

![](https://upload-images.jianshu.io/upload_images/24520697-a06fe701353586db.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

tag是标志位，它用于区分常量类型；name_index是常量池的索引值，它指向常量池中一个CONSTANT_Utf8_info类型常量，，此常量代表了这个类（或者接口）的全限定名。

CONSTANT_Utf8_info型常量的结构：

![](https://upload-images.jianshu.io/upload_images/24520697-a7fb9a8038d9b9e4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

length值说明了这个UTF-8编码的字符串长度是多少字节，它后面紧跟着的长度为length字节的连续数据是一个使用UTF-8缩略编码表示的字符串。

常量池中的17种数据类型的结构总表：

![](https://upload-images.jianshu.io/upload_images/24520697-f0c2d4dd85e30448.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](https://upload-images.jianshu.io/upload_images/24520697-a2ea399c3c4cb1b1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

](https://upload-images.jianshu.io/upload_images/24520697-c2b23da349753852.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 访问标志

在常量池结束之后，紧接着的2个字节代表访问标志，这个标志用于识别一些类或者接口层次的访问信息，包括：这个Class是类还是接口；是否定义为public类型；是否定义为abstract类型；如果是类的话，是否被声明为final；等等。

具体访问标志：

![](https://upload-images.jianshu.io/upload_images/24520697-2795b6df3064c58e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

没有使用到的标志位要求一律为零。

### 类索引、父类索引与接口索引集合

类索引和父类索引都是一个u2类型的数据，而接口索引集合是一组u2类型的数据的集合，Class文件中由这三项数据来确定该类型的继承关系。类索引用于确定这个类的全限定名，父类索引用于确定这个类的父类的全限定名。除了java.lang.Object外，所有Java类的父类索引都不为0。

类索引、父类索引和接口索引集合都按顺序排列在访问标志之后，类索引和父类索引各自指向一个类型为CONSTANT_Class_info的类描述符常量，通过CONSTANT_Class_info类型的常量中的索引值可以找到定义在CONSTANT_Utf8_info类型的常量中的全限定名字符串。

![](https://upload-images.jianshu.io/upload_images/24520697-91a34a13ac039cb9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

对于接口索引集合，入口的第一项u2类型的数据为接口计数器，表示索引表的容量。

### 字段表集合

字段表用于描述接口或者类中声明的变量。字段可以包括的修饰符有字段的作用域（public、private、protected修饰符）、是实例变量还是类变量（static修饰符）、可变性（final）、并发可见性（volatile修饰符，是否强制从主内存读写）、可否被序列化（transient修饰符）、字段数据类型（基本类型、对象、数组）、字段名称。上述这些信息中，各个修饰符都是布尔值，要么有某个修饰符，要么没有，很适合使用标志位来表示。字段叫做什么名字、字段被定义为什么数据类型只能引用常量池中的常量来描述。

字段表结构：

![image-20210220105135646](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\image-20210220105135646.png)

字段修饰符放在access_flags项目中，它与类中的access_flags项目是非常类似的，都是一个u2的数据类型。

字段访问标志

![](https://upload-images.jianshu.io/upload_images/24520697-7e5752b711ea6d04.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

跟随access_flags标志的是两项索引值：name_index和descriptor_index。它们都是对常量池项的引用，分别代表着字段的简单名称以及字段和方法的描述符。

描述符标识字符含义：

![](https://upload-images.jianshu.io/upload_images/24520697-be63b82ebc66acf9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 方法表集合

Class文件存储
格式中对方法的描述与对字段的描述采用了几乎完全一致的方式，方法表的结构如同字段表一样，依次包括访问标志（access_flags）、名称索引（name_index）、描述符索引（descriptor_index）、属性表集合（attributes）几项。这些数据项目的含义也与字段表中的非常类似，仅在访问标志和属性表集合的可选项中有所区别。

方法表结构：

![](https://upload-images.jianshu.io/upload_images/24520697-85d8d8733f157366.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

方法访问标志：

![image-20210220110151880](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\image-20210220110151880.png)

方法里的Java代码，经过Javac编译器编译成字节码指令之后，存放在方法属性表集合中一个名为“Code”的属性里面。

与字段表集合相对应地，如果父类方法在子类中没有被重写，方法表集合中就不会出现来自父类的方法信息。但同样地，有可能会出现由编译器自动添加的方法，最常见的便是类构造器`<clinit>()`方法和实例构造器`<init>()`方法。

### 属性表集合

属性表（attribute_info）在前面的讲解之中已经出现过数次，Class文件、字段表、方法表都可以携带自己的属性表集合，以描述某些场景专有的信息。

《Java虚拟机规范》允许只要不与已有属性名重复，任何人实现的编译器都可以向属性表中写入自己定义的属性信息，Java虚拟机运行时会忽略掉它不认识的属性。

虚拟机规范预定义的属性：

![](https://upload-images.jianshu.io/upload_images/24520697-fc36d602ca0d76b0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](https://upload-images.jianshu.io/upload_images/24520697-a1ef2b604fec5685.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

对于每一个属性，它的名称都要从常量池中引用一个CONSTANT_Utf8_info类型的常量来表示，而属性值的结构则是完全自定义的，只需要通过一个u4的长度属性去说明属性值所占用的位数即可。

一个符合规则的属性表：

![image-20210220135120049](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\image-20210220135120049.png)

#### Code 属性

Java程序方法体里面的代码经过Javac编译器处理之后，最终变为字节码指令存储在Code属性内。Code属性出现在方法表的属性集合之中，但并非所有的方法表都必须存在这个属性，譬如接口或者抽象类中的方法就不存在Code属性。

Code属性表的结构：

![](https://upload-images.jianshu.io/upload_images/24520697-d7a39f2906394bb9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

attribute_name_index是一项指向CONSTANT_Utf8_info型常量的索引，此常量值固定为“Code”，它代表了该属性的属性名称，attribute_length指示了属性值的长度，由于属性名称索引与属性长度一共为6个字节，所以属性值的长度固定为整个属性表长度减去6个字节。

max_stack代表了操作数栈（Operand Stack）深度的最大值。在方法执行的任意时刻，操作数栈都不会超过这个深度。虚拟机运行的时候需要根据这个值来分配栈帧（Stack Frame）中的操作栈深度。

max_locals代表了局部变量表所需的存储空间。在这里，max_locals的单位是变量槽，byte、char、float、int、short、boolean和returnAddress等长度不超过32位的数据类型，每个局部变量占用一个变量槽，而double和long这两种64位的数据类型则需要两个变量槽来存放。方法参数（包括 this），显式异常处理程序的参数（atch块中所定义的异常）、方法体中定义的局部变量都需要依赖局部变量表来存放。注意，并不是在方法中用了多少个局部变量，Java虚拟机会将局部变量表中的变量槽进行重用，当代码执行超出一个局部变量的作用域时，这个局部变量所占的变量槽可以被其他局部变量所使用。

code_length和code用来存储Java源程序编译后生成的字节码指令。code_length代表字节码长度，code是用于存储字节码指令的一系列字节流。每个指令就是一个u1类型的单字节。

如果把一个Java程序中的信息分为代码（Code，方法体里面的Java代码）和元数据（Metadata，包括类、字段、方法定义及其他信息）两部分，那么在整个Class文件里，Code属性用于描述代码，所有的其他数据项目都用于描述元数据。

包含四个字段，这些字段的含义为：如果当字节码从第start_pc行[1]到第end_pc行之间（不含第end_pc行）出现了类型为catch_type或者其子类的异常（catch_type为指向一个CONSTANT_Class_info型常量的索引），则转到第handler_pc行继续处理。当catch_type的值为0时，代表任意异常情况都需要转到handler_pc处进行处理。

属性表结构：

![](https://upload-images.jianshu.io/upload_images/24520697-e4e9a3045aba0ff0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### Exceptions 属性

这里的Exceptions属性是在方法表中与Code属性平级的一项属性，不要与前面刚刚讲解完的异常表产生混淆。Exceptions属性的作用是列举出方法中可能抛出的受查异常，也就是方法描述时在throws关键字后面列举的异常。

Exceptions属性结构：

![](https://upload-images.jianshu.io/upload_images/24520697-a3520cde471d6120.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

此属性中的number_of_exceptions项表示方法可能抛出number_of_exceptions种受查异常，每一种受查异常使用一个exception_index_table项表示；exception_index_table是一个指向常量池中CONSTANT_Class_info型常量的索引，代表了该受查异常的类型。

#### LineNumberTable属性

LineNumberTable属性用于描述Java源码行号与字节码行号（字节码的偏移量）之间的对应关系。

LineNumberTable属性结构：

![](https://upload-images.jianshu.io/upload_images/24520697-b9fdcacb93719a3e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

line_number_table是一个数量为line_number_table_length、类型为line_number_info的集合，line_number_info表包含start_pc和line_number两个u2类型的数据项，前者是字节码行号，后者是Java源码行号。

#### 未完待续

## 5.2 字节码指令

Java虚拟机的指令由一个字节长度的、代表着某种特定操作含义的数字（称为操作码，Opcode）以及跟随其后的零至多个代表此操作所需的参数（称为操作数，Operand）构成。由于Java虚拟机采用面向操作数栈而不是面向寄存器的架构，所以大多数指令都不包含操作数，只有一个操作码，指令参数都存放在操作数栈中。

### 字节码与数据类型

在Java虚拟机的指令集中，大多数指令都包含其操作所对应的数据类型信息。举个例子，iload指令用于从局部变量表中加载int型的数据到操作数栈中，而fload指令加载的则是float类型的数据。

大部分指令都没有支持整数类型byte、char和short，甚至没有任何指令支持boolean类型。编译器会在编译期或运行期将byte和short类型的数据带符号扩展为相应的int类型数据，将boolean和char类型数据零位扩展为相应的int类型数据。

### 加载和存储指令

加载和存储指令用于将数据在栈帧中的局部变量表和操作数栈之间来回传输，这类指令包括：

- 将一个局部变量加载到操作栈：iload、iload_n、lload、lload_n、fload、fload_n、dload、dload_n、aload、aload_n
- 将一个数值从操作数栈存储到局部变量表：istore、istore_n、lstore、lstore_n、fstore、fstore_n、dstore、dstore_n、astore、astore_n
- 将一个常量加载到操作数栈：bipush、sipush、ldc、ldc_w、ldc2_w、aconst_null、iconst_m1、iconst_i、lconst_l、fconst_f、dconst_d
- 扩充局部变量表的访问索引的指令：wide

上面所列举的指令助记符中，有一部分是以n结尾的，意思是省略掉了显式的操作数，不需要进行取操作数的动作，因为实际上操作数就隐含在指令中。（例如iload_0的语义与操作数为0时的iload指令语义完全一致）。

### 运算指令

算术指令用于对两个操作数栈上的值进行某种特定运算，并把结果重新存入到操作栈顶。大体上运算指令可以分为两种：对整型数据进行运算的指令与对浮点型数据进行运算的指令。不存在直接支持byte、short、char和boolean类型的算术指令，对于上述几种数据的运算，应使用操作int类型的指令代替。所有的算术指令包括：

- 加法指令：iadd、ladd、fadd、dadd
- 减法指令：isub、lsub、fsub、dsub
- 乘法指令：imul、lmul、fmul、dmul
- 除法指令：idiv、ldiv、fdiv、ddiv
- 求余指令：irem、lrem、frem、drem
- 取反指令：ineg、lneg、fneg、dneg
- 位移指令：ishl、ishr、iushr、lshl、lshr、lushr
- 按位或指令：ior、lor
- 按位与指令：iand、land
- 按位异或指令：ixor、lxor
- 局部变量自增指令：iinc
- 比较指令：dcmpg、dcmpl、fcmpg、fcmpl、lcmp

如果某个操作结果没有明确的数学定义的话，将会使用NaN（Not a Number）值来表示。所有使用NaN值作为操作数的算术操作，结果都会返回NaN。

### 类型转换指令

类型转换指令可以将两种不同的数值类型相互转换，这些转换操作一般用于实现用户代码中的显式类型转换操作。

Java虚拟机直接支持宽化类型转换，即小范围类型向大范围类型的安全转换

- int类型到long、float或者double类型
- long类型到float、double类型
- float类型到double类型

与之相对的，处理窄化类型转换（Narrowing Numeric Conversion）时，就必须显式地使用转换指令来完成，这些转换指令包括i2b、i2c、i2s、l2i、f2i、f2l、d2i、d2l和d2f。窄化类型转换可能会导致转换结果产生不同的正负号、不同的数量级的情况，转换过程很可能会导致数值的精度丢失。

### 对象创建与访问指令

虽然类实例和数组都是对象，但Java虚拟机对类实例和数组的创建与操作使用了不同的字节码指令。对象创建后，就可以通过对象访问指令获取对象实例或者数组实例中的字段或者数组元素，这些指令包括：

- 创建类实例的指令：new
- 创建数组的指令：newarray、anewarray、multianewarray
- 访问类字段（static字段，或者称为类变量）和实例字段（非static字段，或者称为实例变量）的指令：getfield、putfield、getstatic、putstatic
- 把一个数组元素加载到操作数栈的指令：baload、caload、saload、iaload、laload、faload、daload、aaload
- 将一个操作数栈的值储存到数组元素中的指令：bastore、castore、sastore、iastore、fastore、dastore、aastore
- 取数组长度的指令：arraylength
- 检查类实例类型的指令：instanceof、checkcast

### 操作数栈管理指令

如同操作一个普通数据结构中的堆栈那样，Java虚拟机提供了一些用于直接操作操作数栈的指令，包括：

- 操作数栈的栈顶一个或两个元素出栈：pop、pop2
- 复制栈顶一个或两个数值并将复制值或双份的复制值重新压入栈顶：dup、dup2、dup_x1、dup2_x1、dup_x2、dup2_x2
- 将栈最顶端的两个数值互换：swap

### 控制转移指令

控制转移指令可以让Java虚拟机有条件或无条件地从指定位置指令（而不是控制转移指令）的下一条指令继续执行程序，从概念模型上理解，可以认为控制指令就是在有条件或无条件地修改PC寄存器的值。控制转移指令包括：

- 条件分支：ifeq、iflt、ifle、ifne、ifgt、ifge、ifnull、ifnonnull、if_icmpeq、if_icmpne、if_icmplt、if_icmpgt、if_icmple、if_icmpge、if_acmpeq和if_acmpne
- 复合条件分支：tableswitch、lookupswitch
- 无条件分支：goto、goto_w、jsr、jsr_w、ret

### 方法调用和返回指令

以下五条指令用于方法调用：

- invokevirtual指令：用于调用对象的实例方法，根据对象的实际类型进行分派（虚方法分派），这也是Java语言中最常见的方法分派方式。
- invokeinterface指令：用于调用接口方法，它会在运行时搜索一个实现了这个接口方法的对象，找出适合的方法进行调用。
- invokespecial指令：用于调用一些需要特殊处理的实例方法，包括实例初始化方法、私有方法和父类方法。
- invokestatic指令：用于调用类静态方法（static方法）。
- invokedynamic指令：用于在运行时动态解析出调用点限定符所引用的方法。并执行该方法。前面四条调用指令的分派逻辑都固化在Java虚拟机内部，用户无法改变，而invokedynamic指令的分派逻辑
  是由用户所设定的引导方法决定的。

方法调用指令与数据类型无关，而方法返回指令是根据返回值的类型区分的，包括ireturn（当返回值是boolean、byte、char、short和int类型时使用）、lreturn、freturn、dreturn和areturn，另外还有一条return指令供声明为void的方法、实例初始化方法、类和接口的类初始化方法使用。

### 异常处理指令

在Java程序中显式抛出异常的操作（throw语句）都由athrow指令来实现，除了用throw语句显式抛出异常的情况之外，《Java虚拟机规范》还规定了许多运行时异常会在其他Java虚拟机指令检测到异常状况时自动抛出。例如前面介绍整数运算中，当除数为零时，虚拟机会在idiv或ldiv指令中抛出ArithmeticException异常。

而在Java虚拟机中，处理异常（catch语句）不是由字节码指令来实现的（很久之前曾经使用jsr和ret指令来实现，现在已经不用了），而是采用异常表来完成。

### 同步指令

Java虚拟机可以支持方法级的同步和方法内部一段指令序列的同步，这两种同步结构都是使用管程（Monitor，更常见的是直接将它称为“锁”）来实现的。

方法级的同步是隐式的，无须通过字节码指令来控制，它实现在方法调用和返回操作之中。虚拟机可以从方法常量池中的方法表结构中的ACC_SYNCHRONIZED访问标志得知一个方法是否被声明为同步方法。当方法调用时，调用指令将会检查方法的ACC_SYNCHRONIZED访问标志是否被设置，如果设置了，执行线程就要求先成功持有管程，然后才能执行方法，最后当方法完成（无论是正常完成还是非正常完成）时释放管程。在方法执行期间，执行线程持有了管程，其他任何线程都无法再获取到同一个管程。如果一个同步方法执行期间抛出了异常，并且在方法内部无法处理此异常，那这个同步方法所持有的管程将在异常抛到同步方法边界之外时自动释放。

Java虚拟机的指令集中有monitorenter和monitorexit两条指令来支持synchronized关键字的语义。

代码同步演示：

```java
void onlyMe(Foo f) {
    synchronized(f) {
    	doSomething();
    }
}
```

编译后，字节码序列如下：

```
Method void onlyMe(Foo)
0 aload_1 // 将对象f入栈
1 dup 　　 // 复制栈顶元素（即f的引用）
2 astore_2 // 将栈顶元素存储到局部变量表变量槽 2中
3 monitorenter // 以栈定元素（即f）作为锁，开始同步
4 aload_0 // 将局部变量槽 0（即this指针）的元素入栈
5 invokevirtual #5 // 调用doSomething()方法
8 aload_2 // 将局部变量Slow 2的元素（即f）入栈
9 monitorexit // 退出同步
10 goto 18 // 方法正常结束，跳转到18返回
13 astore_3 // 从这步开始是异常路径，见下面异常表的Taget 13
14 aload_2 // 将局部变量Slow 2的元素（即f）入栈
15 monitorexit // 退出同步
16 aload_3 // 将局部变量Slow 3的元素（即异常对象）入栈
17 athrow // 把异常对象重新抛出给onlyMe()方法的调用者
18 return // 方法正常返回

Exception table:
FromTo Target Type
4 10 13 any
13 16 13 any
```

编译器必须确保无论方法通过何种方式完成，方法中调用过的每条monitorenter指令都必须有其对应的monitorexit指令，而无论这个方法是正常结束还是异常结束。

为了保证在方法异常完成时monitorenter和monitorexit指令依然可以正确配对执行，编译器会自动产生一个异常处理程序，这个异常处理程序声明可处理所有的异常，它的目的就是用来执行monitorexit指令。

# 6 虚拟机类加载机制

Java虚拟机把描述类的数据从Class文件加载到内存，并对数据进行校验、转换解析和初始化，最终形成可以被虚拟机直接使用的Java类型，这个过程被称作虚拟机的类加载机制。在Java语言里面，类型的加载、连接和初始化过程都是在程序运行期间完成的。Java天生可以动态扩展的语言特性就是依赖运行期动态加载和动态连接这个特点实现的。例如，编写一个面向接口的应用程序，可以等到运行时再指定其实际的实现类。

## 6.1 类加载的时机

一个类型从被加载到虚拟机内存中开始，到卸载出内存为止，它的整个生命周期将会经历加载、验证、准备、解析、初始化、使用和卸载七个阶段，其中验证、准备、解析三个部分统称为连接。

![](https://upload-images.jianshu.io/upload_images/24520697-39c8c8a776b4a66f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

加载、验证、准备、初始化和卸载这五个阶段的顺序是确定的，在某些情况下可以在初始化阶段之后再开始，这是为了支持Java语言的运行时绑定特性（也称为动态绑定或晚期绑定）

《Java虚拟机规范》则是严格规定了有且只有六种情况必须立即对类进行“初始化”（而加载、验证、准备自然需要在此之前开始）：

1. 遇到new、getstatic、putstatic或invokestatic这四条字节码指令时，如果类型没有进行过初始化，则需要先触发其初始化阶段。能够生成这四条指令的典型Java代码场景有：
   - 使用new关键字实例化对象的时候。
   - 读取或设置一个类型的静态字段（被final修饰、已在编译期把结果放入常量池的静态字段除外）
     的时候。
   - 调用一个类型的静态方法的时候。
2. 使用java.lang.reflect包的方法对类型进行反射调用的时候，如果类型没有进行过初始化，则需要先触发其初始化。
3. 当初始化类的时候，如果发现其父类还没有进行过初始化，则需要先触发其父类的初始化。
4. 当虚拟机启动时，用户需要指定一个要执行的主类（包含main()方法的那个类），虚拟机会先初始化这个主类。
5. 当使用JDK 7新加入的动态语言支持时，如果一个java.lang.invoke.MethodHandle实例最后的解析结果为REF_getStatic、REF_putStatic、REF_invokeStatic、REF_newInvokeSpecial四种类型的方法句柄，并且这个方法句柄对应的类没有进行过初始化，则需要先触发其初始化。
6. 当一个接口中定义了JDK 8新加入的默认方法（被default关键字修饰的接口方法）时，如果有这个接口的实现类发生了初始化，那该接口要在其之前被初始化。

这六种场景中的行为称为对一个类型进行主动引用，除此之外，所有引用类型的方式都不会触发初始化，称为被动引用。下面举例说明何为被动引用。

```java
// 非主动使用类字段
// 通过子类引用父类的静态字段，不会导致子类初始化
public class SuperClass {
    static {
        System.out.println("SuperClass init...");
    }
    public static int value = 123;
}
public class SubClass extends SuperClass {
    static {
        System.out.println("SubClass init...");
    }
}
public class NotInitialization {
    public static void main(String[] args) {
    	System.out.println(SubClass.value);
    }
}
```

结果只会输出“SuperClass init！”，而不会输出“SubClass init！”。对于静态字段，只有直接定义这个字段的类才会被初始化，因此通过其子类来引用父类中定义的静态字段，只会触发父类的初始化而不会触发子类的初始化。

```java
// 非主动使用类字段
// 常量在编译阶段会存入调用类的常量池中，本质上没有直接引用到定义常量的类，因此不会触发定义常量的类的初始化
public class ConstClass {
    static {
        System.out.println("ConstClass init");
    }
    public static final String HELLOWORLD = "hello world";
}
public Class NotInitialization {
    public static void main(String[] args) {
        System.out.print(ConstClass.HELLOWORLD);
    }
}
```

上述代码运行之后，没有输出“ConstClass init”，这是因为虽然在Java源码中确实引用了ConstClass类的常量HELLOWORLD，但其实在编译阶段通过常量传播优化，已经将此常量的值“hello world”直接存储在NotInitialization类的常量池中，以后NotInitialization对常量ConstClass.HELLOWORLD的引用，实际都被转化为NotInitialization类对自身常量池的引用了。

接口也有初始化过程，而接口中不能使用“static{}”语句块，但编译器仍然会为接口生成`<clinit>()`类构造器，用于初始化接口中所定义的成员变量。接口与类真正有所区别的是，当一个类在初始化时，要求其父类全部都已经初始化过了，但是一个接口在初始化时，并不要求其父接口全部都完成了初始化，只有在真正使用到父接口的时候（如引用接口中定义的常量）才会初始化。

## 6.2 类加载的过程

### 加载

“加载”（Loading）阶段是整个“类加载”过程中的一个阶段，在加载阶段，Java虚拟机需要完成以下三件事情：

1. 通过一个类的全限定名来获取定义此类的二进制字节流。
2. 将这个字节流所代表的静态存储结构转化为方法区的运行时数据结构。
3. 在内存中生成一个代表这个类的java.lang.Class对象，作为方法区这个类的各种数据的访问入口。

加载阶段既可以使用Java虚拟机里内置的引导类加载器来完成，也可以由用户自定义的类加载器去完成，开发人员通过定义自己的类加载器去控制字节流的获取方式（重写一个类加载器的findClass()或loadClass()方法），实现根据自己的想法来赋予应用程序获取运行代码的动态性。

数组类本身不通过类加载器创建，它是由Java虚拟机直接在内存中动态构造出来的。但是数组类的元素类型最终还是要靠类加载器来完成加载，一个数组类（下面简称为C）创建过程遵循以下规则：

- 如果数组的组件类型（Component Type，指的是数组去掉一个维度的类型，注意和前面的元素类型区分开来）是引用类型，那就递归采用本节中定义的加载过程去加载这个组件类型，数组C将被标识在加载该组件类型的类加载器的类名称空间上。
- 如果数组的组件类型不是引用类型（例如int[]数组的组件类型为int），Java虚拟机将会把数组C标记为与引导类加载器关联。
- 数组类的可访问性与它的组件类型的可访问性一致，如果组件类型不是引用类型，它的数组类的可访问性将默认为public，可被所有的类和接口访问到。

类型数据妥善安置在方法区之后，会在Java堆内存中实例化一个java.lang.Class类的对象，这个对象将作为程序访问方法区中的类型数据的外部接口。

### 验证

验证是连接阶段的第一步，这一阶段的目的是确保Class文件的字节流中包含的信息符合《Java虚拟机规范》的全部约束要求。验证阶段大致上会完成下面四个阶段的检验动作：文件格式验证、元数据验证、字节码验证和符号引用验证。

### 准备

准备阶段是正式为类中定义的静态变量分配内存并设置类变量初始值（通常情况下是 0 值）的阶段，从概念上讲，这些变量所使用的内存都应当在方法区中进行分配，但必须注意到方法区本身是一个逻辑上的区域，在JDK 7及之前，HotSpot使用永久代来实现方法区时，实现是完全符合这种逻辑概念的；而在JDK 8及之后，类变量则会随着Class对象一起存放在Java堆中，这时候“类变量在方法区”就完全是一种对逻辑概念的表述了。

如果类字段的字段属性表中存在ConstantValue属性，那在准备阶段变量值就会被初始化为ConstantValue属性所指定的初始值。

### 解析

解析阶段是Java虚拟机将常量池内的符号引用替换为直接引用的过程。

- 符号引用：用一组符号来描述所引用的目标。
- 直接引用：可以直接指向目标的指针、相对偏移量或者是一个能间接定位到目标的句柄。

虚拟机实现可以对第一次解析的结果进行缓存，譬如在运行时直接引用常量池中的记录，并把常量标识为已解析状态，从而避免解析动作重复进行。无论是否真正执行了多次解析动作，Java虚拟机都需要保证的是在同一个实体中，如果一个符号引用之前已经被成功解析过，那么后续的引用解析请求就应当一直能够
成功；同样地，如果第一次解析失败了，其他指令对这个符号的解析请求也应该收到相同的异常，哪怕这个请求的符号在后来已成功加载进Java虚拟机内存之中。

不过对于invokedynamic指令，上面的规则就不成立了。当碰到某个前面已经由invokedynamic指令触发过解析的符号引用时，并不意味着这个解析结果对于其他invokedynamic指令也同样生效。因为invokedynamic指令的目的本来就是用于动态语言支持，它对应的引用称为动态调用点限定符，这里动态的含义是指必须等到程序实际运行到这条指令时，解析动作才能进行。

#### 类或接口的解析

假设当前代码所处的类为D，如果要把一个从未解析过的符号引用N解析为一个类或接口C的直接引用，那虚拟机完成整个解析的过程需要包括以下3个步骤：

1）如果C不是一个数组类型，那虚拟机将会把代表N的全限定名传递给D的类加载器去加载这个类C。在加载过程中，由于元数据验证、字节码验证的需要，又可能触发其他相关类的加载动作，例如加载这个类的父类或实现的接口。一旦这个加载过程出现了任何异常，解析过程就将宣告失败。

2）如果C是一个数组类型，并且数组的元素类型为对象，也就是N的描述符会是似“[Ljava/lang/Integer”的形式，那将会按照第一点的规则加载数组元素类型。如果N的描述符如前面所假设的形式，需要加载的元素类型就是“java.lang.Integer”，接着由虚拟机生成一个代表该数组维度和元素的数组对象。

3）如果上面两步没有出现任何异常，那么C在虚拟机中实际上已经成为一个有效的类或接口了，但在解析完成前还要进行符号引用验证，确认D是否具备对C的访问权限。如果发现不具备访问权限，将抛出java.lang.IllegalAccessError异常。

在JDK 9引入了模块化以后，一个public类型也不再意味着程序任何位置都有它的访问权限，我们还必须检查模块间的访问权限。

如果我们说一个D拥有C的访问权限，那就意味着以下3条规则中至少有其中一条成立：

- 被访问类C是public的，并且与访问类D处于同一个模块。
- 被访问类C是public的，不与访问类D处于同一个模块，但是被访问类C的模块允许被访问类D的模块进行访问。
- 被访问类C不是public的，但是它与访问类D处于同一个包中。

#### 字段解析

要解析一个未被解析过的字段符号引用，首先将会对字段所属的类或接口的符号引用进行解析。如果解析成功完成，那把这个字段所属的类或接口用C表示，《Java虚拟机规范》要求按照如下步骤对C进行后续字段的搜索：

1）如果C本身就包含了简单名称和字段描述符都与目标相匹配的字段，则返回这个字段的直接引用，查找结束。

2）否则，如果在C中实现了接口，将会按照继承关系从下往上递归搜索各个接口和它的父接口，如果接口中包含了简单名称和字段描述符都与目标相匹配的字段，则返回这个字段的直接引用，查找结束。

3）否则，如果C不是java.lang.Object的话，将会按照继承关系从下往上递归搜索其父类，如果在父类中包含了简单名称和字段描述符都与目标相匹配的字段，则返回这个字段的直接引用，查找结束。

4）否则，查找失败，抛出java.lang.NoSuchFieldError异常。

如果查找过程成功返回了引用，将会对这个字段进行权限验证，如果发现不具备对字段的访问权限，将抛出java.lang.IllegalAccessError异常。

#### 方法解析

方法解析需要先解析出方法所属类或接口的符号引用。用C表示这个类，接下来虚拟机将会按照如下步骤进行后续的方法搜索：

1）由于Class文件格式中类的方法和接口的方法符号引用的常量类型定义是分开的，如果在类的方法表中发现class_index中索引的C是个接口的话，那就直接抛出java.lang.IncompatibleClassChangeError异常。

2）如果通过了第一步，在类C中查找是否有简单名称和描述符都与目标相匹配的方法，如果有则返回这个方法的直接引用，查找结束。

3）否则，在类C的父类中递归查找是否有简单名称和描述符都与目标相匹配的方法，如果有则返回这个方法的直接引用，查找结束。

4）否则，在类C实现的接口列表及它们的父接口之中递归查找是否有简单名称和描述符都与目标相匹配的方法，如果存在匹配的方法，说明类C是一个抽象类，这时候查找结束，抛出 java.lang.AbstractMethodError 异常。

5）否则，宣告方法查找失败，抛出java.lang.NoSuchMethodError。

最后，如果查找过程成功返回了直接引用，将会对这个方法进行权限验证，如果发现不具备对此方法的访问权限，将抛出java.lang.IllegalAccessError异常。

#### 接口方法解析

接口方法也是需要先解析出所属的类或接口的符号引用，用C表示这个接口，接下来虚拟机将会按照如下步骤进行后续的接口方法搜索：

1）与类的方法解析相反，如果在接口方法表中发现class_index中的索引C是个类而不是接口，那么就直接抛出java.lang.IncompatibleClassChangeError异常。

2）否则，在接口C中查找是否有简单名称和描述符都与目标相匹配的方法，如果有则返回这个方法的直接引用，查找结束。

3）否则，在接口C的父接口中递归查找，直到java.lang.Object类（接口方法的查找范围也会包括Object类中的方法）为止，看是否有简单名称和描述符都与目标相匹配的方法，如果有则返回这个方法的直接引用，查找结束

4）对于规则3，由于Java的接口允许多重继承，如果C的不同父接口中存有多个简单名称和描述符都与目标相匹配的方法，那将会从这多个方法中返回其中一个并结束查找，《Java虚拟机规范》中并没有进一步规则约束应该返回哪一个接口方法。但与之前字段查找类似地，不同发行商实现的Javac编译器有可能会按照更严格的约束拒绝编译这种代码来避免不确定性。

5）否则，宣告方法查找失败，抛出java.lang.NoSuchMethodError异常。

在JDK 9之前，Java接口中的所有方法都默认是public的，也没有模块化的访问约束，所以不存在访问权限的问题，接口方法的符号解析就不可能抛出java.lang.IllegalAccessError异常。但在JDK 9中增加了接口的静态私有方法，也有了模块化的访问约束，所以从JDK 9起，接口方法的访问也完全有可能因访问权限控制而出现java.lang.IllegalAccessError异常。

### 初始化

在初始化阶段，则会根据程序员通过程序编码制定的主观计划去初始化类变量和其他资源。初始化阶段就是执行类构造器`<clinit>()`方法的过程。

`<clinit>()`方法是由编译器自动收集类中的所有类变量的赋值动作和静态语句块（static{}块）中的语句合并产生的，编译器收集的顺序是由语句在源文件中出现的顺序决定的，静态语句块中只能访问到定义在静态语句块之前的变量，定义在它之后的变量，在前面的静态语句块可以赋值，但是不能访问。

非法前向引用变量：

```java
public class Test {
    static {
        i = 0; // 给变量赋值可以正常编译通过
        System.out.print(i); // 这句编译器会提示“非法向前引用”
    }
    static int i = 1;
}
```

Java虚拟机会保证在子类的`<clinit>()`方法执行前，父类的`<clinit>()`方法已经执行完毕。

由于父类的`<clinit>()`方法先执行，也就意味着父类中定义的静态语句块要优先于子类的变量赋值操作，下面代码中字段B的值将会是2而不是1：

```java
static class Parent {
    public static int A = 1;
    static {
        A = 2;
	}
}
static class Sub extends Parent {
    public static int B = A;
}
```

如果一个类中没有静态语句块，也没有对变量的赋值操作，那么编译器可以不为这个类生成`<clinit>()`方法。

接口中不能使用静态语句块，但仍然有变量初始化的赋值操作，因此接口与类一样都会生成`<clinit>()`方法。但接口与类不同的是，执行接口的`<clinit>()`方法不需要先执行父接口的`<clinit>()`方法，因为只有当父接口中定义的变量被使用时，父接口才会被初始化。此外，接口的实现类在初始化时也一样不会执行接口的`<clinit>()`方法。

如果多个线程同时去初始化一个类，那么只会有其中一个线程去执行这个类的`<clinit>()`方法。同一个类加载器下，一个类型只会被初始化一次。

## 6.3 类加载器

### 类与类加载器

对于任意一个类，都必须由加载它的类加载器和这个类本身一起共同确立其在Java虚拟机中的唯一性。比较两个类是否相等，只有在这两个类是由同一个类加载器加载的前提下才有意义。

不同的类加载器对instanceof关键字运算的结果的影响：

```java
public class ClassLoaderTest {
    public static void main(String[] args) throws Exception {
        ClassLoader myLoader = new ClassLoader() {
            @Override
            public Class<?> loadClass(String name) throws ClassNotFoundException {
                try {
                    String fileName = name.substring(name.lastIndexOf(".") + 1)+".class";
                    InputStream is = getClass().getResourceAsStream(fileName);
                    if (is == null) {
                    return super.loadClass(name);
                }
                    byte[] b = new byte[is.available()];
                    is.read(b);
                    return defineClass(name, b, 0, b.length);
                } catch (IOException e) {
                    throw new ClassNotFoundException(name);
                }
                }
        };
        Object obj = myLoader.loadClass("org.fenixsoft.classloading.ClassLoaderTest").newInstance();
        System.out.println(obj.getClass());
        System.out.println(obj instanceof org.fenixsoft.classloading.ClassLoaderTest);
    }
}
```

运行结果:

```
class org.fenixsoft.classloading.ClassLoaderTest
false
```

原因是Java虚拟机中同时存在了两个ClassLoaderTest类，一个是由虚拟机的应用程序类加载器所加载的，另外一个是由我们自定义的类加载器加载的，虽然它们都来自同一个Class文件，但在Java虚拟机中仍然是两个互相独立的类，做对象所属类型检查时的结果自然为false。

### 双亲委派模型

站在Java虚拟机的角度来看，只存在两种不同的类加载器：一种是启动类加载器（Bootstrap ClassLoader），这个类加载器使用C++语言实现，是虚拟机自身的一部分；另外一种就是其他所有的类加载器，这些类加载器都由Java语言实现，独立存在于虚拟机外部，并且全都继承自抽象类java.lang.ClassLoader。

站在Java开发人员的角度来看，Java一直保持着三层类加载器、双亲委派的类加载架构。

绝大多数Java程序都会使用到以下3个系统提供的类加载器来进行加载：

- 启动类加载器（Bootstrap Class Loader）：这个类加载器负责加载存放在<JAVA_HOME>\lib目录，或者被-Xbootclasspath参数所指定的路径中存放的，而且是Java虚拟机能够识别的（按照文件名识别，如rt.jar、tools.jar，名字不符合的类库即使放在lib目录中也不会被加载）类库加载到虚拟机的内存中。启动类加载器无法被Java程序直接引用，用户在编写自定义类加载器时，如果需要把加载请求委派给引导类加载器去处理，那直接使用null代替即可。

  ```java
  /**
  Returns the class loader for the class. Some implementations may use null to represent the bootstrap class loader. */
  public ClassLoader getClassLoader() {
      ClassLoader cl = getClassLoader0();
      if (cl == null)
          return null;
          SecurityManager sm = System.getSecurityManager();
      if (sm != null) {
          ClassLoader ccl = ClassLoader.getCallerClassLoader();
          if (ccl != null && ccl != cl && !cl.isAncestor(ccl)) {
          	sm.checkPermission(SecurityConstants.GET_CLASSLOADER_PERMISSION);
          }
      }
      return cl;
  }
  ```

- 扩展类加载器（Extension Class Loader）：这个类加载器是在类sun.misc.Launcher$ExtClassLoader中以Java代码的形式实现的。它负责加载<JAVA_HOME>\lib\ext目录中，或者被java.ext.dirs系统变量所指定的路径中所有的类库。JDK的开发团队允许用户将具有通用性的类库放置在ext目录里以扩展Java SE的功能。
- 应用程序类加载器（Application Class Loader）：也称它为系统类加载器，这个类加载器由sun.misc.Launcher$AppClassLoader来实现。它负责加载用户类路径（ClassPath）上所有的类库。

类加载器双亲委派模型：

![](https://upload-images.jianshu.io/upload_images/24520697-645f005e541ffe27.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

各种类加载器之间的层次关系被称为类加载器的双亲委派模型。双亲委派模型要求除了顶层的启动类加载器外，其余的类加载器都应有自己的父类加载器。不过这里类加载器之间的父子关系一般不是以继承（Inheritance）的关系来实现的，而是通常使用组合（Composition）关系来复用父加载器的代码。

双亲委派模型的工作过程是：如果一个类加载器收到了类加载的请求，它首先不会自己去尝试加 载这个类，而是把这个请求委派给父类加载器去完成，每一个层次的类加载器都是如此，因此所有的 加载请求最终都应该传送到最顶层的启动类加载器中，只有当父加载器反馈自己无法完成这个加载请 求（它的搜索范围中没有找到所需的类）时，子加载器才会尝试自己去完成加载。

使用双亲委派模型来组织类加载器之间的关系，一个显而易见的好处就是Java中的类随着它的类 加载器一起具备了一种带有优先级的层次关系。例如类java.lang.Object，它存放在rt.jar之中，无论哪一 个类加载器要加载这个类，最终都是委派给处于模型最顶端的启动类加载器进行加载，因此Object类 在程序的各种类加载器环境中都能够保证是同一个类。反之，如果没有使用双亲委派模型，都由各个 类加载器自行去加载的话，如果用户自己也编写了一个名为java.lang.Object的类，并放在程序的 ClassPath中，那系统中就会出现多个不同的Object类，Java类型体系中最基础的行为也就无从保证，应用程序将会变得一片混乱。

## 6.4 Java 模块化系统

# 7 虚拟机字节码执行引擎

## 7.1 运行时栈帧结构

Java虚拟机以方法作为最基本的执行单元，栈帧则是用于支持虚拟机进行方法调用和方法执行背后的数据结构。栈帧存储了方法的局部变量表、操作数栈、动态连接和方法返回地址等信息。

在编译Java程序源码的时候，栈帧中需要多大的局部变量表，需要多深的操作数栈就已经被分析计算出来，并且写入到方法表的Code属性之中。

以Java程序的角度来看，同一时刻、同一条线程里面，在调用堆栈的所有方法都同时处于执行状态。而对于执行引擎来讲，在活动线程中，只有位于栈顶的方法才是在运行的，只有位于栈顶的栈帧才是生效的，其被称为当前栈帧，与这个栈帧所关联的方法被称为当前方法。执行引擎所运行的所有字节码指令都只针对当前栈帧进行操作。

![](https://upload-images.jianshu.io/upload_images/24520697-4da92d42a7301181.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 局部变量表

局部变量表是一组变量值的存储空间，用于存放方法参数和方法内部定义的局部变量。在编译时，方法的Code属性的max_locals数据项中确定了该方法所需分配的局部变量表的最大容量。局部变量表的容量以变量槽为最小单位。

Java虚拟机通过索引定位的方式使用局部变量表，索引值的范围是从0开始至局部变量表最大的变量槽数量。如果访问的是32位数据类型的变量，索引N就代表了使用第N个变量槽，如果访问的是64位数据类型的变量，则说明会同时使用第N和N+1两个变量槽。对于两个相邻的共同存放一个64位数据的两个变量槽，虚拟机不允许采用任何方式单独访问其中的某一个。

当一个方法被调用时，Java虚拟机会使用局部变量表来完成参数值到参数变量列表的传递过程，即实参到形参的传递。如果执行的是实例方法，那局部变量表中第0位索引的变量槽默认是用于传递方法所属对象实例的引用（关键字this）。其余参数则按照参数表顺序排列，占用从1开始的局部变量槽，参数表分配完毕后，再根据方法体内部定义的变量顺序和作用域分配其余的变量槽。

为了尽可能节省栈帧耗用的内存空间，局部变量表中的变量槽是可以重用的，方法体中定义的变量，其作用域并不一定会覆盖整个方法体，如果当前字节码PC计数器的值已经超出了某个变量的作用域，那这个变量对应的变量槽就可以交给其他变量来重用。不过，这样的设计除了节省栈帧空间以外，还会伴随有少量额外的副作用，例如在某些情况下变量槽的复用会直接影响到系统的垃圾收集行为，请看以下代码：

```java
public static void main(String[] args)() {
    byte[] placeholder = new byte[64 * 1024 * 1024];
    System.gc();
}
```

上述代码向内存填充了64MB的数据，然后通知虚拟机进行垃圾收集。在执行System.gc()时，变量placeholder还处于作用域之内，虚拟机自然不敢回收掉placeholder的内存。将代码修改一下：

```java
public static void main(String[] args)() {
    {
    	byte[] placeholder = new byte[64 * 1024 * 1024];
    }
    System.gc();
}
```

placeholder的作用域被限制在花括号以内，从代码逻辑上讲，在执行System.gc()的时候，placeholder已经不可能再被访问了，但执行这段程序，会发现运行结果如下，还是有64MB的内存没有被回收掉。再继续修改：

```java
public static void main(String[] args)() {
    {
    	byte[] placeholder = new byte[64 * 1024 * 1024];
    }
    int a = 0;
    System.gc();
}
```

发现这次内存真的被正确回收了。placeholder能否被回收的根本原因就是：局部变量表中的变量槽是否还存有关于placeholder数组对象的引用。第一次修改中，代码虽然已经离开了placeholder的作用域，但在此之后，再没有发生过任何对局部变量表的读写操作，placeholder原本所占用的变量槽还没有被其他变量所复用，所以作为GC Roots一部分的局部变量表仍然保持着对它的关联。

局部变量不像前面介绍的类变量那样存在准备阶段，类的字段变量有两次赋初始值的过程，一次在准备阶段，赋予系统初始值；另外一次在初始化阶段，赋予程序员定义的初始值。因此即使在初始化阶段程序员没有为类变量赋值也没有关系，类变量仍然具有一个确定的初始值。但是如果一个局部变量定义了但没有赋初始值，那它是完全不能使用的。

### 操作数栈

操作数栈的最大深度也在编译的时候被写入到Code属性的max_stacks数据项之中。32位数据类型所占的栈容量为1，64位数据类型所占的栈容量为2。ava虚拟机的解释执行引擎被称为“基于栈的执行引擎”，里面的“栈”就是操作数栈。

当一个方法刚刚开始执行的时候，这个方法的操作数栈是空的，在方法的执行过程中，会有各种字节码指令往操作数栈中写入和提取内容。例如整数加法的字节码指令iadd，这条指令在运行的时候要求操作数栈中最接近栈顶的两个元素已经存入了两个int型的数值，当执行这个指令时，会把这两个int值出栈并相加，然后将相加的结果重新入栈。

大多虚拟机的实现里都会进行一些优化处理，令两个栈帧出现一部分重叠。让下面栈帧的部分操作数栈与上面栈帧的部分局部变量表重叠在一起，这样做不仅节约了一些空间，更重要的是在进行方法调用时就可以直接共用一部分数据，无须进行额外的参数复制传递。

![](https://upload-images.jianshu.io/upload_images/24520697-0fa4d4c268ebca30.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 动态连接

每个栈帧都包含一个指向运行时常量池中该栈帧所属方法的引用，持有这个引用是为了支持方法调用过程中的动态连接。Class文件的常量池中存有大量的符号引用，字节码中的方法调用指令就以常量池里指向方法的符号引用作为参数。这些符号引用一部分会在类加载阶段或者第一次使用的时候就被转化为直接引用，这种转化被称为静态解析。另外一部分将在每一次运行期间都转化为直接引用，这部分就称为动态连接。

### 方法返回地址

当一个方法开始执行后，只有两种方式退出这个方法。第一种方式是执行引擎遇到任意一个方法返回的字节码指令，这时候可能会有返回值传递给上层的方法调用者，这种退出方法的方式称为“正常调用完成”。

另外一种退出方式是在方法执行的过程中遇到了异常，并且这个异常没有在方法体内得到妥善处理。无论是Java虚拟机内部产生的异常，还是代码中使用athrow字节码指令产生的异常，只要在本方法的异常表中没有搜索到匹配的异常处理器，就会导致方法退出，这种退出方法的方式称为“异常调用完成“。一个方法使用异常完成出口的方式退出，是不会给它的上层调用者提供任何返回值的。

方法正常退出时，主调方法的PC计数器的值就可以作为返回地址，栈帧中很可能会保存这个计数器值。而方法异常退出时，返回地址是要通过异常处理器表来确定的，栈帧中就一般不会保存这部分信息。

## 7.2 方法调用

方法调用并不等同于方法中的代码被执行，方法调用阶段唯一的任务就是确定被调用方法的版本（即调用哪一个方法）。Class文件的编译过程中不包含传统程序语言编译的连接步骤，一切方法调用在Class文件里面存储的都只是符号引用，而不是方法在实际运行时内存布局中的入口地址。某些调用需要在类加载期间，甚至到运行期间才能确定目标方法的直接引用。

### 解析

所有方法调用的目标方法在Class文件里面都是一个常量池中的符号引用，在类加载的解析阶段，会将其中的一部分符号引用转化为直接引用，这种解析能够成立的前提是：方法在程序真正运行之前就有一个可确定的调用版本，并且这个方法的调用版本在运行期是不可改变的。换句话说，调用目标在程序代码写好、编译器进行编译那一刻就已经确定下来。这类方法的调用被称为解析。

在Java语言中符合“编译期可知，运行期不可变”这个要求的方法，主要有静态方法和私有方法两大类，前者与类型直接关联，后者在外部不可被访问，这两种方法各自的特点决定了它们都不可能通过继承或别的方式重写出其他版本，因此它们都适合在类加载阶段进行解析。

调用不同类型的方法，字节码指令集里设计了不同的指令。在Java虚拟机支持以下5条方法调用字节码指令，分别是：

- invokestatic。用于调用静态方法。
- invokespecial。用于调用实例构造器`<init>()`方法、私有方法和父类中的方法。
- invokevirtual。用于调用所有的虚方法。
- invokeinterface。用于调用接口方法，会在运行时再确定一个实现该接口的对象。
- invokedynamic。先在运行时动态解析出调用点限定符所引用的方法，然后再执行该方法。前面4条调用指令，分派逻辑都固化在Java虚拟机内部，而invokedynamic指令的分派逻辑是由用户设定的引导方法来决定的。

只要能被invokestatic和invokespecial指令调用的方法，都可以在解析阶段中确定唯一的调用版本，Java语言里符合这个条件的方法共有静态方法、私有方法、实例构造器、父类方法4种，再加上被final修饰的方法（尽管它使用invokevirtual指令调用），这5种方法调用会在类加载的时候就可以把符号引用解析为该方法的直接引用。这些方法统称为“非虚方法”，与之相反，其他方法就被称为“虚方法”。

### 分派

Java是一门面向对象的程序语言，因为Java具备面向对象的3个基本特征：继承、封装和多态。本节讲解的分派调用过程将会揭示多态性特征的一些最基本的体现，如“重载”和“重写”在Java虚拟机之中是如何实现的。

#### 静态分派

为了解释静态分派和重载，编写了如下重载代码，以分析虚拟机和编译器确定方法版本的过程。

```java
// 方法静态分派演示
public class StaticDispatch {
    static abstract class Human {
    }
    static class Man extends Human {
    }
    static class Woman extends Human {
    }
    public void sayHello(Human guy) {
    	System.out.println("hello,guy!");
    }
    public void sayHello(Man guy) {
    	System.out.println("hello,gentleman!");
    }
    public void sayHello(Woman guy) {
    	System.out.println("hello,lady!");
    }
    public static void main(String[] args) {
        Human man = new Man();
        Human woman = new Woman();
        StaticDispatch sr = new StaticDispatch();
        sr.sayHello(man);
        sr.sayHello(woman);
    }
}
```

运行结果:

```
hello,guy!
hello,guy!
```

为什么虚拟机会选择执行参数类型为Human的重载版本呢？在解决这个问题之前，我们先通过如下代码来定义两个关键概念：

```java
Human man = new Man();
```

我们把上面代码中的“Human”称为变量的“静态类型”（Static Type），或者叫“外观类型”（Apparent Type），后面的“Man”则被称为变量的“实际类型”（Actual Type）或者叫“运行时类型”（Runtime Type）。静态类型和实际类型在程序中都可能会发生变化，区别是静态类型的变化仅仅在使用时发生，变量本身的静态类型不会被改变，并且最终的静态类型是在编译期可知的；而实际类型变化的结果在运行期才可确定，编译器在编译程序的时候并不知道一个对象的实际类型是什么。例如 有如下代码：

```java
// 实际类型变化
Human human = (new Random()).nextBoolean() ? new Man() : new Woman();
// 静态类型变化
sr.sayHello((Man) human)
sr.sayHello((Woman) human)
```

对象human的实际类型是可变的，须等到程序运行到这行的时候才能确定。而human的静态类型是Human，也可以在使用时（如sayHello()方法中的强制转型）临时改变这个类型，但这个改变仍是在编译期是可知的。

前面`main()`里面的两次sayHello()方法调用，在方法接收者已经确定是对象“sr”的前提下，使用哪个重载版本，就完全取决于传入参数的数量和数据类型。代码中故意定义了两个静态类型相同，而实际类型不同的变量，但虚拟机（或者准确地说是编译器）在重载时是通过参数的静态类型而不是实际类型作为判定依据的。由于静态类型在编译期可知，所以在编译阶段，Javac编译器就根据参数的静态类型决定了会使用哪个重载版本，因此选择了`sayHello(Human)`作为调用目标，并把这个方法的符号引用写到`main()`方法里的两条invokevirtual指令的参数中。

所有依赖静态类型来决定方法执行版本的分派动作，都称为静态分派。静态分派的最典型应用表现就是方法重载。静态分派发生在编译阶段，因此确定静态分派的动作实际上不是由虚拟机来执行的，这点也是为何一些资料选择把它归入“解析”而不是“分派”的原因。

需要注意Javac编译器虽然能确定出方法的重载版本，但在很多情况下这个重载版本并不是“唯一”的，往往只能确定一个“相对更合适的”版本。比如下面代码：

```java
public class Overload {
    public static void sayHello(Object arg) {
    	System.out.println("hello Object");
    }
    public static void sayHello(int arg) {
    System.out.println("hello int");
    }
    public static void sayHello(long arg) {
    System.out.println("hello long");
    }
    public static void sayHello(Character arg) {
    System.out.println("hello Character");
    }
    public static void sayHello(char arg) {
    System.out.println("hello char");
    }
    public static void sayHello(char... arg) {
    	System.out.println("hello char ...");
    }
    public static void sayHello(Serializable arg) {
    	System.out.println("hello Serializable");
    }
    public static void main(String[] args) {
    	sayHello('a');
    }
}
```

输出结果：

```
hello char
```

如果注释掉`sayHello(char arg)`方法，输出会变为：

```
hello int
```

这时发生了一次自动类型转换（‘a’转换为97），参数类型为int的重载也是合适的，继续注释`sayHello(int arg)`方法，那输出会变为：

```
hello long
```

这时发生了两次自动类型转换，'a'转型为整数97之后，进一步转型为长整数97L匹配了参数类型为long的重载。按照char>int>long>float>double的顺序转型进行匹配。但不会匹配到byte和short类型的重载，因为char到byte或short的转型是不安全的。我们继续注释掉sayHello(long arg)方法，那输出会变为：

```
hello Character
```

这时发生了一次自动装箱，'a'被包装为它的封装类型java.lang.Character。继续注释掉sayHello(Character arg)方法，那输出会变为：

```
hello Serializable
```

java.lang.Serializable是java.lang.Character类实现的一个接口，当自动装箱之后发现还是找不到装箱类，但是找到了装箱类所实现的接口类型，所以紧接着又发生一次自动转型。Character是绝对不会转型为Integer的，它只能安全地转型为它实现的接口或父类。下面继续注释掉sayHello(Serializable arg)方法，输出变为：

```
hello Object
```

sayHello(Object arg)也注释掉，输出将会变为：

```
hello char ...
```

可见变长参数的重载优先级是最低的，这时候字符'a'被当作了一个char[]数组的元素。

解析与分派这两者之间的关系并不是二选一的排他关系，它们是在不同层次上去筛选、确定目标方法的过程。例如前面说过静态方法会在编译期确定、在类加载期就进行解析，而静态方法显然也是可以拥有重载版本的，选择重载版本的过程也是通过静态分派完成的。

#### 动态分派

动态分派与 Java语言多态性的另外一个重要体现——重写有着很密切的关联。请看如下代码：

```java
public class DynamicDispatch {
    static abstract class Human {
    protected abstract void sayHello();
    }
    static class Man extends Human {
    @Override
        protected void sayHello() {
            System.out.println("man say hello");
        }
    }
    static class Woman extends Human {
    @Override
        protected void sayHello() {
            System.out.println("woman say hello");
        }
    }
    public static void main(String[] args) {
        Human man = new Man();
        Human woman = new Woman();
        man.sayHello();
        woman.sayHello();
        man = new Woman();
        man.sayHello();
	}
}
```

运行结果：

```
man say hello
woman say hello
woman say hello
```

导致这个现象的原因很明显，是因为这两个变量的实际类型不同，Java虚拟机是如何根据实际类型来分派方法执行版本的呢？

查看上述代码对应的字节码：

![](https://upload-images.jianshu.io/upload_images/24520697-377d7d944b7a6c5b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

16～21行是关键部分，16和20行的aload指令分别把刚刚创建的两个对象的引用压到栈顶，这两个对象是将要执行的`sayHello()`方法的所有者，称为接收者；17和21行是方法调用指令，这两条调用指令单从字节码角度来看，无论是指令（都是invokevirtual）还是参数（都是常量池中第22项的常量，注释显示了这个常量是`Human.sayHello()`的符号引用）都完全一样，但是这两句指令最终执行的目标方法并不相同。。根据《Java虚拟机规范》，invokevirtual指令的运行时解析过程大致分为以下几步：

1）找到操作数栈顶的第一个元素所指向的对象的实际类型，记作C。

2）如果在类型C中找到与常量中的描述符和简单名称都相符的方法，则进行访问权限校验，如果通过则返回这个方法的直接引用，查找过程结束；不通过则返回java.lang.IllegalAccessError异常。

3）否则，按照继承关系从下往上依次对C的各个父类进行第二步的搜索和验证过程。

4）如果始终没有找到合适的方法，则抛出java.lang.AbstractMethodError异常。

正是因为invokevirtual指令执行的第一步就是在运行期确定接收者的实际类型，所以两次调用中的invokevirtual指令并不是把常量池中方法的符号引用解析到直接引用上就结束了，还会根据方法接收者的实际类型来选择方法版本，这个过程就是Java语言中方法重写的本质。这种在运行期根据实际类型确定方法执行版本的分派过程称为动态分派。

在Java里面只有虚方法存在，字段永远不可能是虚的，换句话说，字段永远不参与多态，哪个类的方法访问某个名字的字段时，该名字指的就是这个类能看到的那个字段。当子类声明了与父类同名的字段时，虽然在子类的内存中两个字段都会存在，但是子类的字段会遮蔽父类的同名字段。请看如下代码：

```java
// 字段不参与多态
public class FieldHasNoPolymorphic {
    static class Father {
    public int money = 1;
    public Father() {
        money = 2;
        showMeTheMoney();
    }
    public void showMeTheMoney() {
    	System.out.println("I am Father, i have $" + money);
    }
    }
    static class Son extends Father {
    public int money = 3;
    public Son() {
        money = 4;
        showMeTheMoney();
    }
    public void showMeTheMoney() {
    	System.out.println("I am Son, i have $" + money);
    }
    }
    public static void main(String[] args) {
        Father gay = new Son();
        System.out.println("This gay has $" + gay.money);
    }
}
```

输出结果为：

```
I am Son, i have $0
I am Son, i have $4
This gay has $2
```

Son类在创建的时候，首先隐式调用了Father的构造函数，而Father构造函数中对showMeTheMoney()的调用是一次虚方法调用，实际执行的版本是Son::showMeTheMoney()方法，所以输出的是“I am Son”。而这时候虽然父类的money字段已经被初始化成2了，但Son::showMeTheMoney()方法中访问的却是子类的money字段，这时候结果自然还是0，因为它要到子类的构造函数执行时才会被初始化。main()的最后一句通过静态类型访问到了父类中的money，输出了2。

#### 单分派与多分派

方法的接收者与方法的参数统称为方法的宗量。根据分派基于多少种宗量，可以将分派划分为单分派和多分派两种。单分派是根据一个宗量对目标方法进行选择，多分派则是根据多于一个宗量对目标方法进行选择。

单分派、多分派演示：

```java
// 单分派、多分派演示
public class Dispatch {
    static class QQ {}
    static class _360 {}
    public static class Father {
    public void hardChoice(QQ arg) {
    	System.out.println("father choose qq");
    }
    public void hardChoice(_360 arg) {
    	System.out.println("father choose 360");
    }
    }
    public static class Son extends Father {
    public void hardChoice(QQ arg) {
    	System.out.println("son choose qq");
    }
    public void hardChoice(_360 arg) {
    	System.out.println("son choose 360");
    }
    }
    public static void main(String[] args) {
    Father father = new Father();
    Father son = new Son();
    father.hardChoice(new _360());
    	son.hardChoice(new QQ());
    }
}
```

运行结果：

```
father choose 360
son choose qq
```

首先是编译阶段中编译器的选择过程，也就是静态分派的过程。这时候选择目标方法的依据有两点：一是静态类型是Father还是Son，二是方法参数是QQ还是360。这次选择结果的最终产物是产生了两条invokevirtual指令，两条指令的参数分别为常量池中指向Father::hardChoice(360)及Father::hardChoice(QQ)方法的符号引用。因为是根据两个宗量进行选择，所以Java语言的静态分派属于多分派类型。

再看看运行阶段中虚拟机的选择，也就是动态分派的过程。在执行“son.hardChoice(new QQ())”这行代码时，更准确地说，是在执行这行代码所对应的invokevirtual指令时，由于编译期已经决定目标方法的签名必须为hardChoice(QQ)，虚拟机此时不会关心传递过来的参数“QQ”到底是“腾讯QQ”还是“奇瑞QQ”，因为这时候参数的静态类型、实际类型都对方法的选择不会构成任何影响，唯一可以影响虚拟机选择的因素只有该方法的接受者的实际类型是Father还是Son。因为只有一个宗量作为选择依据，所以Java语言的动态分派属于单分派类型

如今的Java语言是一门静态多分派、动态单分派的语言。

#### 虚方法表

动态分派是执行非常频繁的动作，而且动态分派的方法版本选择过程需要运行时在接收者类型的方法元数据中搜索合适的目标方法，因此，Java虚拟机实现基于执行性能的考虑，真正运行时一般不会如此频繁地去反复搜索类型元数据。面对这种情况，一种基础而且常见的优化手段是为类型在方法区中建立一个虚方法表。

虚方法表结构示例：

![image-20210222132817878](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\image-20210222132817878.png)

虚方法表中存放着各个方法的实际入口地址。如果某个方法在子类中没有被重写，那子类的虚方法表中的地址入口和父类相同方法的地址入口是一致的，都指向父类的实现入口。如果子类中重写了这个方法，子类虚方法表中的地址也会被替换为指向子类实现版本的入口地址。Son重写了来自Father的全部方法，因此Son的方法表没有指向Father类型数据的箭头。但是Son和Father都没有重写来自Object的方法，所以它们的方法表中所有从Object继承来的方法都指向了Object的数据类型。

## 7.3 动态类型语言支持

### 动态类型语言

动态类型语言的关键特征是它的类型检查的主体过程是在运行期而不是编译期进行的。C++和Java等就是最常用的静态类型语言。

“变量无类型而变量值才有类型”这个特点也是动态类型语言的一个核心特征。

### java.lang.invoke 包

### invokedynamic 指令

### 实战：掌控方法分派规则

## 7.4 基于栈的字节码解释执行引擎

# 8 类加载及执行子系统的案例与实战

# 9 前端编译与优化

# 10 后端编译与优化

# 11 Java 内存模型与线程

## 11.1 Java 内存模型

### 主内存与工作内存

Java内存模型的主要目的是定义程序中各种变量的访问规则，即关注在虚拟机中把变量值存储到内存和从内存中取出变量值这样的底层细节。此处的变量包括了实例字段、静态字段和构成数组对象的元素。但是不包括局部变量与方法参数，因为后者是线程私有的，不会被共享，自然就不会存在竞争问题。

Java内存模型规定了所有的变量都存储在主内存中。每条线程还有自己的工作内存，线程的工作内存中保存了被该线程使用的变量的主内存副本，线程对变量的所有操作（读取、赋值等）都必须在工作内存中进行，而不能直接读写主内存中的数据。不同的线程之间也无法直接访问对方工作内存中的变量，线程间变量值的传递均需要通过主内存来完成，线程、主内存、工作内存三者的交互关系如图所示：

![](https://upload-images.jianshu.io/upload_images/24520697-5a463788c77128a4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这里所讲的主内存、工作内存与第2章所讲的Java内存区域中的Java堆、栈、方法区等并不是同一个层次的对内存的划分，这两者基本上是没有任何关系的。

### 对于 volatile 型变量的特殊规则

当一个变量被定义成volatile之后，它将具备两项特性：第一项是保证此变量对所有线程的可见性，这里的“可见性”是指当一条线程修改了这个变量的值，新值对于其他线程来说是可以立即得知的。

关于volatile变量的可见性，经常会被开发人员误解，他们会误以为下面的描述是正确的：“volatile变量对所有线程是立即可见的，对volatile变量所有的写操作都能立刻反映到其他线程之中。换句话说，volatile变量在各个线程中是一致的，所以基于volatile变量的运算在并发下是线程安全的”。但实际上volatile变量的运算在并发下一样是不安全的，不能保证原子性，请看如下代码：

```java
// volatile 变量自增运算
public class VolatileTest {
    public static volatile int race = 0;
    public static void increase() {
    	race++;
    }
    private static final int THREADS_COUNT = 20;
    public static void main(String[] args) {
        Thread[] threads = new Thread[THREADS_COUNT];
        for (int i = 0; i < THREADS_COUNT; i++) {
            threads[i] = new Thread(new Runnable() {
                @Override
                public void run() {
                for (int i = 0; i < 10000; i++) {
                increase();
                }
                }
            });
            threads[i].start();
    	}
        // 等待所有累加线程都结束
        while (Thread.activeCount() > 1)
            Thread.yield();
        System.out.println(race);
    }
}
```

这段代码发起了20个线程，每个线程对race变量进行10000次自增操作，如果这段代码能够正确并发的话，最后输出的结果应该是200000。读者运行完这段代码之后，并不会获得期望的结果。

问题就出在自增运算“race++”之中，它由4条字节码指令构成，当getstatic指令把race的值取到操作栈顶时，volatile关键字保证了race的值在此时是正确的，但是在执行iconst_1、iadd这些指令的时候，其他线程可能已经把race的值改变了，而操作栈顶的值就变成了过期的数据，所以putstatic指令执行后就可能把较小的race值同步回主内存之中。

![](https://upload-images.jianshu.io/upload_images/24520697-39d334834dbd0765.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

由于volatile变量只能保证可见性，在不符合以下两条规则的运算场景中，我们仍然要通过加锁（使用synchronized、java.util.concurrent中的锁或原子类）来保证原子性：

- 运算结果并不依赖变量的当前值，或者能够确保只有单一的线程修改变量的值。
- 变量不需要与其他的状态变量共同参与不变约束。

使用volatile变量的第二个语义是禁止指令重排序优化，普通的变量仅会保证在该方法的执行过程中所有依赖赋值结果的地方都能获取到正确的结果，而不能保证变量赋值操作的顺序与程序代码中的执行顺序一致。

为何指令重排序会干扰程序的并发执行，请看如下演示：

```java
Map configOptions;
char[] configText;
// 此变量必须定义为volatile
volatile boolean initialized = false;
// 假设以下代码在线程A中执行
// 模拟读取配置信息，当读取完成后
// 将initialized设置为true,通知其他线程配置可用
configOptions = new HashMap();
configText = readConfigFile(fileName);
processConfigOptions(configText, configOptions);
initialized = true;
// 假设以下代码在线程B中执行
// 等待initialized为true，代表线程A已经把配置信息初始化完成
while (!initialized) {
	sleep();
}
// 使用线程A中初始化好的配置信息
doSomethingWithConfig();
```

如果定义initialized变量时没有使用volatile修饰，就可能会由于指令重排序的优化，导致位于线程A中最后一条代码“initialized=true”被提前执行（这里虽然使用Java作为伪代码，但所指的重排序优化是机器级的优
化操作，提前执行是指这条语句对应的汇编代码被提前执行），这样在线程B中使用配置信息的代码就可能出现错误，而volatile关键字则可以避免此类情况的发。

### 原子性、可见性与有序性

Java内存模型是围绕着在并发过程中如何处理原子性、可见性和有序性这三个特征来建立的。

#### 原子性

由Java内存模型来直接保证的原子性变量操作包括read、load、assign、use、store和write这六个，我们大致可以认为，基本数据类型的访问、读写都是具备原子性的（例外就是long和double的非原子性协定）。

如果应用场景需要一个更大范围的原子性保证（经常会遇到），Java内存模型还提供了lock和unlock操作来满足这种需求，尽管虚拟机未把lock和unlock操作直接开放给用户使用，但是却提供了更高层次的字节码指令monitorenter和monitorexit来隐式地使用这两个操作。这两个字节码指令反映到Java代码中就是同步块——synchronized关键字，因此在synchronized块之间的操作也具备原子性。

#### 可见性

可见性就是指当一个线程修改了共享变量的值时，其他线程能够立即得知这个修改。普通变量与volatile变量的区别是，volatile的特殊规则保证了新值能立即同步到主内存，以及每次使用前立即从主内存刷新。因此我们可以说volatile保证了多线程操作时变量的可见性，而普通变量则不能保证这一点。

除了volatile之外，Java还有两个关键字能实现可见性，它们是synchronized和final。同步块的可见性是由“对一个变量执行unlock操作之前，必须先把此变量同步回主内存中”这条规则获得的。而final关键字的可见性是指：被final修饰的字段在构造器中一旦被初始化完成，并且构造器没有把“this”的引用传递出（this引用逃逸是一件很危险的事情，其他线程有可能通过这个引用访问到“初始化了一半”的对象），那么在其他线程中就能看见final字段的值

#### 有序性

Java内存模型的有序性在前面讲解volatile时也比较详细地讨论过了，Java程序中天然的有序性可以总结为一句话：如果在本线程内观察，所有的操作都是有序的；如果在一个线程中观察另一个线程，所有的操作都是无序的。前半句是指“线程内似表现为串行的语义”，后半句是指“指令重排序”现象和“工作内存与主内存同步延迟”现象。

Java语言提供了volatile和synchronized两个关键字来保证线程之间操作的有序性，volatile关键字本身就包含了禁止指令重排序的语义，而synchronized则是由“一个变量在同一个时刻只允许一条线程对其进行lock操作”这条规则获得的，这个规则决定了持有同一个锁的两个同步块只能串行地进入。

### 先行发生远测

这个原则非常重要，它是判断数据是否存在竞争，线程是否安全的非常有用的手段。

先行发生是Java内存模型中定义的两项操作之间的偏序关系，比如说操作A先行发生于操作B，其实就是说在发生操作B之前，操作A产生的影响能被操作B观察到，“影响”包括修改了内存中共享变量的值、发送了消息、调用了方法等。

举个例子：

```java
// 以下操作在线程A中执行
i = 1;
// 以下操作在线程B中执行
j = i;
// 以下操作在线程C中执行
i = 2;
```

假设线程A中的操作“i=1”先行发生于线程B的操作“j=i”，那我们就可以确定在线程B的操作执行后，变量j的值一定是等于1，得出这个结论的依据有两个：一是根据先行发生原则，“i=1”的结果可以被观察到；二是线程C还没登场，线程A操作结束之后没有其他线程会修改变量i的值。现在再来考虑线程C，我们依然保持线程A和B之间的先行发生关系，而C出现在线程A和B的操作之间，但是C与B没有先行发生关系，那j的值会是多少呢？答案是不确定！1和2都有可能，因为线程C对变量i的影响可能会被线程B观察到，也可能不会，这时候线程B就存在读取到过期数据的风险，不具备多线程安全性。

下面是Java内存模型下一些“天然的”先行发生关系，这些先行发生关系无须任何同步器协助就已经存在，可以在编码中直接使用。如果两个操作之间的关系不在此列，并且无法从下列规则推导出来，则它们就没有顺序性保障，虚拟机可以对它们随意地进行重排序。

- 程序次序规则：在一个线程内，按照控制流顺序，书写在前面的操作先行发生于书写在后面的操作。注意，这里说的是控制流顺序而不是程序代码顺序，因为要考虑分支、循环等结构。
- 管程锁定规则：一个unlock操作先行发生于后面对同一个锁的lock操作。这里必须强调的是“同一个锁”，而“后面”是指时间上的先后。
- volatile变量规则：对一个volatile变量的写操作先行发生于后面对这个变量的读操作，这里的“后面”同样是指时间上的先后。
- 线程启动规则：Thread对象的start()方法先行发生于此线程的每一个动作。
- 线程终止规则：线程中的所有操作都先行发生于对此线程的终止检测，我们可以通过Thread::join()方法是否结束、Thread::isAlive()的返回值等手段检测线程是否已经终止执行。
- 线程中断规则：对线程interrupt()方法的调用先行发生于被中断线程的代码检测到中断事件的发生，可以通过Thread::interrupted()方法检测到是否有中断发生。
- 对象终结规则：一个对象的初始化完成（构造函数执行结束）先行发生于它的finalize()方法的开始。
- 传递性：如果操作A先行发生于操作B，操作B先行发生于操作C，那就可以得出操作A先行发生于操作C的结论。

Java语言无须任何同步手段保障就能成立的先行发生规则有且只有上面这些，下面演示一下如何使用这些规则去判定操作间是否具备顺序性，对于读写共享变量的操作来说，就是线程是否安全。

```java
private int value = 0;
pubilc void setValue(int value){
	this.value = value;
}
public int getValue(){
	return value;
}
```

假设存在线程A和B，线程A先（时间上的先后）调用了setValue(1)，然后线程B调用了同一个对象的getValue()，那么线程B收到的返回值是什么？

我们依次分析一下先行发生原则中的各项规则。由于两个方法分别由线程A和B调用，不在一个线程中，所以程序次序规则在这里不适用；由于没有同步块，自然就不会发生lock和unlock操作，所以管程锁定规则不适用；由于value变量没有被volatile关键字修饰，所以volatile变量规则不适用；后面的线程启动、终止、中断规则和对象终结规则也和这里完全没有关系。因为没有一个适用的先行发生规则，所以最后一条传递性也无从谈起，因此我们可以判定，尽管线程A在操作时间上先于线程B，但是无法确定线程B中getValue()方法的返回结果，换句话说，这里面的操作不是线程安全的。

通过上面的例子，我们可以得出结论：一个操作“时间上的先发生”不代表这个操作会是“先行发生”。如果一个操作“先行发生”，也不能推导出这个操作必定是“时间上的先发生”。一典型的例子就是指令重排序：

```java
// 以下操作在同一个线程中执行
int i = 1;
int j = 2;
```

根据程序次序规则，“int i=1”的操作先行发生于“int j=2”，但是“int j=2”的代码完全可能先被处理器执行，这并不影响先行发生原则的正确性。

衡量并发安全问题的时候不要受时间顺序的干扰，一切必须以先行发生原则为准。

### 线程的实现

目前线程是Java里面进行处理器资源调度的最基本单位。

实现线程主要有三种方式：使用内核线程实现（1：1实现），使用用户线程实现（1：N实现），使用用户线程加轻量级进程混合实现（N：M实现）。

#### 内核线程实现

使用内核线程实现的方式也被称为1：1实现。内核线程（Kernel-Level Thread，KLT）就是直接由操作系统内核（Kernel，下称内核）支持的线程，这种线程由内核来完成线程切换，内核通过操纵调度器（Scheduler）对线程进行调度，并负责将线程的任务映射到各个处理器上。每个内核线程可以视为内核的一个分身，这样操作系统就有能力同时处理多件事情。

程序一般不会直接使用内核线程，而是使用内核线程的一种高级接口——轻量级进程（Light Weight Process，LWP），轻量级进程就是我们通常意义上所讲的线程，每个轻量级进程都由一个内核线程支持。这种轻量级进程与内核线程之间1：1的关系称为一对一的线程模型。

![](https://upload-images.jianshu.io/upload_images/24520697-40ce9090ad7daec4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

由于内核线程的支持，每个轻量级进程都成为一个独立的调度单元，即使其中某一个轻量级进程在系统调用中被阻塞了，也不会影响整个进程继续工作。

#### 用户线程实现

用户线程指的是完全建立在用户空间的线程库上，系统内核不能感知到用户线程的存在及如何实现的。用户线程的建立、同步、销毁和调度完全在用户态中完成，不需要内核的帮助。如果程序实现得当，这种线程不需要切换到内核态，因此操作可以是非常快速且低消耗的，也能够支持规模更大的线程数量，部分高性能数据库中的多线程就是由用户线程实现的。这种进程与用户线程之间1：N的关系称为一对多的线程模型。

![](https://upload-images.jianshu.io/upload_images/24520697-223e882ed53d02a2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

且由于操作系统只把处理器资源分配到进程，那诸如“阻塞如何处理”“多处理器系统中如何将线程映射到其他处理器上”这类问题解决起来将会异常困难。

#### 混合实现

将内核线程与用户线程一起使用的实现方式，被称为N：M实现。在这种混合实现下，既存在用户线程，也存在轻量级进程。用户线程还是完全建立在用户空间中，因此用户线程的创建、切换、析构等操作依然廉价，并且可以支持大规模的用户线程并发。而操作系统支持的轻量级进程则作为用户线程和内核线程之间的桥梁，这样可以使用内核提供的线程调度功能及处理器映射，并且用户线程的系统调用要通过轻量级进程来
完成，这大大降低了整个进程被完全阻塞的风险。在这种混合模式中，用户线程与轻量级进程的数量比是不定的，是N：M的关

![](https://upload-images.jianshu.io/upload_images/24520697-0910534f58c0946a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### Java 线程的实现

HotSpot为例，它的每一个Java线程都是直接映射到一个操作系统原生线程来实现的，而且中间没有额外的间接结构，所以HotSpot自己是不会去干涉线程调度的（可以设置线程优先级给操作系统提供调度建议），全权交给底下的操作系统去处理，所以何时冻结或唤醒线程、该给线程分配多少处理器执行时间、该把线程安排给哪个处理器核心去执行等，都是由操作系统完成的，也都是由操作系统全权决定的。

#### 状态转换

Java语言定义了6种线程状态，在任意一个时间点中，一个线程只能有且只有其中的一种状态，并且可以通过特定的方法在不同状态之间转换。这6种状态分别是：

- 新建（New）：创建后尚未启动的线程处于这种状态。
- 运行（Runnable）：包括操作系统线程状态中的Running和Ready，也就是处于此状态的线程有可能正在执行，也有可能正在等待着操作系统为它分配执行时间。
- 无限期等待（Waiting）：处于这种状态的线程不会被分配处理器执行时间，它们要等待被其他线程显式唤醒。以下方法会让线程陷入无限期的等待状态：
  - 没有设置Timeout参数的Object::wait()方法；
  - 没有设置Timeout参数的Thread::join()方法；
  - LockSupport::park()方法。
- 限期等待（Timed Waiting）：处于这种状态的线程也不会被分配处理器执行时间，不过无须等待被其他线程显式唤醒，在一定时间之后它们会由系统自动唤醒。以下方法会让线程进入限期等待状态：
  - Thread::sleep()方法；
  - 设置了Timeout参数的Object::wait()方法；
  - 设置了Timeout参数的Thread::join()方法；
  - LockSupport::parkNanos()方法；
  - LockSupport::parkUntil()方法。
- 阻塞（Blocked）：线程被阻塞了，“阻塞状态”与“等待状态”的区别是“阻塞状态”在等待着获取到一个排它锁，这个事件将在另外一个线程放弃这个锁的时候发生；而“等待状态”则是在等待一段时间，或者唤醒动作的发生。在程序等待进入同步区域的时候，线程将进入这种状态。
- 结束（Terminated）：已终止线程的线程状态，线程已经结束执行。

状态转换图：

![](https://upload-images.jianshu.io/upload_images/24520697-ce6b2bdc6a92680a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# 12 线程安全与锁优化

## 12.1 线程安全

### Java 语言种的线程安全

按照线程安全的“安全程度”由强至弱来排序，可以将Java语言中各种操作共享的数据分为以下五类：不可变、绝对线程安全、相对线程安全、线程兼容和线程对立。

#### 不可变

在Java语言里面，不可变（Immutable）的对象一定是线程安全的，无论是对象的方法实现还是方法的调用者，都不需要再进行任何线程安全保障措施。

Java语言中，如果多线程共享的数据是一个基本数据类型，那么只要在定义时使用final关键字修饰它就可以保证它是不可变的。如果共享数据是一个对象，由于Java语言目前暂时还没有提供值类型的支持，那就需要对象自行保证其行为不会对其状态产生任何影响才行。java.lang.String类的对象实例，它是一个典型的不可变对象，用户调用它的substring()、replace()和concat()这些方法都不会影响它原来的值，只会返回一个新构造的字符串对象。

保证对象行为不影响自己状态的途径有很多种，最简单的一种就是把对象里面带有状态的变量都声明为final。例如java.lang.Integer构造函数，它通过将内部状态变量value定义为final来保障状态不变。

```java
private final int value;
public Integer(int value) {
	this.value = value;
}
```

在Java类库API中符合不可变要求的类型，除了上面提到的String之外，常用的还有枚举类型及java.lang.Number的部分子类，如Long和Double等数值包装类型、BigInteger和BigDecimal等大数据类型。

#### 绝对线程安全

绝对的线程安全能够完全满足Brian Goetz给出的线程安全的定义，这个定义其实是很严格的。在Java API中标注自己是线程安全的类，大多数都不是绝对的线程安全。

举个例子，java.util.Vector是一个线程安全的容器，因为它的add()、get()和size()等方法都是被synchronized修饰的。不过，即使它所有的方法都被修饰成synchronized，也不意味着调用它的时候就永远都不再需要同步手段了，请看如下代码：

```java
private static Vector<Integer> vector = new Vector<Integer>();
public static void main(String[] args) {
    while (true) {
        for (int i = 0; i < 10; i++) {
        	vector.add(i);
        }
        Thread removeThread = new Thread(new Runnable() {
        	@Override
            public void run() {
                for (int i = 0; i < vector.size(); i++) {
                    vector.remove(i);
                }
            }
        });
        Thread printThread = new Thread(new Runnable() {
            @Override
            public void run() {
                for (int i = 0; i < vector.size(); i++) {
                    System.out.println((vector.get(i)));
                }
             }
        });
        removeThread.start();
        printThread.start();
        //不要同时产生过多的线程，否则会导致操作系统假死
        while (Thread.activeCount() > 20);
    }
}
```

尽管这里使用到的Vector的get()、remove()和size()方法都是同步的，但是在多线程的环境中，如果不在方法调用端做额外的同步措施，使用这段代码仍然是不安全的。因为如果另一个线程恰好在错误的时间里删除了一个元素，导致序号i已经不再可用，再用i访问数组就会抛出一个ArrayIndexOutOfBoundsException异常。如果要保证这段代码能正确执行下去，需要做同步措施：

```java
Thread removeThread = new Thread(new Runnable() {
    @Override
    public void run() {
        synchronized (vector) {
            for (int i = 0; i < vector.size(); i++) {
            	vector.remove(i);
            }
        }
    }
});
Thread printThread = new Thread(new Runnable() {
    @Override
    public void run() {
        synchronized (vector) {
            for (int i = 0; i < vector.size(); i++) {
            	System.out.println((vector.get(i)));
            }
        }
    }
});
```

假如Vector一定要做到绝对的线程安全，那就必须在它内部维护一组一致性的快照访问才行，每次对其中元素进行改动都要产生新的快照，这样要付出的时间和空间成本都是非常大的。

#### 相对线程安全

相对线程安全就是我们通常意义上所讲的线程安全，它需要保证对这个对象单次的操作是线程安全的，我们在调用的时候不需要进行额外的保障措施，但是对于一些特定顺序的连续调用，就可能需要在调用端使用额外的同步手段来保证调用的正确性。上述两部分代码就是相对线程安全的案例。

在Java语言中，大部分声称线程安全的类都属于这种类型，例如Vector、HashTable、Collections的synchronizedCollection()方法包装的集合等。

#### 线程兼容

线程兼容是指对象本身并不是线程安全的，但是可以通过在调用端正确地使用同步手段来保证对象在并发环境中可以安全地使用。我们平常说一个类不是线程安全的，通常就是指这种情况。Java类库API中大部分的类都是线程兼容的，如与前面的Vector和HashTable相对应的集合类ArrayList和HashMap等。

#### 线程对立

线程对立是指不管调用端是否采取了同步措施，都无法在多线程环境中并发使用代码。由于Java语言天生就支持多线程的特性，线程对立这种排斥多线程的代码是很少出现的，而且通常都是有害的，应当尽量避免。

一个线程对立的例子是Thread类的suspend()和resume()方法。如果有两个线程同时持有一个线程对象，一个尝试去中断线程，一个尝试去恢复线程，在并发进行的情况下，无论调用时是否进行了同步，目标线程都存在死锁风险——假如suspend()中断的线程就是即将要执行resume()的那个线程，那就肯定要产生死锁了。也正是这个原因，suspend()和resume()方法都已经被声明废弃了。常见的线程对立的操作还有System.setIn()、Sytem.setOut()和System.runFinalizersOnExit()等。

### 线程安全的实现方法

#### 互斥同步

互斥同步（Mutual Exclusion & Synchronization）是一种最常见也是最主要的并发正确性保障手段。同步是指在多个线程并发访问共享数据时，保证共享数据在同一个时刻只被一条（或者是一些，当使用信号量的时候）线程使用。而互斥是实现同步的一种手段，临界区（Critical Section）、互斥量（Mutex）和信号量（Semaphore）都是常见的互斥实现方式。因此在“互斥同步”这四个字里面，互斥是因，同步是果；互斥是方法，同步是目的。

在Java里面，最基本的互斥同步手段就是synchronized关键字，这是一种块结构的同步语法synchronized关键字经过Javac编译之后，会在同步块的前后分别形成monitorenter和monitorexit这两个字节码指令。这两个字节码指令都需要一个reference类型的参数来指明要锁定和解锁的对象。如果Java源码中的synchronized明确指定了对象参数，那就以这个对象的引用作为reference；如果没有明确指定，那将根据synchronized修饰的方法类型（如实例方法或类方法），来决定是取代码所在的对象实例还是取类型对应的Class对象来作为线程要持有的锁。

在执行monitorenter指令时，首先要去尝试获取对象的锁。如果这个对象没被锁定，或者当前线程已经持有了那个对象的锁，就把锁的计数器的值增加一，而在执行monitorexit指令时会将锁计数器的值减一。一旦计数器的值为零，锁随即就被释放了。如果获取对象锁失败，那当前线程就应当被阻塞等待，直到请求锁定的对象被持有它的线程释放为止。

被synchronized修饰的同步块对同一条线程来说是可重入的。这意味着同一线程反复进入同步块也不会出现自己把自己锁死的情况。

被synchronized修饰的同步块在持有锁的线程执行完毕并释放锁之前，会无条件地阻塞后面其他线程的进入。这意味着无法像处理某些数据库中的锁那样，强制已获取锁的线程释放锁；也无法强制正在等待锁的线程中断等待或超时退出。

从执行成本的角度看，持有锁是一个重量级（Heavy-Weight）的操作。主流Java虚拟机实现中，Java的线程是映射到操作系统的原生内核线程之上的，如果要阻塞或唤醒一条线程，则需要操作系统来帮忙完成，这就不可避免地陷入用户态到核心态的转换中，进行这种状态转换需要耗费很多的处理器时间。

自JDK 5起，Java类库中新提供了java.util.concurrent包（J.U.C包），其中的java.util.concurrent.locks.Lock接口便成了Java的另一种全新的互斥同步手段。基于Lock接口，用户能够
以非块结构来实现互斥同步，从而摆脱了语言特性的束缚，改为在类库层面去实现同步。

重入锁（ReentrantLock）是Lock接口最常见的一种实现，顾名思义，它与synchronized一样是可重入的。在基本用法上，ReentrantLock也与synchronized很相似，只是代码写法上稍有区别而已。不过，ReentrantLock与synchronized相比增加了一些高级功能，主要有以下三项：等待可中断、可实现公平锁及锁可以绑定多个条件。

#### 非阻塞同步

同步也被称为阻塞同步，互斥同步属于一种悲观的并发策略，其总是认为只要不去做正确的同步措施（例如加锁），那就肯定会出现问题，无论共享的数据是否真的会出现竞争，它都会进行加锁，这将会导致用户态到核心态转换、维护锁计数器和检查是否有被阻塞的线程需要被唤醒等开销。随着硬件指令集的发展，已经有了另外一个选择，基于冲突检测的乐观并发策略，通俗地说就是不管风险，先进行操作，如果没有其他线程争用共享数据，那操作就直接成功了；如果共享的数据的确被争用，产生了冲突，那再进行其他的补偿措施，最常用的补偿措施是不断地重试，直到出现没有竞争的共享数据为止。这种乐观并发策略的实现不再需要把线程阻塞挂起，因此这种同步操作被称为非阻塞同步，使用这种措施的代码也常被称为无锁编程。

因为无锁并发要求操作和冲突检测这两个步骤具备原子性，如果这里再使用互斥同步来保证就完全失去意义了，所以我们只能靠硬件来实现这件事情，硬件保证某些从语义上看起来需要多次操作的行为可以只通过一
条处理器指令就能完成。以“比较并交换（Compare-and-Swap，CAS）”这条指令来说明。

CAS指令需要有三个操作数，分别是内存位置（在Java中可以简单地理解为变量的内存地址，用V表示）、旧的预期值（用A表示）和准备设置的新值（用B表示）。CAS指令执行时，当且仅当V符合A时，处理器才会用B更新V的值，否则它就不执行更新。但是，不管是否更新了V的值，都会返回V的旧值，上述的处理过程是一个原子操作，执行期间不会被其他线程中断。

在JDK 5之后，Java类库中才开始使用CAS操作，该操作由sun.misc.Unsafe类里面的compareAndSwapInt()和compareAndSwapLong()等几个方法包装提供。HotSpot虚拟机在内部对这些方法做了特殊处理，即时编译出来的结果就是一条平台相关的处理器CAS指令，没有方法调用的过程。不过由于Unsafe类在设计上就不是提供给用户程序调用的类，因此在JDK 9之前只有Java类库可以使用CAS，譬如J.U.C包里面的整数原子类，其中的compareAndSet()和getAndIncrement()等方法都使用了Unsafe类的CAS操作来实现。而如果用户程序也有使用CAS操作的需求，那要么就采用反射手段突破Unsafe的访问限制，要么就只能通过Java类库API来间接使用它。直到JDK 9之后，Java类库才在VarHandle类里开放了面向用户程序使用的CAS操作。

#### 无同步方案

要保证线程安全，也并非一定要进行阻塞或非阻塞同步，同步与线程安全两者没有必然的联系。如果能让一个方法本来就不涉及共享数据，那它自然就不需要任何同步措施去保证其正确性，因此会有一些代码天生就是线程安全的，笔者简单介绍其中的两类。

可重入代码：这种代码又称纯代码，是指可以在代码执行的任何时刻中断它，转而去执行另外一段代码（包括递归调用它本身），而在控制权返回后，原来的程序不会出现任何错误，也不会对结果有所影响。在特指多线程的上下文语境里，可以认为可重入代码是线程安全代码的一个真子集，这意味着相对线程安全来说，可重入性是更为基础的特性，它可以保证代码线程安全，即所有可重入的代码都是线程安全的，

可重入代码有一些共同的特征，例如，不依赖全局变量、存储在堆上的数据和公用的系统资源，用到的状态量都由参数中传入，不调用非可重入的方法等。我们可以通过一个比较简单的原则来判断代码是否具备可重入性：如果一个方法的返回结果是可以预测的，只要输入了相同的数据，就都能返回相同的结果，那它就满足可重入性的要求，当然也就是线程安全的。

线程本地存储（Thread Local Storage）：如果一段代码中所需要的数据必须与其他代码共享，那就看看这些共享数据的代码是否能保证在同一个线程中执行。如果能保证，我们就可以把共享数据的可见范围限制在同一个线程之内，这样，无须同步也能保证线程之间不出现数据争用的问题。

符合这种特点的应用并不少见，大部分使用消费队列的架构模式（如“生产者-消费者”模式）都会将产品的消费过程限制在一个线程中消费完。

Java语言中，如果一个变量要被多线程访问，可以使用volatile关键字将它声明为“易变的”；如果一个变量只要被某个线程独享，Java中就没有类似C++中__declspec(thread)[7]这样的关键字去修饰，不过我们还是可以通过java.lang.ThreadLocal类来实现线程本地存储的功能。每一个线程的Thread对象中都有一个ThreadLocalMap对象，这个对象存储了一组以ThreadLocal.threadLocalHashCode为键，以本地线程变量为值的K-V值对，ThreadLocal对象就是当前线程的ThreadLocalMap的访问入口，每一个ThreadLocal对象都包含了一个独一无二的threadLocalHashCode值，使用这个值就可以在线程K-V值对中找回对应的本地线程变量。

## 锁12.2优化

### 自旋锁与自适应自旋

互斥同步对性能最大的影响是阻塞的实现，挂起线程和恢复线程的操作都需要转入内核态中完成，这些操作给Java虚拟机的并发性能带来了很大的压力。但是在很多时候，共享数据的锁定状态只会持续很短的一段时间，为了这段时间去挂起和恢复线程并不值得。如果物理机器有一个以上的处理器或者处理器核心，能让两个或以上的线程同时并行执行，我们就可以让后面请求锁的那个线程“稍等一会”，但不放弃处理器的执行时间，看看持有锁的线程是否很快就会释放锁。为了让线程等待，我们只须让线程执行一个忙循环（自旋），这项技术就是所谓的自旋锁。

自旋锁在JDK 1.4.2中就已经引入，只不过默认是关闭的，可以使用-XX：+UseSpinning参数来开启，在JDK 6中就已经改为默认开启了。自旋等待不能代替阻塞，且先不说对处理器数量的要求，自旋等待本身虽然避免了线程切换的开销，但它是要占用处理器时间的，所以如果锁被占用的时间很短，自旋等待的效果就会非常好，反之如果锁被占用的时间很长，那么自旋的线程只会白白消耗处理器资源，而不会做任何有价值的工作，这就会带来性能的浪费。因此自旋等待的时间必须有一定的限度，如果自旋超过了限定的次数仍然没有成功获得锁，就应当使用传统的方式去挂起线程。自旋次数的默认值是十次，用户也可以使用参数-XX：PreBlockSpin来自行更改。

不过无论是默认值还是用户指定的自旋次数，对整个Java虚拟机中所有的锁来说都是相同的。在JDK 6中对自旋锁的优化，引入了自适应的自旋。自适应意味着自旋的时间不再是固定的了，而是由前一次在同一个锁上的自旋时间及锁的拥有者的状态来决定的。如果在同一个锁对象上，自旋等待刚刚成功获得过锁，并且持有锁的线程正在运行中，那么虚拟机就会认为这次自旋也很有可能再次成功，进而允许自旋等待持续相对更长的时间，比如持续100次忙循环。另一方面，如果对于某个锁，自旋很少成功获得过锁，那在以后要获取这个锁时将有可能直接省略掉自旋过程，以避免浪费处理器资源。有了自适应自旋，随着程序运行时间的增长及性能监控信息的不断完善，虚拟机对程序锁的状况预测就会越来越精准，虚拟机就会变得越来越“聪明”了。

### 锁消除

锁消除是指虚拟机即时编译器在运行时，对一些代码要求同步，但是对被检测到不可能存在共享数据竞争的锁进行消除。锁消除的主要判定依据来源于逃逸分析的数据支持，如果判断到一段代码中，在堆上的所有数据都不会逃逸出去被其他线程访问到，那就可以把它们当作栈上数据对待，认为它们是线程私有的，同步加锁自然就无须再进行。

### 锁粗化

原则上，我们在编写代码的时候，总是推荐将同步块的作用范围限制得尽量小——只在共享数据的实际作用域中才进行同步，这样是为了使得需要同步的操作数量尽可能变少，即使存在锁竞争，等待锁的线程也能尽可能快地拿到锁。

大多数情况下，上面的原则都是正确的，但是如果一系列的连续操作都对同一个对象反复加锁和解锁，甚至加锁操作是出现在循环体之中的，那即使没有线程竞争，频繁地进行互斥同步操作也会导致不必要的性能损耗。

如果虚拟机探测到有一串零碎的操作都对同一个对象加锁，将会把加锁同步的范围扩展（粗化）到整个操作序列的外部。

### 轻量级锁

“轻量级”是相对于使用操作系统互斥量来实现的传统锁而言的，因此传统的锁机制就被称为“重量级”锁。轻量级锁并不是用来代替重量级锁的，它设计的初衷是在没有多线程竞争的前提下，减少传统的重量级锁使用操作系统互斥量产生的性能消耗。

HotSpot虚拟机的对象头分为两部分，第一部分用于存储对象自身的运行时数据，如哈希码、GC分代年龄等。这部分数据的长度在32位和64位的Java虚拟机中分别会占用32个或64个比特，官方称它为“MarkWord”。这部分是实现轻量级锁和偏向锁的关键。另外一部分用于存储指向方法区对象类型数据的指针，如果是数组对象，还会有一个额外的部分用于存储数组长度。这些对象内存布局的详细内容，我们已经在第2章中学习过，在此不再赘述，只针对锁的角度做进一步细化。

由于对象头信息是与对象自身定义的数据无关的额外存储成本，考虑到Java虚拟机的空间使用效率，Mark Word被设计成一个非固定的动态数据结构，以便在极小的空间内存储尽量多的信息。它会根据对象的状态复用自己的存储空间。例如在32位的HotSpot虚拟机中，对象未被锁定的状态下，Mark Word的32个比特空间里的25个比特将用于存储对象哈希码，4个比特用于存储对象分代年龄，2个比特用于存储锁标志位，还有1个比特固定为0（这表示未进入偏向模式）。对象除了未被锁定的正常状态外，还有轻量级锁定、重量级锁定、GC标记、可偏向等几种不同状态，这些状态下对象头的存储内容如下表所示。

![](https://upload-images.jianshu.io/upload_images/24520697-d673ef1862f8c9a7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

轻量级锁的工作过程：在代码即将进入同步块的时候，如果此同步对象没有被锁定（锁标志位为“01”状态），虚拟机首先将在当前线程的栈帧中建立一个名为锁记录（Lock Record）的空间，用于存储锁对象目前的Mark Word的拷贝（官方为这份拷贝加了一个Displaced前缀，即Displaced Mark Word），这时候线程堆栈与对象头的状态如图所示：

![](https://upload-images.jianshu.io/upload_images/24520697-2f7db646538b8e52.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

然后，虚拟机将使用CAS操作尝试把对象的Mark Word更新为指向Lock Record的指针。如果这个更新动作成功了，即代表该线程拥有了这个对象的锁，并且对象Mark Word的锁标志位（Mark Word的最后两个比特）将转变为“00”，表示此对象处于轻量级锁定状态。这时候线程堆栈与对象头的状态如图示：

![](https://upload-images.jianshu.io/upload_images/24520697-bae24833998a3311.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

如果这个更新操作失败了，那就意味着至少存在一条线程与当前线程竞争获取该对象的锁。虚拟机首先会检查对象的Mark Word是否指向当前线程的栈帧，如果是，说明当前线程已经拥有了这个对象的锁，那直接进入同步块继续执行就可以了，否则就说明这个锁对象已经被其他线程抢占了。如果出现两条以上的线程争用同一个锁的情况，那轻量级锁就不再有效，必须要膨胀为重量级锁，锁标志的状态值变为“10”，此时Mark Word中存储的就是指向重量级锁（互斥量）的指针，后面等待锁的线程也必须进入阻塞状态。

上面描述的是轻量级锁的加锁过程，它的解锁过程也同样是通过CAS操作来进行的，如果对象的Mark Word仍然指向线程的锁记录，那就用CAS操作把对象当前的Mark Word和线程中复制的DisplacedMark Word替换回来。假如能够成功替换，那整个同步过程就顺利完成了；如果替换失败，则说明有其他线程尝试过获取该锁，就要在释放锁的同时，唤醒被挂起的线程。

轻量级锁能提升程序同步性能的依据是“对于绝大部分的锁，在整个同步周期内都是不存在竞争的”这一经验法则。如果没有竞争，轻量级锁便通过CAS操作成功避免了使用互斥量的开销；但如果确实存在锁竞争，除了互斥量的本身开销外，还额外发生了CAS操作的开销。因此在有竞争的情况下，轻量级锁反而会比传统的重量级锁更慢。

### 偏向锁

它的目的是消除数据在无竞争情况下的同步原语，进一步提高程序的运行性能。如果说轻量级锁是在无竞争的情况下使用CAS操作去消除同步使用的互斥量，那偏向锁就是在无竞争的情况下把整个同步都消除掉，连CAS操作都不去做了。

这个锁会偏向于第一个获得它的线程，如果在接下来的执行过程中，该锁一直没有被其他的线程获取，则持有偏向锁的线程将永远不需要再进行同步。

假设当前虚拟机启用了偏向锁（启用参数-XX：+UseBiased Locking，这是自JDK 6起HotSpot虚拟机的默认值），那么当锁对象第一次被线程获取的时候，虚拟机将会把对象头中的标志位设置为“01”、把偏向模式设置为“1”，表示进入偏向模式。同时使用CAS操作把获取到这个锁的线程的ID记录在对象的Mark Word之中。如果CAS操作成功，持有偏向锁的线程以后每次进入这个锁相关的同步块时，虚拟机都可以不再进行任何同步操作（例如加锁、解锁及对Mark Word的更新操作等）。

一旦出现另外一个线程去尝试获取这个锁的情况，偏向模式就马上宣告结束。根据锁对象目前是否处于被锁定的状态决定是否撤销偏向（偏向模式设置为“0”），撤销后标志位恢复到未锁定（标志位为“01”）或轻量级锁定（标志位为“00”）的状态，后续的同步操作就按照上面介绍的轻量级锁那样去执行。偏向锁、轻量级锁的状态转化及对象Mark Word的关系如图所示：

![image-20210223043221200](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\image-20210223043221200.png)

当对象进入偏向状态的时候，Mark Word大部分的空间（23个比特）都用于存储持有锁的线程ID了，这部分空间占用了原有存储对象哈希码的位置，那原来对象的哈希码怎么办呢？

在Java语言里面一个对象如果计算过哈希码，就应该一直保持该值不变（强烈推荐但不强制，因为用户可以重载hashCode()方法按自己的意愿返回哈希码），否则很多依赖对象哈希码的API都可能存在出错风险。而作为绝大多数对象哈希码来源的Object::hashCode()方法，返回的是对象的一致性哈希码（Identity Hash Code），这个值是能强制保证不变的，它通过在对象头中存储计算结果来保证第一次计算之后，再次调用该方法取到的哈希码值永远不会再发生改变。因此，当一个对象已经计算过一致性哈希码后，它就再也无法进入偏向锁状态了；而当一个对象当前正处于偏向锁状态，又收到需要计算其一致性哈希码请求[1]时，它的偏向状态会被立即撤销，并且锁会膨胀为重量级锁。在重量级锁的实现中，对象头指向了重量级锁的位置，代表重量级锁的ObjectMonitor类里有字段可以记录非加锁状态（标志位为“01”）下的Mark Word，其中自然可以存储原来的哈希码。

偏向锁可以提高带有同步但无竞争的程序性能，但它同样是一个带有效益权衡（Trade Off）性质的优化，也就是说它并非总是对程序运行有利。如果程序中大多数的锁都总是被多个不同的线程访问，那偏向模式就是多余的。在具体问题具体分析的前提下，有时候使用参数-XX：-UseBiasedLocking来禁止偏向锁优化反而可以提升性能。