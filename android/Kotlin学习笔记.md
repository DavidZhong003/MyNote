# 泛型 #

## 型变(type variance)的概念 ##

**型变** 指能否对参数类型进行子类型转换

例如有两个类型A,B

- 不变型(invariance) 

	List< A> 和List< B>没有关系

- 协变(covariance)

	List< A> 是List< B>的子类型(subtype)

- 逆变(contravariance)

	List< A> 是List< B>的超类型

**生产者和消费者** 

生产者(Producer)方法指 只能从中读取的对象

fun get():T

消费者(Consumer)方法指 只能写入对象

fun set(t:T)


## java中型变处理 ##

对于协变使用 ? extends A  
A及A的子类

对于逆变使用 ? super A
A及A的超类


## Kotlin中型变处理 ##

### 声明处型变（declaration-site variance） ###

- out

当一个类 C 的类型参数 T 被声明为 out 时，它就只能出现在 C 的成员的输出-位置，但回报是 C<Base> 可以安全地作为 C<Derived>的超类。

- in
 super String
它使得一个类型参数逆变：只可以被消费而不可以被生产。

- 星投影

Kotlin 为此提供了所谓的星投影语法：

对于 Foo <out T : TUpper>，其中 T 是一个具有上界 TUpper 的协变类型参数，Foo <*> 等价于 Foo <out TUpper>。 这意味着当 T 未知时，你可以安全地从 Foo <*> 读取 TUpper 的值。
对于 Foo <in T>，其中 T 是一个逆变类型参数，Foo <*> 等价于 Foo <in Nothing>。 这意味着当 T 未知时，没有什么可以以安全的方式写入 Foo <*>。
对于 Foo <T : TUpper>，其中 T 是一个具有上界 TUpper 的不型变类型参数，Foo<*> 对于读取值时等价于 Foo<out TUpper> 而对于写值时等价于 Foo<in Nothing>。
如果泛型类型具有多个类型参数，则每个类型参数都可以单独投影。 例如，如果类型被声明为 interface Function <in T, out U>，我们可以想象以下星投影：

Function<*, String> 表示 Function<in Nothing, String>；
Function<Int, *> 表示 Function<Int, out Any?>；
Function<*, *> 表示 Function<in Nothing, out Any?>。
注意：星投影非常像 Java 的原始类型，但是安全。


# 委托 #

传统装饰器设计模式: 创建新类,实现与原始类一样的接口,把原来的类作为一个字段保存,与原类同样的行为不需要修改,只需要转发给原始类.


## 类委托 ##

装饰器模式简写,

	`class Derived(b: Base) : Base by b`

编译器默认把方法转发给b

## 属性委托 ##

1. 标准委托


标准库默认实现的几个工厂方法

- 延迟属性lazy

第一次调用 get() 会执行已传递给 lazy() 的 lambda 表达式并记录结果， 后续调用 get() 只是返回记录的结果。

默认lazy属性求值加了同步锁,

属性初始化不必同步,lazy传入LazyThreadSafetyMode.PUBLICATION

属性初始化发生在每个线程,LazyThreadSafetyMode.NONE