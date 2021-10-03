# ArrayDeque

**JDK-11**

Resizable-array implementation of `Deque`.

```
AbstractCollection
        ^
        |
    ArrayDeque
```

# 1. ArrayDeque 属性

```java
public class ArrayDeque<E> extends AbstractCollection<E>
                           implements Deque<E>, Cloneable, Serializable
{

    transient Object[] elements;

    transient int head;

    transient int tail;

}
```

`ArrayDeque` 带有三个属性:

1. `Object[] elements`
    - 存储数据的数组
2. `int head`
    - 头节点的位置
    - `addFirst` 等方法从 `head` 位置插入数据，并且 `head` 总是向左走的 (通过 `dec` 方法)。每次从 `head` 加一个元素，`head` 就会往左走一步 (可能 wrap 一圈，也就是如果 `--head < 0` 则有 `head = elements.length-1`)
3. `int tail`
    - 尾节点的位置
    - `addLast` 等方法从 `tail` 位置插入数据，并且 `tail` 总是向右走的 (通过 `inc` 方法)。每次从 `tail` 加一个元素，`tail` 就会往右走一步 (可能 wrap 一圈，也就是如果 `++tail >= elements.length` 则有 `tail = 0`)

# 2. 构造方式

默认 `ArrayDeque` 的 capacity 为 16. 

```java
public ArrayDeque() {
    elements = new Object[16];
}

public ArrayDeque(int numElements) {
    elements =
        new Object[(numElements < 1) ? 1 :
                    (numElements == Integer.MAX_VALUE) ? Integer.MAX_VALUE :
                    numElements + 1];
}

public ArrayDeque(Collection<? extends E> c) {
    this(c.size());
    copyElements(c);
}
```

# 3. head 头部添加元素

首先，头部添加元素并不代表该元素一定在左边。`offerFirst` 或 `addFirst` 方法都是在 `head` 位置放置元素，`dec` 方法的意思就是 decrement `head` 的数值，如果 `head` 小于 0，这就代表越界了，那么我们把 `head` 设为 `elements.length - 1`。还句话来说就是, `dec` 和 `inc` 方法只是在 decrement/increment 上处理了越界问题而已。初始情况下, `head` 和 `tail` 都为 0 (默认值)，所以，第一次 `dec` 返回的值则是 15 因为 `--head` 越界了， 也就是 `es[15] = e`。如果 `head == tail`，则代表我们需要扩容，后面讲这个。

```java
public boolean offerFirst(E e) {
    addFirst(e);
    return true;
}

public void addFirst(E e) {
    if (e == null)
        throw new NullPointerException();
    final Object[] es = elements;
    es[head = dec(head, es.length)] = e;
    if (head == tail)
        grow(1);
}

static final int dec(int i, int modulus) {
    if (--i < 0) i = modulus - 1;
    return i;
}
```

# 4. tail 尾部添加元素

首先，尾部添加元素并不代表该元素一定在右边。元素是插入到 `tail` 位置的，与 `addFirst` 方法中使用的 `dec` 方法相似，`tail` 会在 `inc` 方法中往右走一步。如果 `head == tail` 则代表需要扩容。

```java
public boolean offerLast(E e) {
    addLast(e);
    return true;
}

public void addLast(E e) {
    if (e == null)
        throw new NullPointerException();
    final Object[] es = elements;
    es[tail] = e;
    if (head == (tail = inc(tail, es.length)))
        grow(1);
}

static final int inc(int i, int modulus) {
    if (++i >= modulus) i = 0;
    return i;
}
```

# 5. 扩容

首先 `grow(int)` 方法中传入的 `int needed` 是指需要多少容量，当插入一个节点时且需要进行扩容，我们则会调用 `grow` 方法并且传入1, 换句话来说我们至少需要扩容到 `oldCapacity + needed` 的大小。然后我们要理解，`jump` 指的是什么意思，`jump` 本质上就是指我们扩容多少, 一般情况下 (扩容后容量小于 `MAX_ARRAY_SIZE`)，我们计算出的新容量会是 `newCapacity = oldCapacity + jump`。

所以，我们可以认为，如果 `oldCapacity < 64`，我们扩容 `oldCapacity + 2`，约等于两倍大小。但如果 `oldCapacity >= 64`，我们则扩容 50%，也就是 `oldCapacity / 2`。

```java
private void grow(int needed) {
    // overflow-conscious code
    final int oldCapacity = elements.length;
    int newCapacity;
    // Double capacity if small; else grow by 50%
    int jump = (oldCapacity < 64) ? (oldCapacity + 2) : (oldCapacity >> 1);
    if (jump < needed
        || (newCapacity = (oldCapacity + jump)) - MAX_ARRAY_SIZE > 0)
        newCapacity = newCapacity(needed, jump);
    final Object[] es = elements = Arrays.copyOf(elements, newCapacity);
    // Exceptionally, here tail == head needs to be disambiguated
    if (tail < head || (tail == head && es[head] != null)) {
        // wrap around; slide first leg forward to end of array
        int newSpace = newCapacity - oldCapacity;
        System.arraycopy(es, head,
                            es, head + newSpace,
                            oldCapacity - head);
        for (int i = head, to = (head += newSpace); i < to; i++)
            es[i] = null;
    }
}

private int newCapacity(int needed, int jump) {
    final int oldCapacity = elements.length, minCapacity;
    if ((minCapacity = oldCapacity + needed) - MAX_ARRAY_SIZE > 0) {
        if (minCapacity < 0)
            throw new IllegalStateException("Sorry, deque too big");
        return Integer.MAX_VALUE;
    }
    if (needed > jump)
        return minCapacity;
    return (oldCapacity + jump - MAX_ARRAY_SIZE < 0)
        ? oldCapacity + jump
        : MAX_ARRAY_SIZE;
}
```

实际扩容也就是进行数组的复制，但是因为我们使用的是双指针，如果我们在 `tail < head` 的情况没有把 `head` 后面使用的元素移到数组的尾部 (新扩容的区域)，那么新增区域并不会被使用。这里的逻辑并不复杂，不管哪种情况，当 `head` 与 `tail` 相遇，这肯定是因为 `dec(head) == tail` 或 `inc(tail) == head`。不管是 `head` 追上 `tail` 还是 `tail` 追上 `head`，`head` 与 `tail` 相遇前的场景基本如下：

```
          <--- head
               |
               v
+------+-------+-----------+
|++++++|       |+++++++++++|
|++++++|       |+++++++++++|
+------+-------+-----------+
       ^
       |
       tail --->
```

记住 `head` 总是通过 `dec` 方法向左移动，而 `tail` 总是通过 `inc` 方法向右移动。扩容以后如下：

```
     <--- head
          |
          v
+---------+----------------+-------------------+
|++++++++++++++++++++++++++|                   |
|++++++++++++++++++++++++++|                   |
+---------+----------------+-------------------+
          ^                          ^
          |                          |
          tail --->                 new space
```

然后我们将 `head` 以及 `head` 后的元素往右移动，移动到新区域内。

```
                         <--- head
                              |
                              v
+---------+-------------------+----------------+
|+++++++++|                   |++++++++++++++++|
|+++++++++|                   |++++++++++++++++|
+---------+-------------------+------+---------+
          ^                          ^
          |                          |
          tail --->                 new space
```

这基本就是以下代码在做的事:

```java
if (tail < head || (tail == head && es[head] != null)) {
    int newSpace = newCapacity - oldCapacity;
    System.arraycopy(es, head,
                        es, head + newSpace,
                        oldCapacity - head);
    for (int i = head, to = (head += newSpace); i < to; i++)
        es[i] = null;
}
```

# 6. head 头部移除元素  

移除元素并不会收缩数组大小，所以逻辑非常直接。在 `head` 位置插入元素的时候，我们会使用 `dec` 方法将 `head` 向左移动，如果是删除的话，则使用 `inc` 将 `head` 向右移动即可。只要确保在移动前将 `elements[head]` 设为 `null` 即可。

```java
public E removeFirst() {
    E e = pollFirst();
    if (e == null)
        throw new NoSuchElementException();
    return e;
}

public E pollFirst() {
    final Object[] es;
    final int h;
    E e = elementAt(es = elements, h = head);
    if (e != null) {
        es[h] = null;
        head = inc(h, es.length);
    }
    return e;
}

static final <E> E elementAt(Object[] es, int i) {
    return (E) es[i];
}
```

# 7. tail 尾部移除元素

在 `tail` 移除元素跟在 `head` 移除元素没有区别。也就是在移动 `tail` 前将 `elements[tail]` 设为 `null` 然后使用 `dec` 将 `tail` 向左移动即可。当所有元素都被移除以后，`head` 会与 `tail` 位置重合。 

```java
public E removeLast() {
    E e = pollLast();
    if (e == null)
        throw new NoSuchElementException();
    return e;
}

public E pollLast() {
    final Object[] es;
    final int t;
    E e = elementAt(es = elements, t = dec(tail, es.length));
    if (e != null)
        es[tail = t] = null;
    return e;
}
```
