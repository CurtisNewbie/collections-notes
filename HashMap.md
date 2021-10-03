# HashMap

**JDK-11**

**忽略部分实现细节**

```
AbstractMap
     ^
     |
  HashMap
```

# 1. HashMap 属性

```java
public class HashMap<K,V> extends AbstractMap<K,V>
    implements Map<K,V>, Cloneable, Serializable {

    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16
    static final int MAXIMUM_CAPACITY = 1 << 30;
    static final float DEFAULT_LOAD_FACTOR = 0.75f;
    static final int TREEIFY_THRESHOLD = 8;
    static final int UNTREEIFY_THRESHOLD = 6;
    static final int MIN_TREEIFY_CAPACITY = 64;

    transient Node<K,V>[] table;
    transient Set<Map.Entry<K,V>> entrySet;
    transient int size;
    transient int modCount;
    int threshold;
    final float loadFactor;
```

- `int DEFAULT_INITIAL_CAPACITY = 1 << 4`
    - 默认容量大小, 2^4,  也就是 16
    - 容量总是 2^*, 因为当 N 是 2 的 M 次方时, `hash % N` 等于 `hash & N - 1`
- `int MAXIMUM_CAPACITY = 1 << 30`
    - 最大容量
- `float DEFAULT_LOAD_FACTOR = 0.75f`
    - 默认扩容因子 75%
- `int TREEIFY_THRESHOLD = 8` 
    - 当 `table` 数组大小超过 `MIN_TREEIFY_CAPACITY` 并且 `table[k]` 内链表大小大于 `TREEIFY_THRESHOLD` 时, 将链表转为红黑树 (也叫 `treeify`)
- `int UNTREEIFY_THRESHOLD = 6` 
    - 当 `table[k]` 内链表大小小于 `UNTREEIFY_THRESHOLD` 时, 将红黑树结构转为链表 (也叫 `untreeify`)
- `int MIN_TREEIFY_CAPACITY = 64`
    - 对 `table[k]` 进行 treeify 所需要 `table` 满足的数组最小容量
- `Node<K,V>[] table`
    - bin 数组, 每一个 bin 装链表或红黑树结构
- `Set<Map.Entry<K,V>> entrySet` 
    - 缓存 `entrySet()`
- `int size`
    - 元素数量
- `int modCount`
    - 用于检测多线程并发修改
- `int threshold`
    - 下一个扩容的大小, 具体值为 `oldCapacity * loadFactor`, 如果是两倍扩容, 则是 `(table.length * 2) * loadFactory`
- `float loadFactor`
    - 扩容因子

# 2. 构造

如果没有设置 `initialCapacity` 和 `loadFactor`, 默认使用 `DEFAULT_INITIAL_CAPACITY` 和 `DEFAULT_LOAD_FACTOR`。这里可以看出来, 数组并不是在构造器中创建的, 实际上是在第一次插入 key-value 时, 在 `resize()` 方法中创建的。`tableSizeFor(int)` 方法用于返回 2 的 N 次方的大小, 这样确保我们能使用 `hash & N - 1` 的技巧。

```java
public HashMap(int initialCapacity, float loadFactor) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal initial capacity: " +
                                            initialCapacity);
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal load factor: " +
                                            loadFactor);
    this.loadFactor = loadFactor;
    this.threshold = tableSizeFor(initialCapacity);
}

public HashMap(int initialCapacity) {
    this(initialCapacity, DEFAULT_LOAD_FACTOR);
}

public HashMap() {
    this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
}

static final int tableSizeFor(int cap) {
    int n = -1 >>> Integer.numberOfLeadingZeros(cap - 1);
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```

# 3. 插入数据  

首先计算 `key` 对应的 `hash`, 然后调用 `putVal` 方法插入数据。

```java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}

static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}

final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                boolean evict) {
    
    Node<K,V>[] tab; Node<K,V> p; int n, i;

    // 1)
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;

    // 2)
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        Node<K,V> e; K k;

        // 3)
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;

        // 4)
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);

        // 5)
        else {
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);

                    // 6)
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
        // 7)
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    // 8)
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}
```

`putVal` 方法:

1. 首先检查 `table` 数组是否已经初始化, 如果没有, 使用 `resize` 方法进行初始化 
2. 利用 `n-1 & hash` 定位 `hash` 应该放入的桶 `p`, 如果该桶为空, 初始化新节点
3. 如果该桶/节点 `p` 不为 `null`, 且第一个节点就命中, 我们记录节点 `e` 为我们找到的, 相同 `hash` 和 相同 `key` 的节点, 因为 `key` 命中, 我们需要覆盖旧值
4. 如果没有命中, 我们检查是否是 `TreeNode`, 如果是, 则代表该桶已经是红黑树结构了, 则在红黑树内插入或覆盖该 entry
5. 如果既不是红黑树, 也没有命中, 那么该节点是链表的头节点, 我们逐个链表进行比较, 如果命中, 我们记录 `e` 为找到节点, 否则, 我们创建新节点并且插入到链表最后
6. 如果加入节点后, 链表长度大于 `TREEIFY_THRESHOLD` (也就是 8), 我们 `treeify` 该桶, 将链表转成红黑树
7. 如果 `e` 不为空, 也就是前面的步骤已经定位到相同 `hash` 和相同 `key` (`equals` 方法) 的节点, 我们覆盖旧值
8. 如果插入新节点后, `size` 大于需要扩容的 `threshold`, 调用方法 `resize` 进行扩容

# 4. 扩容

`HashMap` 扩容主要是在 `resize` 方法中。首先我们关注 `newCap = oldCap << 1` 这段代码, 我们可以确认, 扩容后容量是两倍大小。同时, `threshold` 也是通过 `newThr = oldThr << 1` 扩容为两倍大小。如果是新创建的 `HashMap`, 我们从构造器中可以了解到, 我们并没有创建 `table` 数组, 所以 `table == null` 也就是 `oldCap == 0` 和 `oldThr == 0`。

那么对应这种情况, 我们则会初始化到默认大小 `newCap = DEFAULT_INITIAL_CAPACITY`, 同时 `newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY)`。接下来的代码就是实际的扩容, 而扩容本质上就是重新 `hash` 然后放置到新的数组中。不过精妙的地方可关注链表-重新 hash, 会将链表中元素放置到 `j` 或 `j + oldCap` 上, 这主要是由于两倍扩容的原因。

```java
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

# 5. 拿出数据

搜索的功能与部分插入数据的代码相似, 同样也是先计算 `hash`, 然后通过 `(tab.length - 1) & hash` 计算对应的桶的位置, 然后检查第-一个节点是否命中 (`hash` 相同并且 `key` 也相同)。如果未命中, 则在红黑树/链表中继续匹配查询。

```java
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

# 6. 删除数据

与插入数据, 查询数据相似, 代码实现的复杂度主要在红黑树和链表的节点插入, 查询, 删除中, 而不是 `HashMap` 本身。删除操作第一步是定位到桶, 然后定位到节点, 最后才是删除。而删除, 对于链表则是 `prev.next = node.next`, 但对于红黑树则涉及平衡和旋转。

```java
public V remove(Object key) {
    Node<K,V> e;
    return (e = removeNode(hash(key), key, null, false, true)) == null ?
        null : e.value;
}

final Node<K,V> removeNode(int hash, Object key, Object value,
                            boolean matchValue, boolean movable) {
    Node<K,V>[] tab; Node<K,V> p; int n, index;
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (p = tab[index = (n - 1) & hash]) != null) {
        Node<K,V> node = null, e; K k; V v;
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            node = p;
        else if ((e = p.next) != null) {
            if (p instanceof TreeNode)
                node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
            else {
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key ||
                            (key != null && key.equals(k)))) {
                        node = e;
                        break;
                    }
                    p = e;
                } while ((e = e.next) != null);
            }
        }
        if (node != null && (!matchValue || (v = node.value) == value ||
                                (value != null && value.equals(v)))) {
            if (node instanceof TreeNode)
                ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
            else if (node == p)
                tab[index] = node.next;
            else
                p.next = node.next;
            ++modCount;
            --size;
            afterNodeRemoval(node);
            return node;
        }
    }
    return null;
}
```
    