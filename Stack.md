# Stack

**JDK-11**

Extension of `Vector` that represents a LIFO stack of objects.

```
AbstractCollection
        ^
        |
   AbstractList
        ^
        |
      Vector
        ^
        |
      Stack
```

`Stack` 继承了 `Vector`， 本质上对 `push` 和 `pop` 的实现就是，将 element 放到数组的 `size` 位置， 并且将 element 从 数组 `size-1` 上取出并且移除。注意，`Stack` 是线程安全的，所以我们完全应该考虑使用 `Deque`。

```java
public
class Stack<E> extends Vector<E> {

    public E push(E item) {
        addElement(item);

        return item;
    }

    public synchronized E pop() {
        E       obj;
        int     len = size();

        obj = peek();
        removeElementAt(len - 1);

        return obj;
    }

    // ....
}
```
