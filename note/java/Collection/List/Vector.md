[TOC]

# Vector**（数组实现、线程同步）**

Vector 与 ArrayList 一样，也是通过数组实现的，不同的是它支持线程的同步，即某一时刻只有一个线程能够写 Vector，避免多线程同时写而引起的不一致性，但实现同步需要很高的花费，因此，访问它比访问 ArrayList 慢。

## 构造器

```java
 public Vector() {
   this(10);
}

public Vector(int initialCapacity) {
        this(initialCapacity, 0);
}

 protected Object[] elementData;
 public Vector(int initialCapacity, int capacityIncrement) {
   super();
   if (initialCapacity < 0) throw new IllegalArgumentException("Illegal Capacity: "+ initialCapacity);
   this.elementData = new Object[initialCapacity];
   this.capacityIncrement = capacityIncrement;
}

  public Vector(Collection<? extends E> c) {
    elementData = c.toArray();
    elementCount = elementData.length;
    if (elementData.getClass() != Object[].class)
      elementData = Arrays.copyOf(elementData, elementCount, Object[].class);
}
```

## add(E e)方法

```java
public synchronized boolean add(E e) {
  modCount++;
  ensureCapacityHelper(elementCount + 1);
  elementData[elementCount++] = e;
  return true;
}

private void ensureCapacityHelper(int minCapacity) {
  if (minCapacity - elementData.length > 0)
    grow(minCapacity);
}

private void grow(int minCapacity) {
  int oldCapacity = elementData.length;
  int newCapacity = oldCapacity + ((capacityIncrement > 0) ? capacityIncrement : oldCapacity);
  if (newCapacity - minCapacity < 0)
    newCapacity = minCapacity;
  if (newCapacity - MAX_ARRAY_SIZE > 0)
    newCapacity = hugeCapacity(minCapacity);
    elementData = Arrays.copyOf(elementData, newCapacity);
}
```

可以看到`Vector`的`add`方法和ArrayList`的`add`方法的逻辑是一样的，但是有一些小的差别：

1. 使用`synchronized`修饰，线程安全。
2. 扩容大小在没有指定时是原大小的2倍进行扩容，而`ArrayList`的扩容大小是加上原大小的>>1，也就是1.5倍进行扩容。

## add(int index, E element方法

```java
public void add(int index, E element) {
   insertElementAt(element, index);
}

public synchronized void insertElementAt(E obj, int index) {
  modCount++;
  if (index > elementCount) {
    throw new ArrayIndexOutOfBoundsException(index + " > " + elementCount);
  }
  ensureCapacityHelper(elementCount + 1);
  System.arraycopy(elementData, index, elementData, index + 1, elementCount - index);
  elementData[index] = obj;
  elementCount++;
}
```

## **remove(int index)**

```java
public synchronized E remove(int index) {
  modCount++;
  if (index >= elementCount)
    throw new ArrayIndexOutOfBoundsException(index);
  E oldValue = elementData(index);
  int numMoved = elementCount - index - 1;
  if (numMoved > 0)
    System.arraycopy(elementData, index+1, elementData, index, numMoved);
  elementData[--elementCount] = null; 
  return oldValue;
}
```

## **remove(Object o)**

```java
public boolean remove(Object o) {
  return removeElement(o);
}

public synchronized boolean removeElement(Object obj) {
  modCount++;
  int i = indexOf(obj);
  if (i >= 0) {
    removeElementAt(i);
    return true;
}
  return false;
}

public synchronized int indexOf(Object o, int index) {
  if (o == null) {
    for (int i = index ; i < elementCount ; i++)
      if (elementData[i]==null)
        return i;
  } else {
    for (int i = index ; i < elementCount ; i++)
      if (o.equals(elementData[i]))
        return i;
  }
  return -1;
}
```

