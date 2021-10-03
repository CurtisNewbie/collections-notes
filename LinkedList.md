# LinkedList

**JDK-11**

Doubly-linked list implementation.

```
 AbstractCollection
         ^
         |
    AbstractList
         ^
         |
AbstractSequentialList
         ^
         |
     LinkedList
```

interface:

```
Deque
  ^
  |
Queue    List
  ^       ^
  +---+---+
      |
  LinkedList
```

# 1. Node 构造

每一个 `Node` 都带有上一个和下一个节点的引用，双向链表。

```java
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

# 2. LinkedList 属性

`LinkedList` 内部包含三个属性，`size` 只用于记录存储元素数量，`first` 记录头节点, `last` 记录尾节点。

```java
public class LinkedList<E>
    extends AbstractSequentialList<E>
    implements List<E>, Deque<E>, Cloneable, java.io.Serializable
{

    transient int size = 0;

    transient Node<E> first;

    transient Node<E> last;

}
```

# 3. 增加节点

因为 `LinkedList` 使用双向链表并且实现了 `Deque` 接口，节点可以在头和尾部插入，默认 `add(E)` 插入尾部。无论是使用 `offer*` 还是 `add*` 类型命名方式的方法，核心都是在于 `linkFirst(E)` 和 `linkLast(E)` 方法中。

```java
public boolean add(E e) {
    linkLast(e);
    return true;
}

public boolean offer(E e) {
    return add(e);
}

public void addFirst(E e) {
    linkFirst(e);
}

public void addLast(E e) {
    linkLast(e);
}

public boolean offerFirst(E e) {
    addFirst(e);
    return true;
}

public boolean offerLast(E e) {
    addLast(e);
    return true;
}
```

## 3.1 linkFirst(E) 和 linkLast(E) 方法

实现方式基本上就是创建新节点，如果有头/尾节点，放到后面/前面。如果没有头/尾节点，设置为头/尾节点。`modCount` 用于检测是否有多线程-同时修改数据结构。

```java
private void linkFirst(E e) {
    final Node<E> f = first;
    // new Node(prev, element, next)
    final Node<E> newNode = new Node<>(null, e, f);
    first = newNode;
    if (f == null)
        last = newNode;
    else
        f.prev = newNode;
    size++;
    modCount++;
}

void linkLast(E e) {
    final Node<E> l = last;
    // new Node(prev, element, next)
    final Node<E> newNode = new Node<>(l, e, null);
    last = newNode;
    if (l == null)
        first = newNode;
    else
        l.next = newNode;
    size++;
    modCount++;
}
```

## 4. 删除节点

与添加节点相似，节点可在头部或尾部删除，主要核心也是在 `unlinkFirst(Node)`, `unlinkLast(Node)` 和 `unlink(Node)` 方法中。

```java
public E removeFirst() {
    final Node<E> f = first;
    if (f == null)
        throw new NoSuchElementException();
    return unlinkFirst(f);
}

public E removeLast() {
    final Node<E> l = last;
    if (l == null)
        throw new NoSuchElementException();
    return unlinkLast(l);
}

public E remove(int index) {
    checkElementIndex(index);
    return unlink(node(index));
}

public boolean remove(Object o) {
    if (o == null) {
        for (Node<E> x = first; x != null; x = x.next) {
            if (x.item == null) {
                unlink(x);
                return true;
            }
        }
    } else {
        for (Node<E> x = first; x != null; x = x.next) {
            if (o.equals(x.item)) {
                unlink(x);
                return true;
            }
        }
    }
    return false;
}
```

## 4.1 unlinkFirst(Node) 和 unlinkLast(Node) 方法

对于头节点和尾节点的 `unlink` 本质上一样。对于 `unlinkFirst` 方法，基本上就是，如果 `f.next == null` 那么链表只有这一个节点, 所以 `first = null` 并且 `last = null`。如果 `f.next != null`，那么我们将下一个的节点 `f.next` 设置为 `first`。

```java
private E unlinkFirst(Node<E> f) {
    final E element = f.item;
    final Node<E> next = f.next;
    f.item = null;
    f.next = null; // help GC
    first = next;
    if (next == null)
        last = null;
    else
        next.prev = null;
    size--;
    modCount++;
    return element;
}

private E unlinkLast(Node<E> l) {
    final E element = l.item;
    final Node<E> prev = l.prev;
    l.item = null;
    l.prev = null; // help GC
    last = prev;
    if (prev == null)
        first = null;
    else
        prev.next = null;
    size--;
    modCount++;
    return element;
}
```

## 4.2 unlink(Node) 方法

本质这段代码就是: 

1. 使用 `node(int)` 找到对应位置的节点
    - 如果 `index < (size / 2)` 从头开始找，否则从尾开始找
2. 使用 `unlink(Node)` 删除该节点
    - 如果 `x.prev == null`，设 `x.prev` 为 `first`，否则连接 `x.prev.next = x.next`
    - 如果 `x.next == null`，设 `last` 为 `x.prev`, 否则连接 `x.next.prev = x.prev`

```java
public E remove(int index) {
    checkElementIndex(index);
    return unlink(node(index));
}

Node<E> node(int index) {
    if (index < (size >> 1)) {
        Node<E> x = first;
        for (int i = 0; i < index; i++)
            x = x.next;
        return x;
    } else {
        Node<E> x = last;
        for (int i = size - 1; i > index; i--)
            x = x.prev;
        return x;
    }
}

E unlink(Node<E> x) {
    final E element = x.item;
    final Node<E> next = x.next;
    final Node<E> prev = x.prev;
 
    if (prev == null) {
        first = next;
    } else {
        prev.next = next;
        x.prev = null;
    }

    if (next == null) {
        last = prev;
    } else {
        next.prev = prev;
        x.next = null;
    }

    x.item = null;
    size--;
    modCount++;
    return element;
}
```
