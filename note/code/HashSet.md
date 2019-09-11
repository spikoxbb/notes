# HashSet

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
        // PRESENT = new Object()
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

