# LinkedList

```java
public LinkedList() {
    }
public LinkedList(Collection<? extends E> c) {
    this();
    addAll(c);
}

public boolean add(E e) {
        // 直接往队尾加元素
        linkLast(e);
        return true;
    }

    void linkLast(E e) {
        // 保存原来链表尾部节点，last 是全局变量，用来表示队尾元素
        final Node<E> l = last;
        // 为该元素 e 新建一个节点
        final Node<E> newNode = new Node<>(l, e, null);
        // 将新节点设为队尾
        last = newNode;
        // 如果原来的队尾元素为空，那么说明原来的整个列表是空的，就把新节点赋值给头结点
        if (l == null)
            first = newNode;
        else
        // 原来尾结点的后面为新生成的结点
            l.next = newNode;
        // 节点数 +1
        size++;
        modCount++;
    }
public void add(int index, E element) {
        // 检查 index 有没有超出索引范围
        checkPositionIndex(index);
        // 如果追加到尾部，那么就跟 add(E e) 一样了
        if (index == size)
            linkLast(element);
        else
        // 否则就是插在其他位置
            linkBefore(element, node(index));
    }
    Node<E> node(int index) {
        // assert isElementIndex(index);
        // 如果 index 在前半段，从前往后遍历获取 node
        if (index < (size >> 1)) {
            Node<E> x = first;
            for (int i = 0; i < index; i++)
                x = x.next;
            return x;
        } else {
            // 如果 index 在后半段，从后往前遍历获取 node
            Node<E> x = last;
            for (int i = size - 1; i > index; i--)
                x = x.prev;
            return x;
        }
    }
void linkBefore(E e, Node<E> succ) {
        // assert succ != null;
        // 保存 index 节点的前节点
        final Node<E> pred = succ.prev;
        // 新建一个目标节点
        final Node<E> newNode = new Node<>(pred, e, succ);
        succ.prev = newNode;
        // 如果是在开头处插入的话
        if (pred == null)
            first = newNode;
        else
            pred.next = newNode;
        size++;
        modCount++;
    }
public boolean addAll(int index, Collection<? extends E> c) {
        // index 索引范围判断
        checkPositionIndex(index);

        Object[] a = c.toArray();
        int numNew = a.length;
        if (numNew == 0)
            return false;

        // 保存之前的前节点和后节点
        Node<E> pred, succ;
        // 判断是在尾部插入还是在其他位置插入
        if (index == size) {
            succ = null;
            pred = last;
        } else {
            succ = node(index);
            pred = succ.prev;
        }

        for (Object o : a) {
            @SuppressWarnings("unchecked") E e = (E) o;
            Node<E> newNode = new Node<>(pred, e, null);
            // 如果前节点是空的，就说明是在头部插入了
            if (pred == null)
                first = newNode;
            else
                pred.next = newNode;
            pred = newNode;
        }

        if (succ == null) {
            last = pred;
        } else {
            pred.next = succ;
            succ.prev = pred;
        }

        size += numNew;
        modCount++;
        return true;
    }
public E get(int index) {
        checkElementIndex(index);
        return node(index).item;
}
public E remove(int index) {
        checkElementIndex(index);
        return unlink(node(index));
    }

 E unlink(Node<E> x) {
        // assert x != null;
        final E element = x.item;
        final Node<E> next = x.next;
        final Node<E> prev = x.prev;
        // 如果要删除的是头节点，那么设置头节点为下一个节点
        if (prev == null) {
            first = next;
        } else {
            // 设置该节点的前节点的 next 为该节点的 next
            prev.next = next;
            x.prev = null;
        }
        // 如果要删除的是尾节点，那么设置尾节点为上一个节点
        if (next == null) {
            last = prev;
        } else {
            // 设置该节点的下一个节点的 prev 为该节点的 prev
            next.prev = prev;
            x.next = null;
        }
        // 设置 null 值，size--
        x.item = null;
        size--;
        modCount++;
        return element;
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
 public E set(int index, E element) {
        checkElementIndex(index);
        Node<E> x = node(index);
        E oldVal = x.item;
        // 设置 x 节点的值为新值，然后返回旧值
        x.item = element;
        return oldVal;
    }
public void clear() {
        // 遍历链表，然后一一删除置空
        for (Node<E> x = first; x != null; ) {
            Node<E> next = x.next;
            x.item = null;
            x.next = null;
            x.prev = null;
            x = next;
        }
        first = last = null;
        size = 0;
        modCount++;
    }
```

