# 简介
前段时间项目上要使用单Activity+多Fragment结构。[Fragmentation](https://github.com/YoKeyword/Fragmentation)这个库极大的方便我们对Fragment进行管理，可轻松进行start，pop，hide，show等操作，而且提供调试工具可轻松查看Fragment栈视图，降低开发难度。具体引入使用可参看上面github链接。下面对这个库进行简单的源码分析。


# 源码分析

在使用时候我们常常需要继承SupportActivity或者实现ISupportActivity接口，然后在onCreate方法内调用loadRootFragment方法。

```kotlin
if (homeFragment == null) {
    loadRootFragment(R.id.fl_bottom, getMainFragment() as ISupportFragment)
}
```

现在我们就从这个方法开始入手。


## loadRootFragment

**1. SupportActivity 中的 loadRootFragment**

loadRootFragment这个方法做了什么工作呢，看下它的源码

```java
public void loadRootFragment(int containerId, @NonNull ISupportFragment toFragment) {
    mDelegate.loadRootFragment(containerId, toFragment);
}
```

上面可以看出实际load工作交给mDelegate这个对象做，而这个mDelegate对象又是什么呢？

我们继续查看。

```java  
final SupportActivityDelegate mDelegate = new SupportActivityDelegate(this);
```

原来mDelegate 是 SupportActivityDelegate ，supportActivity中的ISupportActivity接口的大部分实现工作都交给这个代理类去做。而这个代理类也是这个框架的核心，后面很多分析都会接触到它。

**2. SupportActivityDelegate中的loadRootFragment**

来看下代理类中的loadRootFragment 方法：

```java
public void loadRootFragment(int containerId, ISupportFragment toFragment) {
    loadRootFragment(containerId, toFragment, true, false);
}

public void loadRootFragment(int containerId, ISupportFragment toFragment, boolean        addToBackStack, boolean allowAnimation) {
    mTransactionDelegate.loadRootTransaction(getSupportFragmentManager(), containerId, toFragment, addToBackStack, allowAnimation);
}
```

emmmmm,果然优秀的框架都是跳来跳去的。可以看到这个方法的实现又又又又交给一个代理去实现具体逻辑。而这个mTransactionDelegate成员变量实际上是TransactionDelegate。这个变量在**onCreate()**时候被初始化。

```java
    public void onCreate(@Nullable Bundle savedInstanceState) {
        mTransactionDelegate = getTransactionDelegate();
        mDebugStackDelegate = new DebugStackDelegate(mActivity);

        mFragmentAnimator = mSupport.onCreateFragmentAnimator();
        mDebugStackDelegate.onCreate(Fragmentation.getDefault().getMode());
    }
```

可以看见onCreate方法内初始化了很多代理。

**3. TransactionDelegate 的loadRootTranslation方法**
按照惯例，先贴个代码：

```java
    void loadRootTransaction(final FragmentManager fm, final int containerId, final ISupportFragment to, final boolean addToBackStack, final boolean allowAnimation) {
        enqueue(fm, new Action(Action.ACTION_LOAD) {
            @Override
            public void run() {
                bindContainerId(containerId, to);

                String toFragmentTag = to.getClass().getName();
                TransactionRecord transactionRecord = to.getSupportDelegate().mTransactionRecord;
                if (transactionRecord != null) {
                    if (transactionRecord.tag != null) {
                        toFragmentTag = transactionRecord.tag;
                    }
                }

                start(fm, null, to, toFragmentTag, !addToBackStack, null, allowAnimation, TYPE_REPLACE);
            }
        });
    }
```

这里实际上是把一些操作放入一个队列。

**3.1 enqueue 方法**
我们先来看下enqueue方法：

```java
    private void enqueue(FragmentManager fm, Action action) {
        if (fm == null) {
            Log.w(TAG, "FragmentManager is null, skip the action!");
            return;
        }
        mActionQueue.enqueue(action);
    }
```

里面的操作也很简单，检查fragmentManager 不为空，然后把action对象入列ActionQueue对象中。
而经查看Action类是个抽象类，里面定义一些操作（pop，load，back等）标识常量，以及在上面看下的run抽象方法。

而ActionQueue类也比较简单，里面是一个队列以及主线程的handler。

**3.1.1 ActionQueue 的 enqueue方法**

主要看下ActionQueue中的入列方法。

```java
    public void enqueue(final Action action) {
        if (isThrottleBACK(action)) return;

        if (action.action == Action.ACTION_LOAD && mQueue.isEmpty()
                && Thread.currentThread() == Looper.getMainLooper().getThread()) {
            action.run();
            return;
        }

        mMainHandler.post(new Runnable() {
            @Override
            public void run() {
                enqueueAction(action);
            }
        });
    }
```

这个方法可以看到，我们提交过来的Action（实际pop，start等对fragment操作）会先检测下action类型和列队中action中的情况，而初始时候这种load的action 是直接执行。

所以load这个动作具体动作还是在loadRootTransaction方法中。里面就进行了两个操作绑定contentId和进行一个start方法。

**3.2 bindContainerId方法**
来看怎么绑定我们要替换的id：

```java
    private void bindContainerId(int containerId, ISupportFragment to) {
        Bundle args = getArguments((Fragment) to);
        args.putInt(FRAGMENTATION_ARG_CONTAINER, containerId);
    }
```

很简单，里面把要替换的view id 通过fragment的 arguments 传递给 目标rootfragment。
**3.3 start方法**
start是个十分重要的方法，之后分析一些其他动作我们也会看见,下面是它的部分源码：

```java
    private void start(FragmentManager fm, final ISupportFragment from, ISupportFragment to, String toFragmentTag,
                       boolean dontAddToBackStack, ArrayList<TransactionRecord.SharedElement> sharedElementList, boolean allowRootFragmentAnim, int type) {
        FragmentTransaction ft = fm.beginTransaction();
        boolean addMode = (type == TYPE_ADD || type == TYPE_ADD_RESULT || type == TYPE_ADD_WITHOUT_HIDE || type == TYPE_ADD_RESULT_WITHOUT_HIDE);
        Fragment fromF = (Fragment) from;
        Fragment toF = (Fragment) to;
        Bundle args = getArguments(toF);
        args.putBoolean(FRAGMENTATION_ARG_REPLACE, !addMode);
        // 这里处理sharedElement 动画
        ...
        
        if (from == null) {
            ft.replace(args.getInt(FRAGMENTATION_ARG_CONTAINER), toF, toFragmentTag);
            if (!addMode) {
            
                ...
                
            }
        } else {
        
            ...
            
        }

        if (!dontAddToBackStack && type != TYPE_REPLACE_DONT_BACK) {
            ft.addToBackStack(toFragmentTag);
        }
        supportCommit(fm, ft);
    }
```

这个方法相比之前的有点长，参数也有点多。但其实里面所做的事情也比较容易能看懂。判断是否是add模式，进行动画transaction，进行add、hide 或者replace 动作，最后执行supportCommit方法进行事务提交。而对于set loot fragment 动作，from Fragment 为空，所以执行的是 `ft.replace(args.getInt(FRAGMENTATION_ARG_CONTAINER), toF, toFragmentTag);`。

loadRootFragment这里分析就结束了。我们正常使用Fragment时候常常开启translation 然后进行replace操作，再提交：

```kotlin
supportFragmentManager.beginTransaction()
                    .replace(R.id.fl_top, getRootFragment(), getRootFragment().javaClass.simpleName)
                    .commit()
```

而框架内实际也是进行这样的操作，只是它把我们经常会在Activity做的事情交给SupportActivityDelegate代理执行，而SupportActivityDelegate又把这个动作交给TranslationDelegate类执行。

## start Fragment

分析完loadRootFragment方法后，我们再来分析下，Fragment中常用的start方法。

在 ISupportFragment 的实现类中，我们可以这样使用启动一个Fragment `start(DetailFragment.newInstance(HomeBean));`。

如果你使用Kotlin，我们还可以仿照Anko 进行优化封装（不使用可无视）：

1. 创建内联扩展方法

```kotlin
inline fun <reified T : AbsSupportFragment> AbsSupportFragment.startFragment(vararg param: Pair<String, Any?>) = this.start(instanceSupFrg<T>(*param)!!)
```

其中AbsSupportFragment是实现ISupportFragment接口的Fragment基类

2. 反射创建Fragment对象

```kotlin
/**
 * 创建fragment
 */
inline fun <reified T : AbsSupportFragment> instanceSupFrg(vararg param: Pair<String, Any?>) =
        try {
            val instance = T::class.java.getConstructor().newInstance()
            if (param.isNotEmpty()){
                instance.arguments = Bundle().filterArguments(param)
            }
            instance
        } catch (e: Exception) {
            LogUtils.e("instanceFragment", e)
            null
        }
```

3. 获取解析传入的参数 arguments

```kotlin
@Suppress("UNCHECKED_CAST")
fun Bundle.filterArguments(params: Array<out Pair<String, Any?>>): Bundle {
    params.forEach {
        val value = it.second
        when (value) {
            null -> this.putSerializable(it.first, null as Serializable?)
            is Int -> this.putInt(it.first, value)
            is Long -> this.putLong(it.first, value)
            is CharSequence -> this.putCharSequence(it.first, value)
            is String -> this.putString(it.first, value)
            is Float -> this.putFloat(it.first, value)
            is Double -> this.putDouble(it.first, value)
            is Char -> this.putChar(it.first, value)
            is Short -> this.putShort(it.first, value)
            is Boolean -> this.putBoolean(it.first, value)
            is Serializable -> this.putSerializable(it.first, value)
            is Bundle -> this.putBundle(it.first, value)
            is Parcelable -> this.putParcelable(it.first, value)
            is Array<*> -> when {
                value.isArrayOf<CharSequence>() -> this.putCharSequenceArray(it.first, value as Array<out CharSequence>?)
                value.isArrayOf<String>() -> this.putStringArray(it.first, value as Array<out String>?)
                value.isArrayOf<Parcelable>() -> putParcelableArray(it.first, value as Array<Parcelable>?)
                else -> throw AnkoException("Intent extra ${it.first} has wrong type ${value.javaClass.name}")
            }
            is Byte -> this.putByte(it.first, value)
            is ByteArray -> this.putByteArray(it.first, value)
            is IntArray -> this.putIntArray(it.first, value)
            is LongArray -> this.putLongArray(it.first, value)
            is FloatArray -> this.putFloatArray(it.first, value)
            is DoubleArray -> this.putDoubleArray(it.first, value)
            is CharArray -> this.putCharArray(it.first, value)
            is ShortArray -> this.putShortArray(it.first, value)
            is BooleanArray -> this.putBooleanArray(it.first, value)
            else -> throw AnkoException("Intent extra ${it.first} has wrong type ${value.javaClass.name}")
        }
    }
    return this
}
```

这部分代码参考Anko 中fillIntentArguments 方法

4. 封装后使用
简单封装后我们可以这样启动Fragment `startFragment<BillingObtainFragment>()`

```kotlin
startFragment<CameraFaceFragment>(
                        CameraFaceFragment.ORDER_BEAN to bean,
                        CameraFaceFragment.CHECK_IN_NUMBER to count,
                        CameraFaceFragment.FROM_ORDER to true
                )
```

好了，下面就正式开始分析start 怎么实现的了。


**1. SupportFragmentDelegate中的start方法**

无论是继承SupportFragment还是实现ISupportFragment 接口，调用Start方法时都是调用代理类（SupportFragmentDelegate）的 start 方法，而里面最终都是运行start 的重载方法：

```java
    public void start(final ISupportFragment toFragment, @ISupportFragment.LaunchMode int launchMode) {
        mTransactionDelegate.dispatchStartTransaction(mFragment.getFragmentManager(), mSupportF, toFragment, 0, launchMode, TransactionDelegate.TYPE_ADD);
    }
```

而 mTransactionDelegate 变量明显也是代理对象，在onAttach方法内进行初始化：

```java
    public void onAttach(Activity activity) {
        if (activity instanceof ISupportActivity) {
            this.mSupport = (ISupportActivity) activity;
            this._mActivity = (FragmentActivity) activity;
            //这里初始化事务代理
            mTransactionDelegate = mSupport.getSupportDelegate().getTransactionDelegate();
        } else {
            throw new RuntimeException(activity.getClass().getSimpleName() + " must impl ISupportActivity!");
        }
    }
```

这里可以看见其实 SupportFragmentDelegate 里面的事务代理和 SupportActivityDelegate 其实是同一个对象。

**2.TransactionDelegate中dispatchStartTransaction方法**

前面接触过Transaction中的loadRootTransaction 方法，现在我们来看下dispatchStartTransaction会简单好多。

```java
    void dispatchStartTransaction(final FragmentManager fm, final ISupportFragment from, final ISupportFragment to, final int requestCode, final int launchMode, final int type) {
        enqueue(fm, new Action(launchMode == ISupportFragment.SINGLETASK ? Action.ACTION_POP_MOCK : Action.ACTION_NORMAL) {
            @Override
            public void run() {
                doDispatchStartTransaction(fm, from, to, requestCode, launchMode, type);
            }
        });
    }
```

同样也是构成Action对象放入队列中，同样我们只需要关心run方法的内容，这里把具体实现逻辑放到
doDispatchStartTransaction方法。

```java
    private void doDispatchStartTransaction(FragmentManager fm, ISupportFragment from, ISupportFragment to, int requestCode, int launchMode, int type) {
        checkNotNull(to, "toFragment == null");
        // 如果start要获取结果
        if ((type == TYPE_ADD_RESULT || type == TYPE_ADD_RESULT_WITHOUT_HIDE) && from != null) {
            // 判断是否attach 到Activity
            if (!((Fragment) from).isAdded()) {
                Log.w(TAG, ((Fragment) from).getClass().getSimpleName() + " has not been attached yet! startForResult() converted to start()");
            } else {
                // 通过Bundle 存储请求码到from 和 to Fragment中
                saveRequestCode(fm, (Fragment) from, (Fragment) to, requestCode);
            }
        }
        // 现在获取顶部Fragment
        from = getTopFragmentForStart(from, fm);
        // 获取之前replace的 containerId
        int containerId = getArguments((Fragment) to).getInt(FRAGMENTATION_ARG_CONTAINER, 0);
        // 处理栈顶Fragment为空和containerId为空的情况
        if (from == null && containerId == 0) {
            Log.e(TAG, "There is no Fragment in the FragmentManager, maybe you need to call loadRootFragment()!");
            return;
        }
        // 当 containerId 为空时候，放入Fragment 的argument中
        if (from != null && containerId == 0) {
            bindContainerId(from.getSupportDelegate().mContainerId, to);
        }

        // process ExtraTransaction
        String toFragmentTag = to.getClass().getName();
        boolean dontAddToBackStack = false;
        ArrayList<TransactionRecord.SharedElement> sharedElementList = null;
        TransactionRecord transactionRecord = to.getSupportDelegate().mTransactionRecord;
        if (transactionRecord != null) {
            if (transactionRecord.tag != null) {
                toFragmentTag = transactionRecord.tag;
            }
            dontAddToBackStack = transactionRecord.dontAddToBackStack;
            if (transactionRecord.sharedElementList != null) {
                sharedElementList = transactionRecord.sharedElementList;
                // Compat SharedElement
                FragmentationMagician.reorderIndices(fm);
            }
        }
        // 处理启动模式
        if (handleLaunchMode(fm, from, to, toFragmentTag, launchMode)) return;
        // 实际启动
        start(fm, from, to, toFragmentTag, dontAddToBackStack, sharedElementList, false, type);
    }
```

这个方法的代码量比之前多，除去处理异常场景，重要逻辑就两个。1. 处理带有类似启动模式的start；2. 调用start 方法实际启动Fragment。

**3.1 TransactionDelegate的handleLaunchMode方法**

这个方法主要从处理带有启动模式的start

```java
    private boolean handleLaunchMode(FragmentManager fm, ISupportFragment topFragment, final ISupportFragment to, String toFragmentTag, int launchMode) {
        if (topFragment == null) return false;
        final ISupportFragment stackToFragment = SupportHelper.findBackStackFragment(to.getClass(), toFragmentTag, fm);
        if (stackToFragment == null) return false;

        if (launchMode == ISupportFragment.SINGLETOP) {
            if (to == topFragment || to.getClass().getName().equals(topFragment.getClass().getName())) {
                handleNewBundle(to, stackToFragment);
                return true;
            }
        } else if (launchMode == ISupportFragment.SINGLETASK) {
            doPopTo(toFragmentTag, false, fm, DEFAULT_POPTO_ANIM);
            mHandler.post(new Runnable() {
                @Override
                public void run() {
                    handleNewBundle(to, stackToFragment);
                }
            });
            return true;
        }

        return false;
    }
```
里面逻辑也是很简单。通过TAG名字找到目标fragment，如果是SINGLETOP，检查栈顶fragment是不是要启动的fragment，如果是执行handleNewBundle方法（里面逻辑是更新argument，执行fragment的onNewBundle方法）。如果是SINGLETASK模式，执行doPopTo方法，把目标fragment上面的都移除掉，然后再执行handleNewBundle方法。

**3.2 TransactionDelegate的start方法**

又回到的了这个方法，之前我们在看 loadRootFragment 时候接触过这个方法，不过很多细节被我们隐藏,不过拆分下去也比较好理解。处理sharedElement动画->处理loadRoot情况->处理正常启动情况->是否添加到回退栈addToBackStack->提交事务。而其中和我们有关的逻辑是：

```java
            if (addMode) {
                ft.add(from.getSupportDelegate().mContainerId, toF, toFragmentTag);
                if (type != TYPE_ADD_WITHOUT_HIDE && type != TYPE_ADD_RESULT_WITHOUT_HIDE) {
                    ft.hide(fromF);
                }
            } else {
                ft.replace(from.getSupportDelegate().mContainerId, toF, toFragmentTag);
            }
```
主要根据不同type进行add、hide、replace操作。

## pop操作

根据前面的经验，实际操作看TransactionDelegate的pop方法：

```java
    void pop(final FragmentManager fm) {
        enqueue(fm, new Action(Action.ACTION_POP, fm) {
            @Override
            public void run() {
                handleAfterSaveInStateTransactionException(fm, "pop()");
                removeTopFragment(fm);
                FragmentationMagician.popBackStackAllowingStateLoss(fm);
            }
        });
    }
```
里面实际移除的方法是removeTopFragment

```java
    private void removeTopFragment(FragmentManager fm) {
        ISupportFragment top = SupportHelper.getBackStackTopFragment(fm);
        if (top != null) {
            fm.beginTransaction()
                    .setTransition(FragmentTransaction.TRANSIT_FRAGMENT_CLOSE)
                    .remove((Fragment) top)
                    .commitAllowingStateLoss(); 
        }
    }
```

可以看见里面实际上进行remove事务操作。

# 总结

经过源码学习发现，里面用了很多代理模式，但经过剖析可以大概分成3个部分：
1. 外观类
比如SupportFragment，ISupportFragment，SupportActivity，ISupportActivity，SupportFragmentDelegate，SupportActivityDelegate 这些外观类直接跟使用者接触。
2. 核心类
主要是实际进行事务操作的TransactionDele代理类，以及ActionQueue，Action这些封装操作的类。里面处理了加载根Fragment、替换根Fragment、add/replace 子Fragment、隐藏/显示Fragment、启动Fragment，以及Fragment出栈，获取某Fragment ，处理转场动画等操作。

3. 帮助类
主要是一些DebugStackDelegate，SupportHelper，异常处理类，动画类等。