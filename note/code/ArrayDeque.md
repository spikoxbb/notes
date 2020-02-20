# ArrayDeque

```java
   transient Object[] elements; // non-private to simplify nested class access

    transient int head;

    transient int tail;
```

```java
    public ArrayDeque() {
        elements = new Object[16];
    }
    public ArrayDeque(int numElements) {
        allocateElements(numElements);
    }
 
    public ArrayDeque(Collection<? extends E> c) {
        allocateElements(c.size());
        addAll(c);
    }

    private void allocateElements(int numElements) {
        elements = new Object[calculateSize(numElements)];
    }

    private static int calculateSize(int numElements) {
        int initialCapacity = MIN_INITIAL_CAPACITY;
        // Find the best power of two to hold elements.
        // Tests "<=" because arrays aren't kept full.
        if (numElements >= initialCapacity) {
            //计算出一个最接近同时大于numElements的2的幂，11----1010 
            initialCapacity = numElements;//1001
            initialCapacity |= (initialCapacity >>>  1);//1111 
            initialCapacity |= (initialCapacity >>>  2);//1111
            initialCapacity |= (initialCapacity >>>  4);//1111
            initialCapacity |= (initialCapacity >>>  8);//1111
            initialCapacity |= (initialCapacity >>> 16);//1111----15
            initialCapacity++;//16

            if (initialCapacity < 0)   // Too many elements, must back off
                initialCapacity >>>= 1;// Good luck allocating 2 ^ 30 elements
        }
        return initialCapacity;
    }
```

add 使用 addLast 在末尾追加元素：

```java
    public boolean add(E e) {
        addLast(e);
        return true;
    }
```

 **这也对应了为什么容量要为 2的幂次方， 当数组大小刚好为 2 的幂次方时，(tail = (tail + 1) & (elements.length - 1) 为零，也就是说如果head也为0，那么就需要扩容了.**

```csharp
public void addLast(E e) {
    if (e == null)
        throw new NullPointerException();
    elements[tail] = e;
    if ( (tail = (tail + 1) & (elements.length - 1)) == head)
        doubleCapacity();
}

public void addFirst(E e) {
    if (e == null)
        throw new NullPointerException();
    elements[head = (head - 1) & (elements.length - 1)] = e;
    // 如果两者相等，说明数组已经放满了元素，需要扩容
    if (head == tail)
        doubleCapacity();
}
```

 因为 head 不一定为零，所以在扩容的时候，需要恢复head = 0；  **这里我们应该推测出整个数组都是存储数据， 为了方便删除数组而不移动元素，便使用了指针记录状态 。**

```dart
    private void doubleCapacity() {
        assert head == tail;
        int p = head;
        int n = elements.length;
        int r = n - p; // number of elements to the right of p
        int newCapacity = n << 1;
        if (newCapacity < 0)
            throw new IllegalStateException("Sorry, deque too big");
        Object[] a = new Object[newCapacity];
        System.arraycopy(elements, p, a, 0, r);
        System.arraycopy(elements, 0, a, r, p);
        elements = a;
        head = 0;
        tail = n;
    }
```

 head 如果到了数组末尾，那么又会通过 (h + 1) & (elements.length - 1) 变为 0 了 .

```java
 public E pollFirst() {
        int h = head;
        @SuppressWarnings("unchecked")
        E result = (E) elements[h];
        // Element is null if deque empty
        if (result == null)
            return null;
        elements[h] = null;     // Must null out slot
        head = (h + 1) & (elements.length - 1);
        return result;
    }
```

pollLast 获取队尾元素，如果队尾元素此时为 0，那么将回到了 数组末尾。

```java
    public E pollLast() {
        int t = (tail - 1) & (elements.length - 1);
        @SuppressWarnings("unchecked")
        E result = (E) elements[t];
        if (result == null)
            return null;
        elements[t] = null;
        tail = t;
        return result;
    }
```

```java
public E getFirst() {
        @SuppressWarnings("unchecked")
        E result = (E) elements[head];
        if (result == null)
            throw new NoSuchElementException();
        return result;
    }

    /**
     * @throws NoSuchElementException {@inheritDoc}
     */
    public E getLast() {
        @SuppressWarnings("unchecked")
        E result = (E) elements[(tail - 1) & (elements.length - 1)];
        if (result == null)
            throw new NoSuchElementException();
        return result;
    }
```