
## 实现简单的LRU Cache

LRU Cache需要提供的功能：

1. 有存储上限
2. set: 保存键值对Entry，若空间已满，移除最久没有使用的一个Entry
3. get: 根据键查找值，并将键调整为最近使用
4. 对时间的要求：
	1. O(1)时间内保存和调整顺序
	2. O(1)时间内查找和调整顺序

```
public class LRUCache<K, V> {

	//使用双向链表
    private static class Node<K, V> {
	    //key是为了删除节点时，根据key在Map中删除Entry
        K key; 
        V value;
        Node<K, V> pre;
        Node<K, V> next;

        public Node() {}

        public Node(K key, V value) {
            this.key = key;
            this.value = value;
        }
    }

    private int size;
    //使用Map保存所有缓存的内容，get的时间复杂度为O(1)
    //value为指向双向链表的指针，可以O(1)寻找节点在双向链表中的位置
    private Map<K, Node<K, V>> cache = new HashMap<>();

    //使用双向链表保存顺序，调整顺序的时间复杂度为O(1)
    private Node<K, V> head = new Node<>();
    private Node<K, V> tail = new Node<>();

    public LRUCache(int size) {
        if (size <= 0) {
            throw new IllegalArgumentException("Cache size must bigger than 0");
        }
        this.size = size;
        head.next = tail;
        tail.pre = head;
    }

    public void set(K k, V v) {
        Node<K, V> old = cache.get(k);
        //没有旧数据，构建Node，put进Cache，插入到链表的头部
        if (old == null) {
            //如果Cache空间已满，删掉最后一个节点（最久没有使用的节点）
            if (cache.size() == size) {
                Node<K, V> remove = removeLast();
                cache.remove(remove.key);
            }
            Node<K, V> node = new Node<>(k, v);
            cache.put(k, node);
            addFirst(node);
        } else {
            old.value = v;
            moveToFirst(old);
        }
    }

    public V get(K k) {
        Node<K, V> node = cache.get(k);
        if (node == null) {
            return null;
        }
        moveToFirst(node);
        return node.value;
    }

    private void moveToFirst(Node<K, V> node) {
        if (node.pre == head) {
            return;
        }
        removeNode(node);
        addFirst(node);
    }

    private void addFirst(Node<K, V> node) {
        if (node.pre == head) {
            return;
        }
        node.next = head.next;
        node.pre = head;
        head.next = node;
        node.next.pre = node;
    }

    private void removeNode(Node<K, V> node) {
        node.pre.next = node.next;
        node.next.pre = node.pre;
    }

    private Node<K, V> removeLast() {
        Node<K, V> remove = tail.pre;
        tail.pre = remove.pre;
        remove.pre.next = tail;
        return remove;
    }

    @Override
    public String toString() {
        List<K> ks = new ArrayList<>(cache.size());
        Node<K, V> n = head.next;
        while (n != tail) {
            ks.add(n.key);
            n = n.next;
        }
        return ks.toString();
    }

}

```

测试数据：

```
        LRUCache<Integer, Integer> cache = new LRUCache<>(3);
        System.out.println("======测试set======");
        cache.set(1, 1);
        System.out.println("after set 1: " + cache);
        cache.set(2, 2);
        System.out.println("after set 2: " + cache);
        cache.set(3, 3);
        System.out.println("after set 3: " + cache);
        cache.set(4, 4);
        System.out.println("after set 4: " + cache);
        cache.set(1, 1);
        System.out.println("after set 1: " + cache);

        System.out.println("\n======测试get======");

        System.out.println("get 1 = " + cache.get(1));
        System.out.println("after get 1: " + cache);
        System.out.println("get 3 = " + cache.get(3));
        System.out.println("after get 3: " + cache);
        System.out.println("get 4 = " + cache.get(4));
        System.out.println("after get 4: " + cache);

        System.out.println("\n======测试更改======");
        System.out.println("get 1: " + cache.get(1));
        cache.set(1, 11);
        System.out.println("after set 1: " + cache);
        System.out.println("get 1: " + cache.get(1));
        
```

测试结果：

```
======测试set======
after set 1: [1]
after set 2: [2, 1]
after set 3: [3, 2, 1]
after set 4: [4, 3, 2]
after set 1: [1, 4, 3]

======测试get======
get 1 = 1
after get 1: [1, 4, 3]
get 3 = 3
after get 3: [3, 1, 4]
get 4 = 4
after get 4: [4, 3, 1]

======测试更改======
get 1: 1
after set 1: [1, 4, 3]
get 1: 11

```