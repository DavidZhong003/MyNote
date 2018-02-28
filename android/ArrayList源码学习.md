# 简述 #

ArrayList部分UML类图为

![](http://img.blog.csdn.net/20160602191805971)


ArrayList继承AbstractList,实现List、RandomAccess、Cloneable、Serializable接口

- List 接口，有增删改查，遍历等功能。
- RandomAccess，可随机访问。
- Cloneable接口，可被克隆。


# 源码阅读 #

## 构造方法 ##

1. 空构造

	public ArrayList() {
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
    }
这里进行的是存储数据元素的数组`transient Object[] elementData;`（transient关键字是该变量不参与序列化）初始化为空数组（`private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};`）

2. 初始化容量构造（` public ArrayList(int initialCapacity)`）

这个构造器也是初始化elementData数组，如果初始容量小于0则抛异常。

    public ArrayList(int initialCapacity) {
    	if (initialCapacity > 0) {
    		this.elementData = new Object[initialCapacity];
    	} else if (initialCapacity == 0) {
    		this.elementData = EMPTY_ELEMENTDATA;
    	} else {
    		throw new IllegalArgumentException("Illegal Capacity: "+
       			initialCapacity);
    		}
    	}

3.初始化数据构造

这个构造传入一个Collection对象，然后把他转为Object数组赋值给elementData

	public ArrayList(Collection<? extends E> c) {
        elementData = c.toArray();
        if ((size = elementData.length) != 0) {
            // c.toArray might (incorrectly) not return Object[] (see 6260652)
            if (elementData.getClass() != Object[].class)
                elementData = Arrays.copyOf(elementData, size, Object[].class);
        } else {
            // replace with empty array.
            this.elementData = EMPTY_ELEMENTDATA;
        }
    }

## 添加元素 ##

添加元素一般调用add（）方法，这里以最简单add(E e)方法为例。

	public boolean add(E e) {
        ensureCapacityInternal(size + 1);  // Increments modCount!!
        elementData[size++] = e;
        return true;
    }


里面方法很简单，里面调用ensureCapacityInternal（）方法后，把添加的元素放入数组。那ensureCapacityInternal这个方法是干嘛的呢，从字面翻译是确保内置容量（）

	private void ensureCapacityInternal(int minCapacity) {
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
            minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
        }

        ensureExplicitCapacity(minCapacity);
    }

该方法传入最小容量int值，判断数据是否为空，空的话，比较默认容量DEFAULT_CAPACITY（10）和传入的最小容量大小，取更大的数传入方法ensureExplicitCapacity（minCapacity）

	private void ensureExplicitCapacity(int minCapacity) {
        modCount++;

        // overflow-conscious code
        if (minCapacity - elementData.length > 0)
            grow(minCapacity);
    }

这里出现一个新的成员变量modCount（`protected transient int modCount = 0;`），该变量在父类中被定义，初始化为0，用来记录list结构修改次数。

从上面看出，先确保传入的最小容量大于elementData数组容量的长度，然后把最小容量minCapacity传入方法grow（）

	private void grow(int minCapacity) {
        // overflow-conscious code
        int oldCapacity = elementData.length;
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        // minCapacity is usually close to size, so this is a win:
        elementData = Arrays.copyOf(elementData, newCapacity);
    }

这里先获取旧的容量长度，再增长1.5倍（`oldCapacity + (oldCapacity >> 1)`），如果小于传入的最小容量，则新得容量等于传入的最小容量，如果新的容量大于最大数组大小MAX_ARRAY_SIZE，新容量会等于Integer.MAX_VALUE或者MAX_ARRAY_SIZE。最后把elementData和新容量传入Arrays.copyOf方法。而里面调用重构copyOf方法，

	public static <T,U> T[] copyOf(U[] original, int newLength, Class<? extends T[]> newType) {
        @SuppressWarnings("unchecked")
        T[] copy = ((Object)newType == (Object)Object[].class)
            ? (T[]) new Object[newLength]
            : (T[]) Array.newInstance(newType.getComponentType(), newLength);
        System.arraycopy(original, 0, copy, 0,
                         Math.min(original.length, newLength));
        return copy;
    }
	
可以看出主要还是调用系统的数组拷贝方法进行扩容。

	public static native void arraycopy(Object src,  int  srcPos,
                                        Object dest, int destPos,
                                        int length);


这个方法的参数分别是原数组，原数组起始位置，目标数组，目标数组起始位置，拷贝的长度。


## 删除 ##

删除主要调用remove方法，查看源码发现该方法有很多重构，根据索引删除元素，根据对象删除元素。这里吐槽下这bug，当存储的是Integer数据时候，如果想remove 1 对象会调用remove 索引方法。

来看常用的根据索引删除元素方法

	public E remove(int index) {
        if (index >= size)
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));

        modCount++;
        E oldValue = (E) elementData[index];

        int numMoved = size - index - 1;
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        elementData[--size] = null; // clear to let GC do its work

        return oldValue;
    }

如果索引大于数组长度，抛出个异常。然后记录该位置的元素，然后通过上面熟悉的System.arraycopy方法进行数组移动，再把最后一个元素置为null，接着返回删除的元素

## 改 ##

更改某个位置元素，通常调用set函数

	public E set(int index, E element) {
        if (index >= size)
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));

        E oldValue = (E) elementData[index];
        elementData[index] = element;
        return oldValue;
    }

同样这个方法也比较简单，先判断索引是否越界，然后记录旧元素，赋值新元素，返回旧元素。

## 查 ##

查看是否有某元素，通常调用contains方法，该方法体比较简单`return indexOf(o) >= 0;`
我们撸下indexOf源码

	public int indexOf(Object o) {
        if (o == null) {
            for (int i = 0; i < size; i++)
                if (elementData[i]==null)
                    return i;
        } else {
            for (int i = 0; i < size; i++)
                if (o.equals(elementData[i]))
                    return i;
        }
        return -1;
    }
里面先判断查找元素是否是null（Arryalist可以存储null），然后for循环查找元素，返回其索引，若没有则返回-1.

## Iterator ##

ArrayList还实现了Iterator迭代器接口，主要有内部的Itr类完成这功能，来撸下迭代器源码。

Itr为ArrayList的内部类,其中重要的方法是hasNext(),next(),remove()。

1. hasNext()

	public boolean hasNext() {
            return cursor < limit;
        }
	
这个方法比较简单，只需明白cursor 和limit两个变量意义。其中cursor是下个返回元素的索引，默认为0。limit是ArrayList的大小。

2. next()

next方法弹出下个元素

	public E next() {
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
            int i = cursor;
            if (i >= limit)
                throw new NoSuchElementException();
            Object[] elementData = ArrayList.this.elementData;
            if (i >= elementData.length)
                throw new ConcurrentModificationException();
            cursor = i + 1;
            return (E) elementData[lastRet = i];
        }

先进行个判断之前出现的modCount（用于记录ArrayList结构修改的次数），而expectedModCount初始值为modeCount,所以在迭代器循环中如果调用Arraylist的add或remove方法都会抛出异常。后面的逻辑也比较简单，确保索引在范围内，索引改变， return elementData对应元素

3. remove()
	
remove方法也比较简单，实际是调用ArrayList的remove方法，然后更改索引等变量。


# 总结 #

ArrayList内部使用数组存储元素，每次添加元素时候会确保内部容量，容量不够时候进行扩容操作（默认容量为10），扩容系数为1.5（原容量+原容量<<2），内部调用系统拷贝数组方法进行数据转移。所以ArrayList查找快增删慢。