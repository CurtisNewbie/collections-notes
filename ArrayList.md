# ArrayList Notes

**JDK-11**

Resizable array-based implementation.

```
AbstractCollection
        ^
        |
   AbstractList
        ^
        |
    ArrayList
```

# 1. ArrayList 属性 

```java
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable
{

    private static final int DEFAULT_CAPACITY = 10;

    private static final Object[] EMPTY_ELEMENTDATA = {};

    private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};

    transient Object[] elementData; 

    private int size;

}
```

以上是 `ArrayList` 中较重要的参数

- `int DEFAULT_CAPACITY = 10` 
    - 是没有设置 capacity 的情况下，第一次扩容使用的默认大小。当我们使用默认构造器创建 `ArrayList` 的时候，我们直接使用 `DEFAULTCAPACITY_EMPTY_ELEMENTDATA` 作为 `elementData`。当我们第一次使用 `add` 方法加入 element 的时候，`elementData` 是大小为 0 的空数组。这个时候我们必须尝试扩容，而这个时候扩容使用的大小使用的则是 `DEFAULT_CAPACITY`。
- `Object[] EMPTY_ELEMENTDATA = {}`
    - 只是一个空数组，没有特别意义，可能有加快分配空数组性能的目的
- `Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {}`
    - 空数组，用于提示第一次扩容时使用 `DEFAULT_CAPACITY`
- `Object[] elementData` 
    - 实际存储数据的数组
- `int size`
    - 记录当前数据结构的大小

# 2. ArrayList 构造

首先看默认构造器，这里可以看出来我们使用的是 `DEFAULTCAPACITY_EMPTY_ELEMENTDATA` 而不是 `EMPTY_ELEMENTDATA`，因为我们-需要进行引用对比，查看我们是否使用的是默认空数组。

```java
public ArrayList() {
    this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
}
```

然后我们看带有 `initialCapacity` 配置的构造器，这里可以看出来，如果 `initialCapacity == 0`，我们就直接用 `EMPTY_ELEMENTDATA` 空数组。如果 `initialCapacity > 0`，那么就正常分配数组。

```java
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
```

对于传入 `Collection` 参数的构造器，我们只是单纯的进行复制，但如果 `Collection` 本身就是空的，我们直接使用 `EMPTY_ELEMENTDATA` 空数组。

```java
public ArrayList(Collection<? extends E> c) {
    Object[] a = c.toArray();
    if ((size = a.length) != 0) {
        if (c.getClass() == ArrayList.class) {
            elementData = a;
        } else {
            elementData = Arrays.copyOf(a, size, Object[].class);
        }
    } else {
        elementData = EMPTY_ELEMENTDATA;
    }
}
```

# 3. add(E) 添加元素和扩容

常用情况下，我们不指定位置，直接插入到尾部。我们重点关注 `add(e, elementData, size)` 方法的使用，`modCount++` 只是用于检测-多线程下对该容器的修改，用于在检测出多线程修改的情况下抛出 `ConcurrentModificationException`。我们可以看出，实现代码中是先扩-容再插入数据的, 因为 `size` 是在插入以后才增加的。

```java
public boolean add(E e) {
    modCount++;
    add(e, elementData, size);
    return true;
}

private void add(E e, Object[] elementData, int s) {
    if (s == elementData.length)
        elementData = grow();
    elementData[s] = e;
    size = s + 1;
}
```

关于扩容的代码比较直接，实质上就是尝试**至少**扩容到 `size + 1` 的大小，也就是方法中的 `minCapacity`， 但是实际扩容完全可能大于这个 `minCapacity`。我们也可以看出，扩容本质上就是对数组的复制，而扩容后的大小由 `newCapacity` 方法决定。

```java
private Object[] grow() {
    return grow(size + 1);
}

private Object[] grow(int minCapacity) {
    return elementData = Arrays.copyOf(elementData, newCapacity(minCapacity));
}
```

`newCapacity` 方法用于判断，我们应该扩容到多少容量。我们必须要保证的是，我们返回的这个大小是大于等于 `minCapacity` 的。`newCapacity` 是目标扩容大小，实质上就是 `elementData.length * 1.5`，也就是 1.5 倍扩容后的大小。

如果 `newCapacity <= minCapacity`，我们通过比较 `DEFAULTCAPACITY_EMPTY_ELEMENTDATA` 引用，查看我们是否使用的是默认-空数组，如果是，我们尝试扩容到 `Math.max(DEFAULT_CAPACITY, minCapacity)`。一般进入到这个分支的情况下，`minCapacity` 的值应该是 1。如果 `minCapacity < 0`，那么就是 `size + 1` overflow 了，所以抛出 `OutOfMemoryError`。

最后，如果 `newCapacity <= MAX_ARRAY_SIZE`，我们返回 `newCapacity`，否则，我们返回 `Integer.MAX_VALUE`。

```java
private int newCapacity(int minCapacity) {
    int oldCapacity = elementData.length;
    int newCapacity = oldCapacity + (oldCapacity >> 1);
    if (newCapacity - minCapacity <= 0) {
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA)
            return Math.max(DEFAULT_CAPACITY, minCapacity);
        if (minCapacity < 0) // overflow
            throw new OutOfMemoryError();
        return minCapacity;
    }
    return (newCapacity - MAX_ARRAY_SIZE <= 0)
        ? newCapacity
        : hugeCapacity(minCapacity);
}

private static int hugeCapacity(int minCapacity) {
    if (minCapacity < 0) // overflow
        throw new OutOfMemoryError();
    return (minCapacity > MAX_ARRAY_SIZE)
        ? Integer.MAX_VALUE
        : MAX_ARRAY_SIZE;
}
```

# 4. add(int, E) 指定位置插入和扩容

这段代码与普通 `add` 方法并没有很大的不同。首先确保 `index` 在 0 到 `size` 范围内。然后检测 `size == elementData.length` 的场景，如果是相等，我们必须进行扩容来确保我们能插入当前 element 到指定位置。扩容完毕后，进行复制。将 `[ index : size-1 ]` 往后移一格，然后将 element 插入到 `index` 的位置。

```java
public void add(int index, E element) {
    rangeCheckForAdd(index);
    modCount++;
    final int s;
    Object[] elementData;
    if ((s = size) == (elementData = this.elementData).length)
        elementData = grow();
    System.arraycopy(elementData, index,
                        elementData, index + 1,
                        s - index);
    elementData[index] = element;
    size = s + 1;
}

private void rangeCheckForAdd(int index) {
    if (index > size || index < 0)
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
}
```

# 5. remove(int) 删除和左移元素

该方法首先检查 `index` 是否大于等于 `size` 的大小，如果是，抛出异常。然后调用 `fastRemove` 方法，将 `[ index+1 : size-1 ]` 的元素往前移一格。

```java
public E remove(int index) {
    Objects.checkIndex(index, size);
    final Object[] es = elementData;

    @SuppressWarnings("unchecked") E oldValue = (E) es[index];
    fastRemove(es, index);

    return oldValue;
}

private void fastRemove(Object[] es, int i) {
    modCount++;
    final int newSize;
    if ((newSize = size - 1) > i)
        System.arraycopy(es, i + 1, es, i, newSize - i);
    es[size = newSize] = null;
}
```







