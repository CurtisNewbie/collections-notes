# PriorityQueue

**JDK-11**

Classic min or max heap implementation. 核心在于 `sink` 和 `swim`，在代码中对应 `siftDown` 和 `siftUp` 方法。

```
AbstractCollection
       ^
       |
  AbstractQueue
       ^
       |
  PriorityQueue
```

## 1. PriorityQueue 属性

```java
public class PriorityQueue<E> extends AbstractQueue<E>
    implements java.io.Serializable {

    private static final int DEFAULT_INITIAL_CAPACITY = 11;

    transient Object[] queue; 

    int size;

    private final Comparator<? super E> comparator;

    transient int modCount;     
}
```

- `int DEFAULT_INITIAL_CAPACITY` 
    - 默认初始容量
- `Object[] queue` 
    - heap 数组
    - `queue[0]` 是 min/max element
    - `queue[k]` 的 children 是 `queue[2 * k + 1]` 和 `queue[2 * (k + 1)]`
    - `queue[k]` 的 parent 是 `queue[(k - 1) / 2]`
- `int size` 
    - 记录元素数量
- `Comparator<? super E> comparator`
    - nullable `Comparator`，用于进行 priority 比较
- `int modCount`
    - 用于检测并发修改问题

# 2. 构造

默认初始的大小为 `11`，我们可以选择提供 `Comparator`。`size` 默认为 0，`queue[0]` 是会被使用。

```java
public PriorityQueue() {
    this(DEFAULT_INITIAL_CAPACITY, null);
}

public PriorityQueue(int initialCapacity) {
    this(initialCapacity, null);
}

public PriorityQueue(Comparator<? super E> comparator) {
    this(DEFAULT_INITIAL_CAPACITY, comparator);
}

public PriorityQueue(int initialCapacity,
                        Comparator<? super E> comparator) {
    // Note: This restriction of at least one is not actually needed,
    // but continues for 1.5 compatibility
    if (initialCapacity < 1)
        throw new IllegalArgumentException();
    this.queue = new Object[initialCapacity];
    this.comparator = comparator;
}
```

# 3. 添加元素

添加元素时，元素是先添加再更新 `size` 的，所以我们直接进行比较，判断是否需要扩容，如果 `size >= queue.length` 则进行扩容。然后我们调用 `siftUp` 方法，将新节点插入到 `size` 位置，然后从下往上 (从右往左) 进行 `swim` 操作。本质上就是比较 `parent`，如果当前节点比 `parent` 大，那么就跟 `parent` 换位置，继续往上走，直到当前节点小于 `parent`。

```java
public boolean offer(E e) {
    if (e == null)
        throw new NullPointerException();
    modCount++;
    int i = size;
    if (i >= queue.length)
        grow(i + 1);
    siftUp(i, e);
    size = i + 1;
    return true;
}

private void siftUp(int k, E x) {
    if (comparator != null)
        siftUpUsingComparator(k, x, queue, comparator);
    else
        siftUpComparable(k, x, queue);
}

private static <T> void siftUpComparable(int k, T x, Object[] es) {
    Comparable<? super T> key = (Comparable<? super T>) x;
    while (k > 0) {
        int parent = (k - 1) >>> 1;
        Object e = es[parent];
        if (key.compareTo((T) e) >= 0)
            break;
        es[k] = e;
        k = parent;
    }
    es[k] = key;
}
```

# 4. 移除元素

元素总是从 `queue[0]` 上移除的，移除以后，我们拿最底下/最后一个元素 (该元素拥有最大的数值)，放到 `queue[0]` 的位置，然后进行 `sink` 下沉操作。

```java
public E poll() {
    final Object[] es;
    final E result;

    if ((result = (E) ((es = queue)[0])) != null) {
        modCount++;
        final int n;
        final E x = (E) es[(n = --size)];
        es[n] = null;
        if (n > 0) {
            final Comparator<? super E> cmp;
            if ((cmp = comparator) == null)
                siftDownComparable(0, x, es, n);
            else
                siftDownUsingComparator(0, x, es, n, cmp);
        }
    }
    return result;
}

private static <T> void siftDownComparable(int k, T x, Object[] es, int n) {
    Comparable<? super T> key = (Comparable<? super T>)x;
    int half = n >>> 1;           // loop while a non-leaf
    while (k < half) {
        // 1)
        int child = (k << 1) + 1; // assume left child is least
        Object c = es[child];
        int right = child + 1;

        // 2)
        if (right < n &&
            ((Comparable<? super T>) c).compareTo((T) es[right]) > 0)
            c = es[child = right];

        // 3)
        if (key.compareTo((T) c) <= 0)
            break;
        
        // 4)
        es[k] = c;
        k = child;
    }
    // 3)
    es[k] = key;
}
```

`siftDown` 本质上就是 `parent` 与比自己更小的 children 换位置，直到自己比 children 都要小:

1. 查出左子节点 (`2 * k + 1`) 和右子节点 (`2 * (k + 1)`)
2. 找出左节点和右节点中最小的值 `c`，要留意，`child` 一开始指向左节点。当右节点存在而且右节点 < `c` 时，`c = es[right]` 
3. 如果当前节点小于 `c` 也就是 children 中的最小值，这代表我们找到了正确的位置，因为当前节点比 children 都要小，我们设 `es[k]` 为我们新插入的值
4. 否则，我们将当前节点值与最小值 `c` 换位置，而我们当前的位置也移到了 `child`，继续 sink

# 5. 扩容

扩容的逻辑基本如下：

1. 如果 `queue.length < 64`，扩容成 `oldCapacity + oldCapacity + 2`，基本为两倍大小
2. 如果 `queue.length >= 64`，扩容成 `oldCapacity + oldCapacity / 2`，基本为 1.5 倍大小
3. 扩容实际上就是数组复制

```java
private void grow(int minCapacity) {
    int oldCapacity = queue.length;
    // Double size if small; else grow by 50%
    int newCapacity = oldCapacity + ((oldCapacity < 64) ?
                                        (oldCapacity + 2) :
                                        (oldCapacity >> 1));

    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    queue = Arrays.copyOf(queue, newCapacity);
}

private static int hugeCapacity(int minCapacity) {
    if (minCapacity < 0) // overflow
        throw new OutOfMemoryError();
    return (minCapacity > MAX_ARRAY_SIZE) ?
        Integer.MAX_VALUE :
        MAX_ARRAY_SIZE;
}
```