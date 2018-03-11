# 简述 #

LinkList继承AbstractSequentialList类（AbstractSequentialList父类和ArrayList相同也是AbstractList），实现List、Deque（队列）、Cloneable、Serializable接口。

# 源码阅读 #

## 构造方法 ##

有两个构造方法，一个空构造，一个是传入一个Collection对象

	public LinkedList(Collection<? extends E> c) {
        this();
        addAll(c);
    }

其中this（）第一个空构造，里面空实现。主要调用addAll方法,这个方法我们后续再分析。

### Node ###

Node为LinkedList的内部类，源码也比较简单，里面3个成员变量分别记录元素，上个节点，下个节点

	private static class Node<E> {
        E item;
        Node<E> next;
        Node<E> prev;

        Node(Node<E> prev, E element, Node<E> next) {
            this.item = element;
            this.next = next;
            this.prev = prev;
        }
    }



## 增 ##

1. 增加元素有很多方法，先分析最简单的add方法

其源码是

	public boolean add(E e) {
        linkLast(e);
        return true;
    }

里面调用linkLast方法，从字面理解是吧新增的元素链接到最后，来看下它的源码

	void linkLast(E e) {
        final Node<E> l = last;
        final Node<E> newNode = new Node<>(l, e, null);
        last = newNode;
        if (l == null)
            first = newNode;
        else
            l.next = newNode;
        size++;
        modCount++;
    }
从上面代码可以看出，把新增的元素包装成Node（其头节点指向链表原尾节点，尾节点为null）。而这里出现first，last两个成员变量。分别记录这个链表（LinkedList的头节点和尾节点）。当尾节点为空时候，赋值给头节点，否则添加到尾节点。然后修改链表长度size以及链表结构修改次数modCount

而当该LinkedList走空构造添加第一个元素时候，last（`transient Node<E> last;`）初始为null，所以链表的头结点为null

2. add(int index, E element)

根据索引添加元素

	public void add(int index, E element) {
        checkPositionIndex(index);

        if (index == size)
            linkLast(element);
        else
            linkBefore(element, node(index));
    }

先确保传入的索引在0，size之间，再判断如果索引是链表长度，直接添加到最后，否则添加到该索引之前。先来分析node（index）方法，该方法应该是根据索引获取节点元素。

	Node<E> node(int index) {
        // assert isElementIndex(index);

        if (index < (size >> 1)) {
            Node<E> x = first;
            for (int i = 0; i < index; i++)
                x = x.next;
            return x;
        } else {
            Node<E> x = last;
            for (int i = size - 1; i > index; i--)
                x = x.prev;
            return x;
        }
    }


先判断传入的索引更靠近头还是尾（size>>1），然后选择从头或者从尾遍历元素，不断取下个（next）或者上个（prev），只到取到索引对应元素并返回

3. addAll()

addAll有两个重构方法，我们直接看传入添加索引和集合的方法。

public boolean addAll(int index, Collection<? extends E> c) {
        checkPositionIndex(index);

        Object[] a = c.toArray();
        int numNew = a.length;
        if (numNew == 0)
            return false;

        Node<E> pred, succ;
        if (index == size) {
            succ = null;
            pred = last;
        } else {
            succ = node(index);
            pred = succ.prev;
        }

        for (Object o : a) {
            @SuppressWarnings("unchecked") E e = (E) o;
            Node<E> newNode = new Node<>(pred, e, null);
            if (pred == null)
                first = newNode;
            else
                pred.next = newNode;
            pred = newNode;
        }

        if (succ == null) {
            last = pred;
        } else {
            pred.next = succ;
            succ.prev = pred;
        }

        size += numNew;
        modCount++;
        return true;
    }

代码比较长，又看见我们熟悉的方法checkPositionIndex（index）检查索引是否在范围内。然后把传入的集合转为数组，确保数组内有元素（`if (numNew == 0)return false;`）。
接下就是实际添加集合操作。先创建两个节点pred，succ。然后初始化pred（它作用相当于指针）。主要是先判断索引是不是最尾端，如果是指向last也就是链表的尾节点，如果不是通过node（indext）找到对应的插入位置的节点并赋值给succ，然后它的上个节点赋值给pred。
接下就是实际添加元素进链表操作。。。。。。
遍历数组元素，创建节点对象。指针（pred）下移动（`pred.next = newNode; pred = newNode`）,然后如果succ不为空（当插入位置为最尾端时候succ会为空，此时更改成员变量last），进行闭合链表操作。然后进行更新链表大小以及modCount。


## 删 ##

LinkedList删除元素的方法有很多,pop(),remove()等。这里只分析几个。

1. removeFirst（）

这个方法都被pop（），和remove（）方法内部调用。

	public E removeFirst() {
        final Node<E> f = first;
        if (f == null)
            throw new NoSuchElementException();
        return unlinkFirst(f);
    }

内部先取出链表头节点，确保它不为空。然后调用unlinkFirst方法

	private E unlinkFirst(Node<E> f) {
        // assert f == first && f != null;
        final E element = f.item;
        final Node<E> next = f.next;
        f.item = null;
        f.next = null; // help GC
        first = next;
        if (next == null)
            last = null;
        else
            next.prev = null;
        size--;
        modCount++;
        return element;
    }

这里主要是先取出包装在节点中的元素，以及取出其下个节点。然后手动把节点包装元素以及下个节点置为null。把链表的头节点指针（first）指向刚取出的next节点。最后修改size，modCount参数，返回删除的元素。


2. remove（int index）

里面主要调用unlink（node（index））方法，node（index）之前分析过主要是获取该索引的节点，所以我们重点放到方法unlink上。这个方法主要是分离节点。具体分析见注释。

	E unlink(Node<E> x) {
        // assert x != null;
		//取出存储的元素
        final E element = x.item;
		//该节点下个节点
        final Node<E> next = x.next;
		//取出该节点的上个节点
        final Node<E> prev = x.prev;
		//确保改节点不是链表头节点(头节点的上节点为null)
        if (prev == null) {
			//头节点下移
            first = next;
        } else {
			//上节点的下指针指向下节点(断开上节点与传入节点关联)
            prev.next = next;
            x.prev = null;
        }
		//确保节点不是链表尾节店
        if (next == null) {
            last = prev;
        } else {
			//断开下节点与传入节点关联
            next.prev = prev;
            x.next = null;
        }
		//置为空,方便GC
        x.item = null;
		//更改链表大小
        size--;
		//记录结构被修改
        modCount++;
        return element;
    }

# 改 #

改主要调用set方法。源码逻辑也比较简单，取出索引位置节点，更改节点记录的元素值为传入的新值。

	public E set(int index, E element) {
        checkElementIndex(index);
        Node<E> x = node(index);
        E oldVal = x.item;
        x.item = element;
        return oldVal;
    }

# 查 #

主要是方法contains方法，内部是调用indexOf方法。

	public int indexOf(Object o) {
        int index = 0;
        if (o == null) {
            for (Node<E> x = first; x != null; x = x.next) {
                if (x.item == null)
                    return index;
                index++;
            }
        } else {
            for (Node<E> x = first; x != null; x = x.next) {
                if (o.equals(x.item))
                    return index;
                index++;
            }
        }
        return -1;
    }

无论元素是不是null，里面基本是从链表头节点遍历下去，判断节点中存储的元素是否是传入的元素，然后返回索引。

