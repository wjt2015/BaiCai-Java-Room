> 白菜Java自习室 涵盖核心知识

> [Java工程师的进阶之路 容器篇（一）](https://juejin.im/post/6844904182894297101)<br>
> [Java工程师的进阶之路 容器篇（二）](https://juejin.im/post/6844904183045292039)<br>
> [Java工程师的进阶之路 容器篇（三）](https://juejin.im/post/6844904185377325069)<br>

## Java容器——Map详解

### 1. AbstractMap

#### 1.1. EntrySet

AbstractMap 实现了 Map 的基础框架，在这个框架中，最重要的类是 EntrySet 结构。Map 实现不支持迭代器，EntrySet 将键值对包装成一个 Set，这样就可以对 Set 做迭代，实现对 Map 的遍历。

举例来说，containsValue(Object value) 函数在内部使用 entrySet 的迭代器遍历整个 Map，如果找到就返回 true，否则返回 false。
```
public boolean containsValue(Object value) {
    Iterator<Entry<K,V>> i = entrySet().iterator();
    if (value==null) {
        while (i.hasNext()) {
            Entry<K,V> e = i.next();
            if (e.getValue()==null)
                return true;
        }
    } else {
        while (i.hasNext()) {
            Entry<K,V> e = i.next();
            if (value.equals(e.getValue()))
                return true;
        }
    }
    return false;
}
```

#### 1.2. 键视图和值视图

和 List 的 subList 类似，Map 支持返回键视图和值视图。因为 Map 的值要求唯一，而 value 不做要求，所以 Map 的键视图采用 Set 存储，值视图采用 Collection 存储，两个视图如下所示：
```
transient Set<K>        keySet;
transient Collection<V> values;
```

**懒加载模式**

所谓的懒加载模式，即在创建类的时候不创建视图，而是在使用视图的时候，如调用 keySet() 函数时才创建视图。

如下，当 keySet 不为空时，直接返回 keySet；如果为空，才创建 keySet。
```
public Set<K> keySet() {
    Set<K> ks = keySet;
    if (ks == null) {
        ks = new AbstractSet<K>() {
            //...
        };
        keySet = ks;
    }
    return ks;
}
```

**受限制的视图**

和 AbstractList 的 subList 所不同的是，AbstractMap 提供的键视图和值视图是受限制的视图。我们可以在 subList 中删除元素，增加元素，取决于他的原 List 可以支持哪些操作，但是 AbstractMap 的键视图和值视图只支持部分操作。

KeySet 的实现重写了 AbstractSet，但是在 AbstractSet 中，所有的操作都被设置成了抛出 OperationNotSupportException，所以只有其重写的函数和该 Map 的实现所绑定。

下面是 AbstractMap 的 keySet 实现。
```
new AbstractSet<K>() {
    public Iterator<K> iterator() {
        return new Iterator<K>() {
            private Iterator<Entry<K,V>> i = entrySet().iterator();

            public boolean hasNext() {
                return i.hasNext();
            }

            public K next() {
                return i.next().getKey();
            }

            public void remove() {
                i.remove();
            }
        };
    }

    public int size() {
        return AbstractMap.this.size();
    }

    public boolean isEmpty() {
        return AbstractMap.this.isEmpty();
    }

    public void clear() {
        AbstractMap.this.clear();
    }

    public boolean contains(Object k) {
        return AbstractMap.this.containsKey(k);
    }
};
```

### 2. HashMap

HashMap 是平常用的最多的一个 Map 实现，我们列出其几大特性：
```
1. 基于 Hash table（哈希表）的实现</br>
2. 提供所有 map 接口，允许 null 的键和 null 的值</br>
3. 非线程安全</br>
4. 不保证读取键值对的顺序，不仅仅包括键值对 put 和 get 顺序不同，也包括随时间变化，迭代的顺序也可能不同。</br>
5. 对于基本的操作（get 和 put），可以在常数时间上完成。</br>
```

#### 2.1. HashMap 实现原理

散列表 的作用相当于索引，通过散列表可以快速的定位元素的位置。散列表依据如何解决冲突可以划分为多种散列表，在 HashMap 的实现中，采用单独链表法解决冲突。

首先看 Hash table 的原理，简要的概括， hash table 的思想就是分区。

一个简化版的 hash map 如下图所示：
![](https://user-gold-cdn.xitu.io/2020/7/15/1734f1fdfd1e5953?w=500&h=302&f=png&s=74151)

对于需要保存的每一个 entry，求出其 hash 值，并与 hash table 的大小做取模运算（保证每一个 hash 值都可以映射到 hash table 中），然后，将取模运算之后的 hash 值对应的 entry 连接到 hash table 对应的项中，如果已经有元素，则将其连接到已有元素的后面。

hashMap 采用 Node 类作为其 entry 节点，Node 类如下所示：
```
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

    public final int hashCode() {
        return Objects.hashCode(key) ^ Objects.hashCode(value);
    }

    public final V setValue(V newValue) {
        V oldValue = value;
        value = newValue;
        return oldValue;
    }

    public final boolean equals(Object o) {
        if (o == this)
            return true;
        if (o instanceof Map.Entry) {
            Map.Entry<?,?> e = (Map.Entry<?,?>)o;
            if (Objects.equals(key, e.getKey()) &&
                Objects.equals(value, e.getValue()))
                return true;
        }
        return false;
    }
}
```

Node 实际上是一个链表节点，这样才能实现 hash table 的链表引用。

在 HashMap 的实现中，采用 Node 数组作为哈希表结构。
结构如下：
```
transient Node<K,V>[] table;
```
我们来看 HashMap 具体怎么实现存储。

**put 过程**

根据散列表原理，对每一个加入散列表的元素都要求他的哈希值找到对应的存储位置。在 hashMap 中，我们使用 key 查找值，所以在散列表中，只要能根据 key 的 hash 值快速找到 Node 节点，就可以定位 value 值，所以 hashMap 使用 key 的 hash 值作为 hash table 的索引。

以下是 put 函数的部分源代码：
```
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}

final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
    
    // step 1: resize if necessary
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
        
    // step 2: if no entry in current tab, create a new node
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
        
    else {
    
        Node<K,V> e; K k;
        // step 3: check if first entry key equals key
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        else {
            // step 4: this loop state search entry key equals key in s specific hash or create a new entry if no equals entry found
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    break;
                }
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        // step 5: modifify if exist
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            return oldValue;
        }
    }
    ++modCount;
    if (++size > threshold)
        resize();
    return null;
}
```

最主要的函数是 putVal 函数，他把 key 的哈希值，key 值和 value 值最为参数，总共分为五步，如下所示：

**step 1**：如果 table 为空或者 table 长度为 0，重新配置 table</br>
**step 2**：利用 (n - 1) & hash 找到 entry 所在的链表头。如果链表头为空，直接创建新节点。</br>
用 n - 1 和 hash 做与操作，可以将结果限制在 0 到 n - 1 中，优点是速度快，缺点是相当于只有低位参加了 hash 的过程，导致碰撞几率增大。</br>
**step 3**：检查链表头的 key 是否等于所期望的 key。如果一致，记录当前 node
需要满足两个条件 p.hash == hash 和 (k = p.key) == key || (key != null && key.equals(k))
第一个条件指的是同一个键的 hash 结果应该维持不变，所以先检查 hash 结果，如果 hash 结果不一样，则键肯定不一样（一致性）
第二个条件是指满足所期望的 key 和链表头的 key 为同一对象（同一内存）或者 equal 函数相同</br>
**step 4**：遍历链表，如果满足当前遍历的节点和期望的 key 相同，记录当前 node，break；如果链表中没有满足要求的 node，新建节点在链表最后。</br>
**step 5**：对于 step 3 和 step 4，如果存在满足 key 条件的节点，表明在原来的链表中有记录，根据 onlyIfAbsent 参数决定是否更新。</br>

**get 过程**

get 过程比较简单，根据 key 值遍历 Map，如果有元素，则返回值，如果没有，则返回空，代码如下：
```
public V get(Object key) {
    Node<K,V> e;
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}

final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) {
        if (first.hash == hash && // always check first node
               ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        if ((e = first.next) != null) {
            if (first instanceof TreeNode)
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
            do {
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);
        }
    }
    return null;
}
```

**hash 函数**

因为在 get 方法时使用了 (n - 1) & hash，而不是取模运算，相当与只有低位参加了运算，所以碰撞几率相当的高，为了减少碰撞几率，hashMap 使用了一个支持方法 hash ，将 key 值 hash 的高位和低位做与或运算，使得高位也参加到 hash 的计算中，减少碰撞几率。
```
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

#### 2.2. resize 实现

每当哈希表的使用率大于 $loadFactory * capicity$ 时，会自动扩大哈希表，标准是每次扩大两倍。

扩大两倍可以很好的解决元素重新排列的问题。我们知道 hashMap 使用 $ (n - 1) \& hash$ 作为哈希和表的映射，每一个元素需要调整的位置只能是当前位置或者是当前位置加上原来容量之后的位置，高位决定了是当前位置还是两倍之后的位置。

假设当前容量大小为 $16$($0x10000$)，某一元素的哈希值为 $44$($0x101100$)，不考虑内置的 hash 函数，直接将 15 和 44 做与操作，那么
会得到 $0x1100$，即在原来应该放在 12 号位。容量扩大两倍，实际上是对 16 向左移动了 1 位，得到 32，当和 44 做与操作时，实际上前 4 位不会变动，只有第五位可能有区别，在这个例子中，做与操作仍然是放在 12 号位，当然第五位可能是 1，变成当前位置加上原来容量之后的位置。这样就不用重新计算每一个的位置，而只需要计算高位不同的 hash 值并将存放位置加上当前容量的偏移。

> 最大 table 长度为 $1 << 30$

```
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    if (oldCap > 0) {
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; // double threshold
    }
    else if (oldThr > 0) // initial capacity was placed in threshold
        newCap = oldThr;
    else {               // zero initial threshold signifies using defaults
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;
    @SuppressWarnings({"rawtypes","unchecked"})
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    if (oldTab != null) {
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;
                else if (e instanceof TreeNode)
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else { // preserve order
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        next = e.next;
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    if (hiTail != null) {
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;
}
```

#### 2.3. 迭代器构建

HashMap 的迭代器都基于抽象类 HashIterator，都是快速失败迭代器。

HashIterator 基于深度遍历的思想，首先遍历一个链表结构，然后遍历下一个链表结构。
```
abstract class HashIterator {
    Node<K,V> next;        // next entry to return
    Node<K,V> current;     // current entry
    int expectedModCount;  // for fast-fail
    int index;             // current slot

    HashIterator() {
        expectedModCount = modCount;
        Node<K,V>[] t = table;
        current = next = null;
        index = 0;
        if (t != null && size > 0) { // advance to first entry
        // 将 next 指向第一个存在的（不为空）的链表
            do {} while (index < t.length && (next = t[index++]) == null);
        }
    }

    public final boolean hasNext() {
        return next != null;
    }

    final Node<K,V> nextNode() {
        Node<K,V>[] t;
        Node<K,V> e = next;
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
        if (e == null)
            throw new NoSuchElementException();
        if ((next = (current = e).next) == null && (t = table) != null) {
            do {} while (index < t.length && (next = t[index++]) == null);
        }
        return e;
    }

    public final void remove() {
        Node<K,V> p = current;
        if (p == null)
            throw new IllegalStateException();
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
        current = null;
        K key = p.key;
        removeNode(hash(key), key, null, false, false);
        expectedModCount = modCount;
    }
}
```
在 HashIterator 的基础上，hashMap 实现了自己的两个迭代器，如下：
```
final class KeyIterator extends HashIterator
    implements Iterator<K> {
    public final K next() { return nextNode().key; }
}

final class ValueIterator extends HashIterator
    implements Iterator<V> {
    public final V next() { return nextNode().value; }
}

final class EntryIterator extends HashIterator
    implements Iterator<Map.Entry<K,V>> {
    public final Map.Entry<K,V> next() { return nextNode(); }
}
```

#### 2.4. Java 1.8 性能改进

HashMap 最重要的三个参数， threshold 、 loadFactor 和 capacity。

capcity 指的是 table 的大小。

loadFactory 是一个比例，指 table 的填充比例，即当 table 中的元素填充个数大于 $ capacity * loadFactor$ 时（数组中有 $ capacity * loadFactor$ 个项不为空），则扩充节点。

不同于低版本的 HashMap 实现， 1.8 增加了 threshold 指的是当单链表的长度大于 threshold 时，将单链表重新组合成红黑树形式的存储结构，增加读取效率。详细原理可参见 treeMap。

### 3. treeMap

TreeMap 是基于红黑树实现的一个 map，有几大基本特性。
```
1. 基于红黑树实现
2. 提供所有 map 接口。
3. 非线程安全
4. 迭代时的顺序按照键值排列，即存储顺序是有限的（在这个基础上，同时实现了 
NavigableMap 接口，用于查找某个 key 之前的所有元素等操作）
5. 对于 put 操作，时间复杂度是 $O(logn)$，对于增加和删除操作，时间复杂度不能保证
6. 支持 SortedMap 接口，如 firstKey()，lastKey()，取得最大最小的key，
或sub(fromKey, toKey), tailMap(fromKey) 剪取Map的某一段
```

#### 3.1. TreeMap 实现原理

**2-3 查找树**

利用树进行查找时，希望树尽量的平衡，这样才能够保证在每一次的查找保证 $O(logn)$ 的复杂度。在动态插入的过程种要维持平衡二叉树的代价太高，所以使用一种新型的平衡树 - 2-3 查找树。

对于一个二叉查找树，他的每一个节点有一个值和两条连接，左连接指向的二叉查找树的值都小于该节点，右连接指向的二叉查找树的值都大于该节点，对于一个整数类型的数组 int[] a = new int[] {1,2,3,4,5,6,7}，他所构成的平衡二叉查找树如下所示：
![平衡二叉查找树](https://user-gold-cdn.xitu.io/2020/6/10/1729d12aeeca36d1?w=154&h=117&f=webp&s=1730)

现引入 2-3 查找树，定义如下：
```
1. 为一棵空树或由以下节点组成
2. 2-节点，含有一个值和两条连接，左连接指向的 2-3 树中的值都小于该节点，
右连接指向的 2-3 树的值都大于该节点（类似于查找二叉树）
3. 3-节点，含有两个值和三条连接，左连接指向的 2-3 树中的值都小于该节点，
中连接指向的 2-3 树的值位于该节点的两个值之间，右连接指向的 2-3 树的值都大于该节点
```
对于一个字符数组 char[] chars = new char[] {A,C,H,L,P,S,X,E,J,R,M}，他的平衡 2-3 树如下所示：
![平衡2-3树](https://user-gold-cdn.xitu.io/2020/6/10/1729d19820b29b56?w=288&h=156&f=webp&s=2470)

**查找**

2-3 树查找过程和二叉树相似。

**添加**

在一个只有根节点且是 2-节点的树上添加元素。为了保证平衡，我们需要将该节点替换成一个 3-节点，如下所示：
![根-2节点添加](https://user-gold-cdn.xitu.io/2020/6/10/1729d1e6fe9c5405?w=179&h=47&f=webp&s=1134)

在一个只有根节点且是 3-节点的树上添加元素。为了保证平衡，我们需要将该节点做局部变化，操作如下：首先将该节点临时增加一个值变成 4-节点，然后对四节点进行拆分，变成 3 个 2-节点，如下所示：
![根-3节点添加](https://user-gold-cdn.xitu.io/2020/6/10/1729d1f6b744ecbd?w=274&h=206&f=webp&s=4168)

在一个父节点且是 2-节点，该节点是3-节点的树上添加元素。为了保证平衡，我们需要将该节点做局部变化，操作如下：首先将该节点临时增加一个值变成 4-节点，然后对四节点进行拆分，变成 3 个 2-节点，最后将一个 2-节点 和 父节点合并，如下所示：
![2-父 3-子](https://user-gold-cdn.xitu.io/2020/6/10/1729d21112125e1e?w=532&h=269&f=webp&s=7146)

在一个父节点是 3-节点，该节点是3-节点的树上添加元素。为了保证平衡，我们需要将该节点做局部变化，操作如下：首先将该节点临时增加一个值变成 4-节点，然后对四节点进行拆分，变成 3 个 2-节点，最后将一个 2-节点 和 父节点合并然后递归的对父节点进行操作，直到根节点或者父节点是 2-节点停止，如下所示：
![3-父 3-子](https://user-gold-cdn.xitu.io/2020/6/10/1729d22f5eb6962c?w=906&h=427&f=webp&s=17468)

**删除**

由此可见， 2-3 树是由下向上生长的，但是删除操作需要对树进行从上和从下两方面的判断，相对来说，删除非常费时。


#### 3.2. 红黑树

红黑树是一种 2-3 平衡树的实现。不用去定义特殊的新的数据结构，只需要一些附加信息，就可以实现 2-3 树的构建。

在红黑树种，利用黑连接表示 2-3 树的普通节点， 红连接将两个 2-节点连接够成一个三节点。

一个 2-3 树可以化成一个等效的红黑树，如下图所示：
![红黑树等效2-3树](https://user-gold-cdn.xitu.io/2020/6/10/1729d26b5ce26756?w=797&h=242&f=webp&s=8618)

```
1. 红连接均为左连接
2. 没有任何一个节点同时和两条连接相连
3. 该树是黑色完美平衡的，即任意空连接到根节点的路径上的黑连接数量相同。
```

我们定义节点上存在 color 属性，代表的是指向该节点的连接是什么颜色。

红黑树基本的操作是旋转，在一些实现中，某些操作可能会出现红色右连接或者两条连续的红连接，我们定义左旋转是将一条红色的右连接旋转得到一条左连接。右连接相反，如下图所示：
![左旋转](https://user-gold-cdn.xitu.io/2020/6/10/1729d2b4572cdc36?w=619&h=238&f=webp&s=8710)
左旋转的伪代码如下所示：
```
Node rotateLeft(Node h) {
    Node x = h.right;
    h.right = x.left;
    x.left = h;
    x.color = h.color;
    h.color = red;
}
```
只需更改他的 color 属性为红，并将他原来自身的 color 属性赋值给右连接节点就行。

同理，右旋转示意图如下：
![右旋转](https://user-gold-cdn.xitu.io/2020/6/10/1729d2f1172c0886?w=623&h=231&f=webp&s=8618)
右旋转的伪代码如下所示：
```
NOde rotateRight(Node h) {
    node x= h.left;
    h.left = x.right;
    x.right = h;
    x.color = h.color;
    h.color = red;
}
```
对于每一个插入，都插入一个红连接。

向单个 2-节点中插入新键。如果新键小于老键，则增加一个红色左连接，否则增加一个红色右连接并进行左旋转，两种情况都能产生一个等效的3-连接，如下所示：
![单个2-节点中插入新键](https://user-gold-cdn.xitu.io/2020/6/10/1729d30925b1eb0d?w=450&h=215&f=webp&s=5020)

向树底部的 2-节点中插入新键。总是增加一个新的红色连接，如果他的父节点是 2-节点，那么按照如上两种方式调整节点就行。

向一棵双键树（3-节点）中插入新建，分为了三种子情况，第一种情况是新键最大，第二种情况是新键在中间，第三种情况是新键最小。

如下如所示：
![3-节点插入新建](https://user-gold-cdn.xitu.io/2020/6/10/1729d36380fee45d?w=994&h=397&f=webp&s=19644)

对于一个节点和两个红色连接直接相联，这种情况等效于一个 4-节点，当将这两个红色连接变黑时，需要将父节点由黑变红，因为这样的变换会在父节点产生一个 3-节点，理由如下：
![红黑颜色变换](https://user-gold-cdn.xitu.io/2020/6/10/1729d3735a2645f1?w=349&h=103&f=webp&s=3066)
每产生一个红色连接都会向上传递直到根节点。

**具体实现**

如下代码展示了红黑树在 Map 中的存储节点。
```
static final class Entry<K,V> implements Map.Entry<K,V> {
    K key;
    V value;
    Entry<K,V> left;
    Entry<K,V> right;
    Entry<K,V> parent;
    boolean color = BLACK;

    Entry(K key, V value, Entry<K,V> parent) {
        this.key = key;
        this.value = value;
        this.parent = parent;
    }

    public K getKey() {
        return key;
    }

    public V getValue() {
        return value;
    }

    public V setValue(V value) {
        V oldValue = this.value;
        this.value = value;
        return oldValue;
    }

    public boolean equals(Object o) {
        if (!(o instanceof Map.Entry))
            return false;
        Map.Entry<?,?> e = (Map.Entry<?,?>)o;

        return valEquals(key,e.getKey()) && valEquals(value,e.getValue());
    }

    public int hashCode() {
        int keyHash = (key==null ? 0 : key.hashCode());
        int valueHash = (value==null ? 0 : value.hashCode());
        return keyHash ^ valueHash;
    }

    public String toString() {
        return key + "=" + value;
    }
}
```
以 boolean color 存储指向该节点的连接的颜色。

每当增加一个元素后，调用 fixAfterInsect 函数对红黑树进行修正，如下所示：
```
private void fixAfterInsertion(Entry<K,V> x) {
    x.color = RED;

    while (x != null && x != root && x.parent.color == RED) {
        if (parentOf(x) == leftOf(parentOf(parentOf(x)))) {
            Entry<K,V> y = rightOf(parentOf(parentOf(x)));
            if (colorOf(y) == RED) {
                setColor(parentOf(x), BLACK);
                setColor(y, BLACK);
                setColor(parentOf(parentOf(x)), RED);
                x = parentOf(parentOf(x));
            } else {
                if (x == rightOf(parentOf(x))) {
                    x = parentOf(x);
                    rotateLeft(x);
                }
                setColor(parentOf(x), BLACK);
                setColor(parentOf(parentOf(x)), RED);
                rotateRight(parentOf(parentOf(x)));
            }
        } else {
            Entry<K,V> y = leftOf(parentOf(parentOf(x)));
            if (colorOf(y) == RED) {
                setColor(parentOf(x), BLACK);
                setColor(y, BLACK);
                setColor(parentOf(parentOf(x)), RED);
                x = parentOf(parentOf(x));
            } else {
                if (x == leftOf(parentOf(x))) {
                    x = parentOf(x);
                    rotateRight(x);
                }
                setColor(parentOf(x), BLACK);
                setColor(parentOf(parentOf(x)), RED);
                rotateLeft(parentOf(parentOf(x)));
            }
        }
    }
    root.color = BLACK;
}
```

### 4. ConcurrentHashMap
```
1. 基于 hash map 实现
2. 提供所有 map 接口
3. 不允许以 null 作为键或者值
4. 线程安全，检索操作不需要锁定整个表
5. 检索操作通常不会阻塞，所以有可能和修改操作重叠
6. 迭代时的顺序按照键值排列，即存储顺序是有限的（在这个基础上，
同时实现了 NavigableMap 接口，用于查找某个 key 之前的所有元素等操作）
```

#### 4.1. 实现原理

在 JDK 1.7 及其之前的版本中，ConcurrentHashMap 使用 Segement 数据结构作为上锁的最小单元，每一个 Segment 容纳了一个 table，结构如图所示：
![](https://user-gold-cdn.xitu.io/2020/6/10/1729d3f8176fbfc1?w=288&h=304&f=webp&s=3826)
每当进行 get 操作时，定位到对应的 segment，上锁，并执行 put 操作。

**hash 操作**

计算 hash 时，同样使用了内置的 hash 函数对 hash 进行再次求解。不过相比于 HashMap，ConcurrentHashMap 使用了single-word Wang/Jenkins hash 算法的变种。代码如下：
```
private static int hash(int h) {
    // Spread bits to regularize both segment and index locations,
    // using variant of single-word Wang/Jenkins hash.
    h += (h <<  15) ^ 0xffffcd7d;
    h ^= (h >>> 10);
    h += (h <<   3);
    h ^= (h >>>  6);
    h += (h <<   2) + (h << 14);
    return h ^ (h >>> 16);
}
```
> **Wang/Jenkins Hash 算法关键特性</br>**
> 雪崩性（更改输入参数的任何一位，就将引起输出有一半以上的位发生变化）</br>
> 可逆性（input ==> hash ==> inverse_hash ==> input）

因此，使用 Wang/Jenkins Hash 更加能够获得冲突更小的 hash。

**存储**

在每一个 Segment 中，使用 HashEntry 作为存储结构，HashEntry 的定义如下：
```
static final class HashEntry<K,V> {
        final int hash;
        final K key;
        volatile V value;
        volatile HashEntry<K,V> next;

        HashEntry(int hash, K key, V value, HashEntry<K,V> next) {
            this.hash = hash;
            this.key = key;
            this.value = value;
            this.next = next;
        }

        //保证set对所有线程可见（volatile 语义）
        final void setNext(HashEntry<K,V> n) {
            UNSAFE.putOrderedObject(this, nextOffset, n);
        }

        static final sun.misc.Unsafe UNSAFE;
        static final long nextOffset;
        static {
            try {
                UNSAFE = sun.misc.Unsafe.getUnsafe();
                Class k = HashEntry.class;
                nextOffset = UNSAFE.objectFieldOffset
                    (k.getDeclaredField("next"));
            } catch (Exception e) {
                throw new Error(e);
            }
        }
    }
```

其中，UNSAFE 为 Java 直接访问内存的函数，objectFieldOffset 为获得某一变量的偏移量， putOrderedObject(this, nextOffset, n) 是将 n 放在 this 偏移量 nextOffset 的位置，并且是一个具有 volatile 语义的修改，对所有的线程可见。

Segment 直接继承 ReentrantLock，可以简化锁或者一些单独的构造器，使得其可以单独的当成一个锁。结构如下:
```
static final class Segment<K,V> extends ReentrantLock implements Serializable {
     
        private static final long serialVersionUID = 2249069246763182397L;

    
        static final int MAX_SCAN_RETRIES =
            Runtime.getRuntime().availableProcessors() > 1 ? 64 : 1;

        //真实的存储结构
        transient volatile HashEntry<K,V>[] table;

        //对一个 segment 所有元素计数
        transient int count;

        transient int modCount;

        //当 table 中包含的 HashEntry 元素的个数超过本变量值时，触发 table 的再散列
        transient int threshold;

        final float loadFactor;
        
        //... method

       
}
```
整个类 维持了一个 Segment 数组，和必要的信息，如下：
```
public class ConcurrentHashMap<K, V> extends AbstractMap<K, V>  
        implements ConcurrentMap<K, V>, Serializable {  
    /** 
     * segments 的掩码值
     * key 的散列码的高位用来选择具体的 segment  
     */  
    final int segmentMask;  

    /** 
     * segment 外偏移量，和segment 维持多少个 table 有关
     */  
    final int segmentShift;  

    /** 
     * 由 Segment 对象组成的数组，每个都是一个特别的Hash Table
     */  
    final Segment<K,V>[] segments; 
    
    // 根据 hash 找到对应 segment
    private Segment<K,V> segmentForHash(int h) {
        // 重点在 (h >>> segmentShift) & segmentMask 函数
        long u = (((h >>> segmentShift) & segmentMask) << SSHIFT) + SBASE;
        // 在 volatile 的环境下读取 segments 的第 u 个元素
        return (Segment<K,V>) UNSAFE.getObjectVolatile(segments, u);
    }
    
 }
```

**读写**

对于put操作，如果Key对应的数组元素为null，则通过CAS操作将其设置为当前值。如果Key对应的数组元素（也即链表表头或者树的根元素）不为null，则对该元素使用synchronized关键字申请锁，然后进行操作。如果该put操作使得当前链表长度超过一定阈值，则将该链表转换为树，从而提高寻址效率。

对于读操作，由于数组被volatile关键字修饰，因此不用担心数组的可见性问题。同时每个元素是一个Node实例（Java 7中每个元素是一个HashEntry），它的Key值和hash值都由final修饰，不可变更，无须关心它们被修改后的可见性问题。而其Value及对下一个元素的引用由volatile修饰，可见性也有保障。
```
static class Node<K,V> implements Map.Entry<K,V> {
  final int hash;
  final K key;
  volatile V val;
  volatile Node<K,V> next;
}
```

对于Key对应的数组元素的可见性，由Unsafe的getObjectVolatile方法保证。

put、remove和get操作只需要关心一个Segment，而size操作需要遍历所有的Segment才能算出整个Map的大小。一个简单的方案是，先锁住所有Segment，计算完后再解锁。但这样做，在做size操作时，不仅无法对Map进行写操作，同时也无法进行读操作，不利于对Map的并行操作。
```
static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
  return (Node<K,V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);
}
```

**size操作**

put方法和remove方法都会通过addCount方法维护Map的size。size方法通过sumCount获取由addCount方法维护的Map的size。

> [Java工程师的进阶之路 容器篇（一）](https://juejin.im/post/6844904182894297101)<br>
> [Java工程师的进阶之路 容器篇（二）](https://juejin.im/post/6844904183045292039)<br>
> [Java工程师的进阶之路 容器篇（三）](https://juejin.im/post/6844904185377325069)<br>