前面我们撸了List的源码，接着本想撸个Set的源码，无论是HashSet还是TreeSet里面都有个内置的Map。所以先来撸个Map的源码。

- hashMap 构造做了什么事
- hashMap 使用什么存储元素
- hashMap 怎么添加元素
- hashMap 如何删除元素
- hashMap 如何实现扩容

# hashMap构造 #

HashMap有4个构造:

1. 空构造HashMap() 
	
	public HashMap() {
        this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
    }

里面进行loadFactor成员变量初始化,而loadFactor是什么呢。在后面的阅读中会发现这个成员变量很常见，它是负载系数，默认为0.75。在扩容时候我们将知道这个变量的作用。

2. 传入初始容量构造

	public HashMap(int initialCapacity) {
        this(initialCapacity, DEFAULT_LOAD_FACTOR);
    }

这里主要把初始容量以及默认的负载系数传入第三个构造

3. 初始容量，负载系数构造

		public HashMap(int initialCapacity, float loadFactor) {
        	if (initialCapacity < 0)
            	throw new IllegalArgumentException("Illegal initial capacity: " +initialCapacity);
        	if (initialCapacity > MAXIMUM_CAPACITY)
            	initialCapacity = MAXIMUM_CAPACITY;
        	if (loadFactor <= 0 || Float.isNaN(loadFactor))
            	throw new IllegalArgumentException("Illegal load factor: " +loadFactor);
        	this.loadFactor = loadFactor;
        	this.threshold = tableSizeFor(initialCapacity);
    	}
 里面逻辑也比较简单，先对传入的初始容量以及加载系数进行判断限制，然后是成员变量赋值。但这里我们看到一个新的成员变量，threshold，阀值，而它是通过tableSizeFor方法确定的，来我们撸下这个方法

		static final int tableSizeFor(int cap) {
        	int n = cap - 1;
        	n |= n >>> 1;
        	n |= n >>> 2;
        	n |= n >>> 4;
        	n |= n >>> 8;
        	n |= n >>> 16;
        	return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
    	}

看到这就有点蛋疼，这里出现了两个位运算符号>>>（无符号右移）、|=(或等)。我们举几个数来看看这段代码的逻辑，当传入0时候，n=-1，转成2进制有如下步骤
- 绝对值转为二进制 0000 0001
- 反码	1111 1110
- 反码+1  ，1111 1111

所以以 n>>>1 为 0111 1111 1111 1111 1111 1111 1111 1111（负数有符号右移1补位，无符号0补位），|=后 高位又为1，最终为1111 1111 ... 1111 （十进制为-1），经过后面几个步骤最终还是-1 ，最终返回1。而对于正数，这几个步骤的作用是把最高位后面都置为了1，然后加1返回。比如传入的20，n就为19，二进制 0001 0011，经过`n |= n >>> 1;`得到0001 1011，然后经过`n |= n >>> 2`得到0001 1111 ，然后之后的步骤 n 都为 31，最终返回32，所以可以传入容量x，2^m-1<=x<=2^m,最终返回 2^m 值。当然他有个最大值为MAXIMUM_CAPACITY（1<<30=2^30）


4. 传入其他map

这个构造传入了其他map

		public HashMap(Map<? extends K, ? extends V> m) {
        	this.loadFactor = DEFAULT_LOAD_FACTOR;
        	putMapEntries(m, false);
    	}

而这里也是初始化负载系数，然后调用putMapEntries方法，而putMapEntries就是存放方法。放到后面解析。

# HashMap 存储内部类 #

HashMap中实际存储元素内部类为Node节点，与LinkedList中的节点不同的是，HashMap中的节点是单链表，只记录下个节点next

	static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        V value;
        Node<K,V> next;

        Node(int hash, K key, V value, Node<K,V> next) {
            this.hash = hash;
            this.key = key;
            this.value = value;
            this.next = next;
        }

        public final K getKey()        { return key; }
        public final V getValue()      { return value; }
        public final String toString() { return key + "=" + value; }
		
		....
        ....
    }

除了上面介绍的next属性，它还有3个比较重要的属性，hash值，存储的key值，以及value值。


而在HashMap中还有个重要的属性`transient Node<K,V>[] table;`，一个由上述节点Node组成的数组。所以HashMap中的存储结构其实是数组+链表，既有数组的查找优势，又有链表的增删优势。


# HashMap添加元素 #

我们一般添加元素都是调用put方法，来撸下put方法内做了什么事情。

	public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }

发现里面先调用hash方法计算key的hash值再调用putVal方法存放元素。

- hash 方法计算hash值

	
		static final int hash(Object key) {
        	int h;
        	return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    	}
	判断key是否为空,空的hash值为0,不为空为原hash值的高位+(高位异或地位)

- putVal存放元素

		final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        else {
            Node<K,V> e; K k;
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        return null;
    }


这段代码有点长我们分开分析。
首先有几个局部变量赋值的动作，tab 为成员变量table（节点数组），n为table的长度，p为数组中某个位置（传入hash值计算后）的Node。

1. 当原来数组为空时候进行resize 调整大小。

resize里面代码比较多，主要逻辑分原数组（table）空和不为空时候的情况。当为空时候，初始化大小为16的数组。而初始阀值（threshold）为初始容量乘以载荷因子（12）。而原数组不为空，newCap为原来的两倍。

