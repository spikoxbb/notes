[TOC]

# HashSet

哈希表边存放的是哈希值。HashSet 存储元素的顺序并不是按照存入时的顺序（和 List 显然不同） 而是按照哈希值来存的所以取数据也是按照哈希值取得。元素的哈希值是通过元素的hashcode 方法来获取的, HashSet 首先判断两个元素的哈希值，如果哈希值一样，接着会比较equals 方法 如果 equls 结果为 true ，HashSet 就视为同一个元素。

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

    //不是 public 的，所以不对外公开。
    HashSet(int initialCapacity, float loadFactor, boolean dummy) {
        map = new LinkedHashMap<>(initialCapacity, loadFactor);
    }

//value 是 PRESENT ，其实就是 new Object() 
public boolean add(E e) {
        // final PRESENT = new Object()
        return map.put(e, PRESENT)==null;
    }

 public boolean remove(Object o) {
        return map.remove(o)==PRESENT;
    }

  public boolean contains(Object o) {
        return map.containsKey(o);
    }

 public Iterator<E> iterator() {
        return map.keySet().iterator();
    }

 public int size() {
        return map.size();
    }
```

Set的remove是移除一个元素,并且返回是否移除成功的boolean，而HashSet的remove是使用HashMap实现,则是map.remove

而map的移除会返回value,如果底层value都是存null,显然将无法分辨是否移除成功.