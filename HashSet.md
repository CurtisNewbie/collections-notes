# HashSet

**JDK-11**

```
AbstractCollection
         ^
         |
    AbstractSet
         ^
         |     contains
      HashSet ----------- HashMap
```

# 1. HashSet 属性

`HashSet` 内部包含了 `HashMap` 和 一个空对象作 `Object PRESENT` 为公用 value。

```java
public class HashSet<E>
    extends AbstractSet<E>
    implements Set<E>, Cloneable, java.io.Serializable
{

    private transient HashMap<E,Object> map;

    // Dummy value to associate with an Object in the backing Map
    private static final Object PRESENT = new Object();

}
```

# 2. 构造

构造的方式与 `HashMap` 构造器基本一致, 因为 `HashSet` 的构造基本依赖于构造内部的 `HashMap`。默认的 `loadFactor` 仍是 `0.75`, 并且默认的 capacity 也仍是 `16`。

```java
public HashSet() {
    map = new HashMap<>();
}

public HashSet(Collection<? extends E> c) {
    map = new HashMap<>(Math.max((int) (c.size()/.75f) + 1, 16));
    addAll(c);
}

public HashSet(int initialCapacity, float loadFactor) {
    map = new HashMap<>(initialCapacity, loadFactor);
}

public HashSet(int initialCapacity) {
    map = new HashMap<>(initialCapacity);
}
```

# 3. 内部实现

内部实现基本都依赖于 `HashMap`:

```java
public Iterator<E> iterator() {
    return map.keySet().iterator();
}

public int size() {
    return map.size();
}

public boolean isEmpty() {
    return map.isEmpty();
}

public boolean contains(Object o) {
    return map.containsKey(o);
}

public boolean add(E e) {
    return map.put(e, PRESENT)==null;
}

public boolean remove(Object o) {
    return map.remove(o)==PRESENT;
}

public void clear() {
    map.clear();
}
```
