# ConcurrentLinkedQueue

在使用 ConcurrentLinkedQueue 时,如果涉及到队列是否为空的判断,切记不可使用 size()==0的做法,因为在 size()方法中,是通过遍历整个链表来实现的,在队列元素很多的时候,size()方法十分消耗性能和时间,只是单纯的判断队列为空使用 isEmpty()即可.

ConcurrentLinkedQueue 中有两个 volatile 类型的 Node 节点分别用来存在列表的首尾节点,其中 head节点存放链表第一个 item 为 null 的节点,tail 则总指向最后一个节点。Node 节点内部则维护一个变量 item用来存放节点的值,next 用来存放下一个节点,从而链接为一个单向无界列表。

```java
public ConcurrentLinkedQueue() {
    head = tail = new Node<E>(null);
}
```

ConcurrentLinkedQueue对Node的CAS操作有这样几个：

```java
//更改Node中的数据域item   
boolean casItem(E cmp, E val) {
    return UNSAFE.compareAndSwapObject(this, itemOffset, cmp, val);
}
//更改Node中的指针域next
void lazySetNext(Node<E> val) {
    UNSAFE.putOrderedObject(this, nextOffset, val);
}
//更改Node中的指针域next
boolean casNext(Node<E> cmp, Node<E> val) {
    return UNSAFE.compareAndSwapObject(this, nextOffset, cmp, val);
}
```

offer 操作是在链表末尾添加一个元素:

```java
public boolean offer(E e) {
    //e 为 null 则抛出空指针异常
    checkNotNull(e);
    //构造 Node 节点构造函数内部调用 unsafe.putObject
    final Node<E> newNode = new Node<E>(e);
    //p被认为队列真正的尾节点，tail不一定指向对象真正的尾节点，因为在ConcurrentLinkedQueue中tail是被延迟更新的
    for (Node<E> t = tail, p = t;;) {
        Node<E> q = p.next;
        //如果 q=null 说明 p 是尾节点则插入
        //此时失败的线程看到的q为新成功的线程新插入的节点走向分支３
        if (q == null) {
            //将插入的Node设置成当前队列尾节点p的next节点，p暂时未改变
            if (p.casNext(null, newNode)) {
                if (p != t) // hop two nodes at a time
                    //第二次插入才更新
                    casTail(t, newNode);
                // Failure is OK.
                return true;
            }
            // Lost CAS race to another thread; re-read next
        }
        //部分线程offer部分线程poll
        else if (p == q)
            //多线程操作时候,由于 poll 时候会把老的 head 变为自引用,然后 head 的 next 变为新 head,所以这里需要重新找新的 head,因为新的 head 后面的节点才是激活的节点
            p = (t != (t = tail)) ? t : head;
        else
            //单线程情况p==t,节点失败的线程修改p为q，即新加入的尾节点
            //多线程情况下可能读到修改后tail的结果，t != (t = tail)这个操作并非一个原子操作，此时t也是队列真正的尾节点
            p = (p != t && t != (t = tail)) ? t : q;
    }
}
```

poll方法源码如下：

```csharp
public E poll() {
    restartFromHead:
    for (;;) {
        //变量p作为队列要删除真正的队头节点，h（head）指向的节点并不一定是队列的队头节点。
        for (Node<E> h = head, p = h, q;;) {
            E item = p.item;

            if (item != null && p.casItem(item, null)) {
                // Successful CAS is the linearization point
                // for item to be removed from this queue.
                if (p != h) // hop two nodes at a time
                    updateHead(h, ((q = p.next) != null) ? q : p);
                return item;
            }
            else if ((q = p.next) == null) {
                updateHead(h, p);
                return null;
            }
            else if (p == q)
                continue restartFromHead;
            else
                p = q;
        }
    }
}
```