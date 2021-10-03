# Vector

**JDK-11**

Synchronized version of `ArrayList`.

```
AbstractCollection
        ^
        |
   AbstractList
        ^
        |
     Vector 
```

`Vector` 与 `ArrayList` 的实现基本上没有很大的区别。属性上来说，也基本一致，也是使用数组进行数据存储。

```java
public class Vector<E>
    extends AbstractList<E>
    implements List<E>, RandomAccess, Cloneable, java.io.Serializable
{

    protected Object[] elementData;

    protected int elementCount;

    protected int capacityIncrement;

}
```

最大的不同是方法签名上的 `synchronized` 关键字。

```java
public synchronized void addElement(E obj) {
    modCount++;
    add(obj, elementData, elementCount);
}
```

不过扩容的 `factor` 不太一致，在 `Vector` 中，如果我们设置了 `capacityIncrement` 我们会在扩容的时候，使用该变量 (`newCapacity = oldCapacity + capacityIncrement`)。如果没有进行设置，我们会扩容两倍大小 (`newCapacity = oldCapacity + oldCapacity`)。

```java
private Object[] grow(int minCapacity) {
    return elementData = Arrays.copyOf(elementData,
                                        newCapacity(minCapacity));
}

private Object[] grow() {
    return grow(elementCount + 1);
}

private int newCapacity(int minCapacity) {
    int oldCapacity = elementData.length;
    int newCapacity = oldCapacity + ((capacityIncrement > 0) ?
                                        capacityIncrement : oldCapacity);
    if (newCapacity - minCapacity <= 0) {
        if (minCapacity < 0) // overflow
            throw new OutOfMemoryError();
        return minCapacity;
    }
    return (newCapacity - MAX_ARRAY_SIZE <= 0)
        ? newCapacity
        : hugeCapacity(minCapacity);
}
```


