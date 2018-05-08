Rxjava源码阅读笔记


本次阅读目的:

1. 理解Rxjava 重要的发布者/订阅者 等几个重要角色
2. 通过一个简单例子理解整个事件发布订阅流程
3. 线程调度的过程
4. 理解RxjavaPlugins类的作用
5. 总结


# 几个重要的类 #

在Rxjava 中有下面几个重要的事件发布(被观察者)的类

Flowable : 0...N的流,支持响应流和背压
Observable: 0...N的流,不支持背压
Single:一个流,某个item 或者 error
Comletable : 一个流,没有items,只有完成、错误
Maybe :只有一个items或者错误

# 事件发布流程 #


这里以常用的Flowable为例：


	`Flowable.just(1)//①  (②③④⑤⑥⑦⑧⑨)
                .map { it.toString() }//②
                .subscribeOn(Schedulers.io())//③
                .observeOn(AndroidSchedulers.mainThread())//④
				//⑤	
                .subscribe(object :Subscriber<String>{
                    override fun onComplete() {
                    }

                    override fun onSubscribe(s: Subscription?) {
                        s?.request(100)
                    }

                    override fun onNext(t: String?) {
                    }

                    override fun onError(t: Throwable?) {
                    }
                })`


## 对象创建分析 ##

1. Flowable.just(1)

源码: 
	`public static <T> Flowable<T> just(T item) {
        ObjectHelper.requireNonNull(item, "item is null");
        return RxJavaPlugins.onAssembly(new FlowableJust<T>(item));
    }`

- 里面先检查传入的item不为空
- 创建FlowableJust对象(构造内传入item T)
- 执行相关的钩子函数
	`public static <T> Flowable<T> onAssembly(@NonNull Flowable<T> source) {
        Function<? super Flowable, ? extends Flowable> f = onFlowableAssembly;
        if (f != null) {
            return apply(f, source);
        }
        return source;
    }`
	
而钩子函数的概念是在执行目标程序之前进行加工处理,而在这里是onFlowableAssembly静态变量为空所以不进行加工处理直接返回源Flowable

(注,此时的Flowable : FlowableJust)


2. map{}操作后

	`public final <R> Flowable<R> map(Function<? super T, ? extends R> mapper) {
        ObjectHelper.requireNonNull(mapper, "mapper is null");
        return RxJavaPlugins.onAssembly(new FlowableMap<T, R>(this, mapper));
    }`

同just分析,不过这时候的Flowable对象变为FlowableMap
构造内传入上个Flowable(就是FlowableJust)和mapper函数

3.subscribeOn

传入Schedulers.io() 实际是IoScheduler对象,而IoScheduler对象构造内传入一个RxThreadFactory对象,而RxThreadFactory其实就是一个线程工厂实现对象

- 把上个Flowable对象(FlowableMap),Scheduler对象(IoScheduler)
false(判断是否是非预定请求)


4.observeOn(AndroidSchedulers.mainThread())

- 这里传入的调度器实际上是HandlerScheduler对象,它的构建传入了一个主线程的Handler对象(`new HandlerScheduler(new Handler(Looper.getMainLooper()));`)
- 这个把上个Flowable(FlowableSubscribeOn),调度器(HandlerScheduler),false(是否延迟误差),128 (int buffsize)

5.subscribe(s) 订阅

首先肯定是贴一下源码
	`@BackpressureSupport(BackpressureKind.SPECIAL)
    @SchedulerSupport(SchedulerSupport.NONE)
    @Override
    public final void subscribe(Subscriber<? super T> s) {
        ObjectHelper.requireNonNull(s, "s is null");
        try {
            s = RxJavaPlugins.onSubscribe(this, s);

            ObjectHelper.requireNonNull(s, "Plugin returned null Subscriber");

            subscribeActual(s);
        } catch (NullPointerException e) { // NOPMD
            throw e;
        } catch (Throwable e) {
            Exceptions.throwIfFatal(e);
            // can't call onError because no way to know if a Subscription has been set or not
            // can't call onSubscribe because the call might have set a Subscription already
            RxJavaPlugins.onError(e);

            NullPointerException npe = new NullPointerException("Actually not, but can't throw other exceptions due to RS");
            npe.initCause(e);
            throw npe;
        }
    }`

里面主要的两个方法是`RxJavaPlugins.onSubscribe(this, s)`,`subscribeActual(s)`


RxJavaPlugins.onSubscribe也是钩子函数,同上面的分析

