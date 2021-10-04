# TreeSet

**JDK-11**

`NavigableSet` implementation based on `TreeMap`.

```
    Set
     ^
     |
 SortedSet
     ^
     |
NavigableSet
     ^
     |
AbstractSet
     ^
     |
  TreeSet
```

# 1. TreeSet 属性

与 `LinkedHashMap` 相似, `TreeSet` 内部包含了 `NavigableMap` (记住, `NavigableMap` 继承了 `SortedMap`), 同时该 `Map` 的 `value` 同样使用空对象 `Object PRESENT` 来作为存在的标志。

```java
public class TreeSet<E> extends AbstractSet<E>
    implements NavigableSet<E>, Cloneable, java.io.Serializable
{
    /**
     * The backing map.
     */
    private transient NavigableMap<E,Object> m;

    // Dummy value to associate with an Object in the backing Map
    private static final Object PRESENT = new Object();

}
```

# 2. 构造

本质上就是在构造内部的 `NavigableMap` 或者说 `TreeMap`.

```java
TreeSet(NavigableMap<E,Object> m) {
    this.m = m;
}

public TreeSet() {
    this(new TreeMap<>());
}

public TreeSet(Comparator<? super E> comparator) {
    this(new TreeMap<>(comparator));
}

public TreeSet(Collection<? extends E> c) {
    this();
    addAll(c);
}

public TreeSet(SortedSet<E> s) {
    this(s.comparator());
    addAll(s);
}

public Iterator<E> iterator() {
    return m.navigableKeySet().iterator();
}
```

# 3. 内部代码

大部分方法都是依附于 `NavigableMap` 实现的。

```java
public boolean contains(Object o) {
    return m.containsKey(o);
}

public boolean add(E e) {
    return m.put(e, PRESENT)==null;
}

public boolean remove(Object o) {
    return m.remove(o)==PRESENT;
}

public void clear() {
    m.clear();
}
```

