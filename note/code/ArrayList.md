# ArrayList

```java
//空参构造,EmptyArray是一个系统的类库
public ArrayList() {
	array = EmptyArray.OBJECT;
}
public ArrayList(int capacity) {
	if (capacity < 0) {
		throw new IllegalArgumentException("capacity < 0: " + capacity);
    }
    array = (capacity == 0 ? EmptyArray.OBJECT : new Object[capacity]);
}
public ArrayList(Collection<? extends E> collection) {
	if (collection == null) {
		throw new NullPointerException("collection == null");
	}
    //list 集合的 toArray 和 Set 集合的 toArray返回的都是 Object[]数组。
	Object[] a = collection.toArray();
	if (a.getClass() != Object[].class) {
		Object[] newArray = new Object[a.length];
		System.arraycopy(a, 0, newArray, 0, a.length);
		a = newArray;
	}
	array = a;
	size = a.length;
}
@Override 
public boolean add(E object) {
	Object[] a = array;
	int s = size;
    //如果集合的长度已经等于数组的长度了,说明数组已经满了,该重新分配新数组
	if (s == a.length) {
        //如果当前的长度小于MIN_CAPACITY_INCREMENT/2(这个常量值是 12)
		Object[] newArray = new Object[s+(s<(MIN_CAPACITY_INCREMENT/2)?MIN_CAPACITY_INCREMENT:s>>1)];
        System.arraycopy(a, 0, newArray, 0, s);
		array = a = newArray;
	}
	a[s] = object;
	size = s + 1;
	modCount++;
	return true;
}
@Override 
public E remove(int index) {
	Object[] a = array;
	int s = size;
	if (index >= s) {
		throwIndexOutOfBoundsException(index, s);	
    }
	@SuppressWarnings("unchecked")
	E result = (E) a[index];
	System.arraycopy(a, index + 1, a, index, --s - index);
	a[s]= null; // Prevent memory leak
	size = s;
	modCount++;
	return result;
}
@Override 
public void clear() {
	if (size != 0) {
		Arrays.fill(array, 0, size, null);
	size = 0;
	modCount++;
	}
}
```

