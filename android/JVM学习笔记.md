# 内存模型 #

先祭出一张经典的内存结构图

![JVM](http://img.my.csdn.net/uploads/201209/29/1348934141_8447.jpg)


其中内存区域中虚拟机栈（VM stack）,本地方法栈（Native Method Stack），程序计数器（Program Counter Register）是线程私有，
方法区（Method Area），堆（Heap）为线程共有。

## 程序计数器 ##

主要作用是记录当前线程执行的字节码行号，分支、循环、异常、线程恢复等很多功能都依赖它来完成。如果该线程正在执行java方法，这个计数器记录字节码指令地址，而执行Native方法时候，程序计数器的值为空（Undefined）。

## Java虚拟机栈 ##

描述Java方法执行内存的模型。
该区域线程私有。每个线程创建都会创建出相应的栈（VM stack）,它存储了局部变量，方法出口，动态链接等信息。而一个方法的调用到执行完成就是对应一个栈帧在VM stack中入栈到出栈的过程。
而局部变量表存放基本数据类型以及对象引用类型（reference 类型），以及记录字节码指令地址（returnAddress类型）。
可抛出StackOverFlow、OutOfMemory异常。

## 本地方法栈 ##

同java虚拟机栈，不同的是记录的native方法

## Java堆 ##
Java虚拟机中内存最大的部分，该区域的特点是：
1. 线程共享
2. 存储对象实例以及数组
3. 垃圾回收机制主要管理区域
4. 物理内存空间不连续
5. 无内存空间且无法扩展时候抛出OOM

## 方法区 ##

存储被虚拟机加载的类信息，常量，静态变量等数据。和Java堆一样被线程共享，可动态扩展，无内存且无法扩展时候也抛出OOM。垃圾回收一般很难触及此区域。主要是常量池回收和类的卸载。

### 运行时常量池 ###

方法区的一部分，存储编译时生成的字面量和符号引用。不同于Class文件常量池，该区域具有动态性，存储的常量不要求一定要编译时期产生。运行时也能把新的常量放入池中。

## 直接内存 ##
不是虚拟机运行时数据区中的一部分。在NIO中（jdk1.4后引入），使用Native函数直接分配堆外内存，然后通过java堆中DirecByteBuffer对象管理


# 垃圾回收 #
垃圾回收机制主要针对Java堆(Heap)和方法区(Method Area),而虚拟机栈等线程私有区域随着线程销毁而被回收所以不参与垃圾回收。
## 一个对象是否可以被回收 ##
垃圾回收要做的第一件事就是确定对象的存活情况，
### 引用计数算法 ###
给对象添加一个引用计数器，每当对象被一个地方引用就加一，引用失效时候则减一。



- 优点是：实现简单，判断效率高。


- 缺点是：两个对象互相引用时候，GC无法回收

### 可达性分析算法 ###
以GCRoots对象作为起始点,向下搜索,能够到达的对象都是可达的,证明引用链(Reference Chain)上的对象可用。反之证明不可用。在Java中能够作为GCRoots对象的有，


- 虚拟机栈中引用的对象
- 方法区中静态属性引用的对象
- 方法区中常量引用的对象
- 本地方法栈中引用的对象

#### 引用概念 ####
无论引用计数法还是可达性分析算法，判断对象是否可用都和引用的概念有关。JDK1.2之前，java引用的定义（reference类型数据指向的内存起始地址，这块内存代表一个引用）只有被引用和没有引用区分。为了丰富引用，进行了扩充：

1. 强引用（Strong Reference）

	只要强引用存在，永远不会回收掉被引用的对象

2. 软引用（Soft Reference）

	有用但并非必需的对象，内存溢出前将会回收对象
    
		Object obj = new Object();
		SoftReference<Object> sf = new SoftReference<Object>(obj);
		obj = null;
		sf.get();

3. 弱引用（Weak Reference）

	非必需对象，存活到下次GC前

		Object obj = new Object();
		WeakReference<Object> wf = new WeakReference<Object>(obj);
		obj = null;	
		wf.get();
		wf.isEnQueued();
4. 虚引用 （Phantom Reference）
	
	无法通过虚引用来获取一个对象实例。为一个对象设置虚引用的目的是当这个对象被系统回收时候得到一个通知。

		Object obj = new Object();
		PhantomReference<Object> pf = new PhantomReference<Object>(obj);
		obj=null;
		pf.get();
		pf.isEnQueued();

#### 对象死亡过程 ####
一个对象死亡要经历两次标记过程：

1. 没有与GCRoots对象的引用链，被标记第一次。
2. 分析是否有必要执行finalize方法（当该方法没有被覆盖或者被判断没被调用过则被判定没必要执行），如果有必要执行则放入队列，由Finalizer线程执行。
3. 执行finalize方法，如果再无与GCRoots的引用链，被标记第二次然后被回收。

#### 方法区对象的回收 ####
方法区的回收机率较小，主要是废弃常量和无用的类。

1. 废弃常量回收

	无任何对象引用了常量池中某个常量，当发生内存回收而且有必要的话，这个常量将被清理。

2. 无用的类回收

	无用的类筛选条件
		
		- Java堆中不存在该类的实例
		- 加载该类的ClassLoader对象被回收
		- 对应的java.lang.Class 对象没被引用

	可以通过 -Xnoclassgc 参数来控制是否对类进行卸载。

## 垃圾回收算法 ##
	
### 标记-清除 ###
垃圾回收的基础算法，整个过程分为标记和清除两个阶段。第一阶段经过可达性分析后进行对象标记，第二阶段对标记可回收的对象进行清除。
	
![标记-清除](https://upload-images.jianshu.io/upload_images/2843224-352ca6b5f3df119b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/637)

这个算法的缺点是，1标记和清除两个过程效率低。2，容易产生空间碎片。
### 标记-整理 ###
同标记-清除算法，在清除后把剩下的对象往后面移动。


![标记-整理](https://upload-images.jianshu.io/upload_images/2843224-8390dfb29dbdcb6a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)

这个算法虽然不会产生空间碎片，但同样效率较低
### 复制 ###
解决了标记的效率问题。把内存均分两部分，每次使用其中一部分，当一块内存用完把这块存活的对象复制到另外一块上，然后清空这块区域。

![复制算法](https://upload-images.jianshu.io/upload_images/2843224-258224e19e16e13a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/649)

优点是不产生碎片以及效率高，但内存利用率低，当大多数对象存活时候，复制的效率也低。
### 分代 ###
现在商用虚拟机常用算法，根据对象的生存时间把内存分为几块。一般把Java堆划分新生代和老年代。新生代使用复制算法，老年代使用标记-清除或者标记-整理算法。

![分代算法](https://upload-images.jianshu.io/upload_images/2843224-e9a5603e920a6bbe.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)

当对象较大或者长时间存活时放入老年代否则放入Eden区域。

而新生代的复制算法经过改进。不是均分两个区域。而是划分出一个较大的Eden和两个相等较小的Survivor区域，回收时把Eden和Survivor存活对象复制到另一块Survivor区域。
## 垃圾收集器 ##

## 内存分配以及回收策略 ##

## GC触发条件 ##


# 类加载机制 #

