> 白菜Java自习室 涵盖核心知识

> [Java工程师的进阶之路 容器篇（一）](https://juejin.im/post/6844904182894297101)<br>
> [Java工程师的进阶之路 容器篇（二）](https://juejin.im/post/6844904183045292039)<br>
> [Java工程师的进阶之路 容器篇（三）](https://juejin.im/post/6844904185377325069)<br>

## Java容器——List详解

### 1. List 接口

List 接口规定了对列表的操作函数和迭代函数，具体接口定义如下：

```
// Collection的API
abstract boolean         add(E object)
abstract boolean         addAll(Collection<? extends E> collection)
abstract void            clear()
abstract boolean         contains(Object object)
abstract boolean         containsAll(Collection<?> collection)
abstract boolean         equals(Object object)
abstract int             hashCode()
abstract boolean         isEmpty()
abstract Iterator<E>     iterator()
abstract boolean         remove(Object object)
abstract boolean         removeAll(Collection<?> collection)
abstract boolean         retainAll(Collection<?> collection)
abstract int             size()
abstract <T> T[]         toArray(T[] array)
abstract Object[]        toArray()
// 相比与Collection，List新增的API：
abstract void                add(int location, E object)
abstract boolean             addAll(int location, Collection<? extends E> collection)
abstract E                   get(int location)
abstract int                 indexOf(Object object)
abstract int                 lastIndexOf(Object object)
abstract ListIterator<E>     listIterator(int location)
abstract ListIterator<E>     listIterator()
abstract E                   remove(int location)
abstract E                   set(int location, E object)
abstract List<E>             subList(int start, int end)
```

其中，subList() 函数采用**视图模式**，所谓视图模式，即我们对 subList() 返回的 List 的修改会被反应到原 List 中（反之亦然）。

> 对视图非结构的更改，都会反应在原始列表中（反之亦然）</br>
> 对视图结构的修改（改变数组大小等会使迭代产生不正确的操作）可能会对程序造成一些不良影响（在当前的实现中，不良影响就是会抛出一个名为 ConcurrentModificationException 的异常）

### 2. AbstractList

AbstractList 类定义如下：
```
public abstract class AbstractList<E> extends AbstractCollection<E> implements List<E>
```
AbstractList 类也实现了大部分的接口方法，包括利用迭代器实现的查找等 indexOf 方法，一个较为核心的理念，是他实现了 add(index, e) 和 remove(e) 方法，只不过实现只有一条语句，就是抛出一个 UnsupportedOperationException 异常，而不是将其定义为 abstract 方法，代码如下：
```
public void add(int index, E element) {
    throw new UnsupportedOperationException();
}
public E remove(int index) {
    throw new UnsupportedOperationException();
}
```
#### 2.1. Iterator 实现

AbstractList 类的实现使用了内部类。

引用了外部 size()、get(int index) 和 remove(int index) 方法。size() 用于获得存储的对象总数，get(i) 用于获得在特定 index 节点的对象并返回，remove(int index) 主要用于元素的删除。

引用了外部变量 modCount。主要用于记录结构修改的次数，所谓的结构修改，即指那些更改列表的大小，或者以一种可能导致迭代过程中产生错误结果的修改。
```
private class Itr implements Iterator<E> {
    //调用 next 函数时需要返回对象的 index
    int cursor = 0;

    // 最后一次返回的对象 index
    int lastRet = -1;

    //所期望的修改的计数
    int expectedModCount = modCount;

    public boolean hasNext() {
        return cursor != size();
    }

    public E next() {
        checkForComodification();
        try {
            int i = cursor;
            E next = get(i);
            lastRet = i;
            cursor = i + 1;
            return next;
        } catch (IndexOutOfBoundsException e) {
            checkForComodification();
            throw new NoSuchElementException();
        }
    }

    public void remove() {
        if (lastRet < 0)
            throw new IllegalStateException();
        checkForComodification();
        try {
            //引用外部函数，为防止重名，加限定词 AbstractList.this
            AbstractList.this.remove(lastRet);
            if (lastRet < cursor)
                cursor--;
            lastRet = -1;
            expectedModCount = modCount;
        } catch (IndexOutOfBoundsException e) {
            throw new ConcurrentModificationException();
        }
    }

    final void checkForComodification() {
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
    }
}
```
实现非常简单，在内部维护两个指针，表明下一个元素 index 和 当前返回元素的 index。调用 next() 时，根据指针返回元素，并对指针进行自加，调用 remove() 时，删除元素，并自减指针。

在 Itr 的基础上，同时还实现了 ListIterator，用于实现前向迭代，和在迭代到特定位置时增加元素。

#### 2.2. for-each 删除元素

当用如下代码删除元素时，会出现 ConcurrentModificationException 异常。
```
    // function 1
    for (String string : list) {
        list.remove(string);
    }
    
    // function 2
    Iterator<String> iterator = list.iterator();
    while (iterator.hasNext()) {
        String string = iterator.next();
        list.remove(string);
    }
```
首先说 ConcurrentModificationException 是什么异常。

> 1. 由检测到并发修改的方法在此类中不允许引发，避免数据的混乱。如：一个线程在另一线程对集合进行迭代时修改集合。</br>
> 2. 某些Iterator实现(包括JRE提供的所有通用集合实现)可能会选择在检测到此行为时引发此异常。这样的迭代器被称为快速失败迭代器(fail-fast)，这样可以避免将来不可确定的问题。</br>
> 3. 如果单个线程发出一系列违反对象约定的方法调用，该对象也可能引发此异常。例如，线程在使用快速失败迭代器迭代集合时直接修改集合，则迭代器将抛出此异常。

Itr 实现了快速迭代器的设计思路：List 的实现中，对 add(int inex, E e) 和 remove(int index) 进行重写，增加 modCount 的自增，然后在 itr 中和 expectedModCount 对比，如果对比不一致，即有修改，则抛出异常。

一个 for-each 循环，会被编译器翻译成迭代器，如果该迭代器基于快速失败理论，那么在 for-each 循环中就会引发 ConcurrentModificationException 异常。所以如果需要在迭代中删除元素，必须显式的使用迭代器，并使用迭代器的删除方式。

标准删除格式如下：
```
public void removeTest(E eParam) {
    Iterator<E> iterator = list.iterator();
    while (iterator.hasNext()) {
        E e = iterator.next();
        if (e.equals(ePaeam)) {
            iterator.remove();
        }
    }
}
```

#### 2.3. subList 实现

在 AbstractList 中，subList 函数主要返回两种类型的 List，两种 List 的实现都使用了 fail-fast 原则，确保在对 sublist 进行操作的时候，不能对原始的 list 列表进行操作，否则抛出 ConcurrentModificationException 异常。

subList() 函数体如下：
```
public List<E> subList(int fromIndex, int toIndex) {
    return (this instanceof RandomAccess ?
        new RandomAccessSubList<>(this, fromIndex, toIndex) :
        new SubList<>(this, fromIndex, toIndex));
}
```

如果当前 List 实现了随机访问( RandomAccess )，则返回一个可以随机访问的 RandomAccessSubList，否则，返回普通 SubList。

SubList类维护了一个指向原列表的指针和相对于原列表的偏移量，对该 subList 的所有操作，都会通过指向原列表的指针和偏移量对原列表进行操作。

实现 fail-fast，使用 modCount 和 l.modCount 记录修改数。，执行 subList 的每一个方法之前，都会检验原列表的修改数（l.modCount）是否和当前记录的一样，如果不一样，则抛出异常。即 subList 创建之后不允许对原列表进行直接修改。

RandomAccessSubList<E> 实现很简单，通过继承 SubList 获得 fail-fast 特性，然后重写 subList 方法，返回一个新的 RandomAccessSubList 对象，而不是 SubList 对象。代码如下：
```
class RandomAccessSubList<E> extends SubList<E> implements RandomAccess {
    RandomAccessSubList(AbstractList<E> list, int fromIndex, int toIndex) {
        super(list, fromIndex, toIndex);
    }

    public List<E> subList(int fromIndex, int toIndex) {
        return new RandomAccessSubList<>(this, fromIndex, toIndex);
    }
}
```

### 3. ArrayList

ArrayList 是 AbstractList 的直接子类，使用数组对元素进行存储，并且实现了 Seriable 接口。

ArrayList 是一个非同步的列表，需要使用包装类对其进行包装，使其在异步环境下保证正确性。

ArrayList 的size、isEmpty、get、set、iterator 和 listIterator 操作的事件复杂度都是常数级别，add 操作平均需要 $O(n)$ 的时间，即插入 n 个数需要 $O(n)$ 的时间，其他的操作都是线性时间。

ArrayList 允许 null 的加入。iterator 和 listIterator 支持 fail-fast，并且支持 RandomAccess。

#### 3.1. MAX_ARRAY_SIZE

在 ArrayList 中，可以存储的最大容量被标记为 MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8，即 $2^{31} -1 - 8$。这时为了保证在不同的平台都可以最数组进行寻址，有些平台实现保留了一些头文件，所以不能寻址到 $2^{31} -1$。在 64 位电脑上，通常可以寻址到 $2^{31} -2$，所以推荐到减 8 。

#### 3.2. ArrayList 扩容

在 ArrayList 中，扩容使用了四个函数 ensureCapacityInternal(int minCapacity)、ensureExplicitCapacity(int minCapacity)、grow(int minCapacity)和 hugeCapacity(int minCapacity) 完成。函数参数的意义都是所需要的最小数组大小。ensureXXX 代表确认是否满足大小要求，如果不满足，则修改大小。

在对元素进行增加的操作时，首先会调用 ensureCapacityInternal(int minCapacity)
```
private void ensureCapacityInternal(int minCapacity) {
    if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
        minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
    }

    ensureExplicitCapacity(minCapacity);
}
```
该函数的主要作用就是当数组是空表时，将期望的大小设置为所需大小和默认大小的最大值，并交由 ensureExplicitCapacity(minCapacity) 处理。
```
private void ensureExplicitCapacity(int minCapacity) {
    modCount++;

    // overflow-conscious code
    if (minCapacity - elementData.length > 0)
        grow(minCapacity);
}
```
调用 ensureExplicitCapacity(minCapacity) 函数时，会将修改标记自加 1，并且判断 所需的最小容量是不是大于数组大小，是的话进行扩容。
```
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
```
ArrayList 的扩容逻辑是判断原始数据增加 1/2 倍后能否满足条件，如果不能满足条件，则使用最小期望的容量作为扩充的大小并验证最小扩充的大小是否超过 MAX_ARRAY_SIZE，如果超过，调用 hughCapacity 处理，最后将 elementData 复制到新长度数组中。

hugeCapacity() 主要是抛出错误或者修改太大的数尝试使用 Integer.MAX_VALUE 分配空间。
```
private static int hugeCapacity(int minCapacity) {
    if (minCapacity < 0) // overflow
        throw new OutOfMemoryError();
    return (minCapacity > MAX_ARRAY_SIZE) ?
        Integer.MAX_VALUE :
        MAX_ARRAY_SIZE;
}
```

#### 3.3. batchRemove

batchRemove 函数主要用在removeAll(Collection<?> c)，中，使用了原地排序的思想。

### 4. LinkedList

LinkedList 由 AbstractSequentialList 继承而来。AbstractSequentialList 抽象类的主要作用就是减少顺序列表实现的工作量。如：所有的顺序列表只有通过迭代器访问或者修改元素，则他实现了get、set 和 remove 等方法，子类只需要实现 iterator 就可以完成顺序列表的实现。

典型的实现如下：
```
public E get(int index) {
    try {
        return listIterator(index).next();
    } catch (NoSuchElementException exc) {
        throw new IndexOutOfBoundsException("Index: "+index);
    }
}

public boolean addAll(int index, Collection<? extends E> c) {
    try {
        boolean modified = false;
        ListIterator<E> e1 = listIterator(index);
        Iterator<? extends E> e2 = c.iterator();
        while (e2.hasNext()) {
            e1.add(e2.next());
            modified = true;
        }
        return modified;
    } catch (NoSuchElementException exc) {
        throw new IndexOutOfBoundsException("Index: "+index);
    }
}

public E remove(int index) {
    try {
        ListIterator<E> e = listIterator(index);
        E outCast = e.next();
        e.remove();
        return outCast;
    } catch (NoSuchElementException exc) {
        throw new IndexOutOfBoundsException("Index: "+index);
    }
}
```

LinkedList 类定义如下：
```
public class LinkedList<E>
    extends AbstractSequentialList<E>
    implements List<E>, Deque<E>, Cloneable, java.io.Serializable
```
LinkedList 实现了 List 接口和 Deque 接口，用于实现双向的操作。

#### 4.1. 存储结构与迭代器

顾名思义，LinkedList 使用链表作为存储单元，单元定义如下：
```
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
```
他维护了一个值和分别指向前后的指针，主要就是使用指针的移动实现访问。
```
private class ListItr implements ListIterator<E> {
    private Node<E> lastReturned;
    private Node<E> next;
    private int nextIndex;
    private int expectedModCount = modCount;

    ListItr(int index) {
        // assert isPositionIndex(index);
        next = (index == size) ? null : node(index);
        nextIndex = index;
    }

    public boolean hasNext() {
        return nextIndex < size;
    }

    public E next() {
        checkForComodification();
        if (!hasNext())
            throw new NoSuchElementException();

        lastReturned = next;
        next = next.next;
        nextIndex++;
        return lastReturned.item;
    }

    public boolean hasPrevious() {
        return nextIndex > 0;
    }

    public E previous() {
        checkForComodification();
        if (!hasPrevious())
            throw new NoSuchElementException();

        lastReturned = next = (next == null) ? last : next.prev;
        nextIndex--;
        return lastReturned.item;
    }

    public int nextIndex() {
        return nextIndex;
    }

    public int previousIndex() {
        return nextIndex - 1;
    }

    public void remove() {
        checkForComodification();
        if (lastReturned == null)
            throw new IllegalStateException();

        Node<E> lastNext = lastReturned.next;
        unlink(lastReturned);
        if (next == lastReturned)
            next = lastNext;
        else
            nextIndex--;
        lastReturned = null;
        expectedModCount++;
    }

    public void set(E e) {
        if (lastReturned == null)
            throw new IllegalStateException();
        checkForComodification();
        lastReturned.item = e;
    }

    public void add(E e) {
        checkForComodification();
        lastReturned = null;
        if (next == null)
            linkLast(e);
        else
            linkBefore(e, next);
        nextIndex++;
        expectedModCount++;
    }

    public void forEachRemaining(Consumer<? super E> action) {
        //...
    }

    final void checkForComodification() {
        //fail-fast check
}
```

### 5. CopyOnWriteArrayList

CopyOnWriteArrayList 是一个线程安全的列表，是 ArrayList 的线程安全版本。他保证的内存一致性模型是：将对象放入 CopyOnWriteArrayList 的线程操作先于访问或者移除元素 (As with other concurrent collections, actions in a thread prior to placing an object into a CopyOnWriteArrayList happen-before actions subsequent to the access or removal of that element from the CopyOnWriteArrayList in another thread.)

类使用 volatile 关键字修饰 Object[]，保证 Object[] 的修改随所有线程可见。用锁保证线程安全，并且对 object[] 的访问只能使用内置的 getArray 方法和 setArray 方法。关键代码如下：
```
public class CopyOnWriteArrayList<E>
    implements List<E>, RandomAccess, Cloneable, java.io.Serializable {
    private static final long serialVersionUID = 8673264195747942595L;

    //锁，保证线程安全
    final transient ReentrantLock lock = new ReentrantLock();

    //使用 volatile 的数据存储结构，保证可见性
    private transient volatile Object[] array;

    //获得 object[] 对象的唯一途径
    final Object[] getArray() {
        return array;
    }

    // 设置 object[] 对象的唯一途径
    final void setArray(Object[] a) {
        array = a;
    }

    // 创建空表
    public CopyOnWriteArrayList() {
        setArray(new Object[0]);
    }

    public CopyOnWriteArrayList(Collection<? extends E> c) {
        Object[] elements;
        if (c.getClass() == CopyOnWriteArrayList.class)
            elements = ((CopyOnWriteArrayList<?>)c).getArray();
        else {
            elements = c.toArray();
            // c.toArray might (incorrectly) not return Object[] (see 6260652)
            if (elements.getClass() != Object[].class)
                elements = Arrays.copyOf(elements, elements.length, Object[].class);
        }
        setArray(elements);
    }

    public CopyOnWriteArrayList(E[] toCopyIn) {
        setArray(Arrays.copyOf(toCopyIn, toCopyIn.length, Object[].class));
    }
    //...
}
```

CopyOnWriteArrayList 是典型的读写分离模式，使用复制-写的思路对写操作保证线程安全，如 add，set 等操作，所有的写操作都会在一个新的副本数组上完成，并最后连接到旧数组上。所以 CopyOnWriteArrayList 在写操作时非常的耗费时间和内存，但是对于读操作没有影响，适合用在读操作多于写操作的地方。

实现原理如下图：
![](https://user-gold-cdn.xitu.io/2020/6/8/17291eac95c3e11f?w=1185&h=321&f=webp&s=17254)

几乎每一个写操作都实现了以上的过程，这样就保证了写的数据只能在写之后才能被读到，但是这样引起的问题是在写得过程中，会读取原 list 的值，只有在彻底写完后才能读取最新值，不能保证实时性。

看 add 方法的实现：
```
public boolean add(E e) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        int len = elements.length;
        Object[] newElements = Arrays.copyOf(elements, len + 1);
        newElements[len] = e;
        setArray(newElements);
        return true;
    } finally {
        lock.unlock();
    }
}
```
可以看到确实是先上锁，然后复制原来的值到新的数组，对新的数组进行操作，然后再将新的数组连接到原 List 中。

> [Java工程师的进阶之路 容器篇（一）](https://juejin.im/post/6844904182894297101)<br>
> [Java工程师的进阶之路 容器篇（二）](https://juejin.im/post/6844904183045292039)<br>
> [Java工程师的进阶之路 容器篇（三）](https://juejin.im/post/6844904185377325069)<br>