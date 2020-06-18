# HashMap

## 类结构

`HashMap`继承自`AbstractMap`，实现了`Map`接口，用来存储key-value键值对。

内部两个关键的静态内部类是`Node`和`TreeNode`：

* `Node`实现了`Map.Entry`接口
* `TreeNode`继承自`LinkedHashMap.Entry`，而它又继承自`HashMap.Node`

它们都是用于实现链表的一个节点。

```java
public class HashMap<K,V> extends AbstractMap<K,V>
    implements Map<K,V>, Cloneable, Serializable {
    
    static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        V value;
        Node<K,V> next;
    }
    
    static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {}
}
```



## 存储

内部存储使用的是一个`Node`数组，数组中的每一个元素代表一个链表的Head。相同Hash值的元素会被保存在同一个链表中，这种解决冲突的方法称为拉链法。

```java
transient Node<K,V>[] table;
```

以下参数将会影响存储结构：

```java
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16
static final int MAXIMUM_CAPACITY = 1 << 30;
static final float DEFAULT_LOAD_FACTOR = 0.75f;
static final int TREEIFY_THRESHOLD = 8;
    
transient int size;
int threshold;
final float loadFactor;
```

* `size`：当前元素个数。
* `threshold`：`resize()`扩容操作的阈值，当`size`大小达到这个阈值后，进行扩容。

* `loadFactor`：扩容引子，默认为0.75。

  > threshold = capacity * loadFactor

* `DEFAULT_INITIAL_CAPACITY`：默认初始容量为16。
* `MAXIMUM_CAPACITY`：最大容量为1 << 30。
* `DEFAULT_LOAD_FACTOR`：默认扩容引子为0.75。
* `TREEIFY_THRESHOLD`：链表转化为树形的阈值为8。



## 初始化

源码如下：

```java
public HashMap(int initialCapacity, float loadFactor) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal initial capacity: " +
                                           initialCapacity);
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal load factor: " +
                                           loadFactor);
    this.loadFactor = loadFactor;
    this.threshold = tableSizeFor(initialCapacity);
}

static final int tableSizeFor(int cap) {
    int n = cap - 1;
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```

初始化时会将`treshold`重置为大于等于`initialCapacity`的最小的2的次方（并在第一次`put()`方法时初始化数组）。



## Hash

源码如下：

```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

hash值的计算使用的是`Object.hashCode()`方法再进行高低位的异或。

> 使用高低位异或是为了避免低位相同的哈希值在计算桶位置时发生碰撞，让高位参与到计算中，使哈希表更散列。



## Put

`put()`方法最终调用的是`putVal()`方法。

源码如下：

```java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}

final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    if ((tab = table) == null || (n = tab.length) == 0)
        /* ----- 如果table数组为空，进行初始化 ----- */
        n = (tab = resize()).length;
    if ((p = tab[i = (n - 1) & hash]) == null)
        /* ----- 如果链表为空，初始化链表的Head节点 ----- */
        tab[i] = newNode(hash, key, value, null);
    else {
        Node<K,V> e; K k;
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            /* ----- 如果链表头保存的key值和查询的key相等，选取头部节点 ----- */
            e = p;
        else if (p instanceof TreeNode)
            /* ----- 如果链表节点为红黑树，调用TreeNode的方法进行put ----- */
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    /* ----- 如果链表中没有找到，新建一个节点 ----- */
                    p.next = newNode(hash, key, value, null);
                    /* ----- 如果链表长度超过阈值8，转化为红黑树存储 ----- */
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        if (e != null) { // existing mapping for key
            /* ----- 如果存在一个旧的值 ----- */	
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    /* ----- 如果size超过threshold，进行resize()操作 ----- */	
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}
```

核心步骤为：

* 如果`Table`为空，调用`resize()`初始化链表数组。

* 使用`hash() & (size - 1)`计算链表的下标。

  > 这个操作等同于一个取模操作，前提是哈希表的size值为2的次方。

* 向链表中插入节点。

  * 如果链表为空初始化一个头部节点。
  * 如果链表头部节点存储的是红黑树节点，调用红黑树的插入方法。
  * 递归查找链表中的节点是否哈希值相等并且key的equals()也相等。
    * 如果没有找到key相同的节点，在链表末尾插入一个新节点。并检查是否需要转化为树。
    * 如果找到了相同key的节点，进行更新。

* 检查Table大小是否到达扩容阈值threshold，如果到了扩容阈值，调用`resize()`方法进行扩容。



## Get

`get()`方法最终调用的是`getNode()`方法。

源码如下：

```java
public V get(Object key) {
    Node<K,V> e;
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}

final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) {
        if (first.hash == hash && // always check first node
            /* ----- 如果链表头部节点的key与查询的key相同，返回它的值 ----- */
            ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        if ((e = first.next) != null) {
            if (first instanceof TreeNode)
                /* ----- 如果链表节点为红黑树，调用TreeNode的方法进行get ----- */
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
            do {
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    /* ----- 如果在链表中找到了一个节点的key与查询的key相同，返回它的值 ----- */
                    return e;
            } while ((e = e.next) != null);
        }
    }
    return null;
}
```

核心步骤如下：

* 使用`hash() & (size - 1)`计算链表的下标。
* 依次遍历链表中的节点。
  * 检查头部节点的key是否与查询的key相同，如果相同返回这个节点的值。
  * 如果链表头部节点存储的是红黑树节点，调用红黑树的获取方法。
  * 依次遍历整个链表，检查结点的key是否与查询的key相同，如果相同返回这个节点的值。



## Remove

`remove()`方法最终调用的是`removeNode()`方法。

源码如下：

```java
public V remove(Object key) {
    Node<K,V> e;
    return (e = removeNode(hash(key), key, null, false, true)) == null ?
        null : e.value;
}

final Node<K,V> removeNode(int hash, Object key, Object value,
                               boolean matchValue, boolean movable) {
    Node<K,V>[] tab; Node<K,V> p; int n, index;
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (p = tab[index = (n - 1) & hash]) != null) {
        Node<K,V> node = null, e; K k; V v;
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
			/* ----- 如果链表头部节点的key与查询的key相同，记录要删除的节点为它 ----- */
            node = p;
        else if ((e = p.next) != null) {
            if (p instanceof TreeNode)
                /* ----- 如果链表节点为红黑树，调用TreeNode的方法进行get，记录要删除的节点 ----- */
                node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
            else {
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key ||
                         (key != null && key.equals(k)))) {
                        /* ----- 如果在链表中找到了一个节点的key与查询的key相同，记录要删除的节点为它 ----- */
                        node = e;
                        break;
                    }
                    p = e;
                } while ((e = e.next) != null);
            }
        }
        if (node != null && (!matchValue || (v = node.value) == value ||
                             (value != null && value.equals(v)))) {
            if (node instanceof TreeNode)
                ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
            else if (node == p)
                tab[index] = node.next;
            else
                p.next = node.next;
            ++modCount;
            --size;
            afterNodeRemoval(node);
            return node;
        }
    }
    return null;
}
```

核心步骤如下：

* 使用`hash() & (size - 1)`计算链表的下标。
* 依次遍历链表中的节点。
  * 检查头部节点的key是否与查询的key相同，标记删除节点。
  * 如果链表头部节点存储的是红黑树节点，调用红黑树的方法标记删除的节点。
  * 依次遍历整个链表，检查结点的key是否与查询的key相同，标记删除节点。



## Resize

```java
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    if (oldCap > 0) {
        /* ----- 扩大一倍容量 ----- */
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; // double threshold
    }
    else if (oldThr > 0) // initial capacity was placed in threshold
        /* ----- 初始化时记录的threshold值 ----- */
        newCap = oldThr;
    else {               // zero initial threshold signifies using defaults
        /* ----- 使用默认值 ----- */
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;
    @SuppressWarnings({"rawtypes","unchecked"})
    /* ----- 创建新大小的Node数组 ----- */
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    if (oldTab != null) {
        /* ----- 重新计算桶的位置 ----- */
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
                if (e.next == null)
                    /* ----- 仅有一个节点 ----- */
                    newTab[e.hash & (newCap - 1)] = e;
                else if (e instanceof TreeNode)
                    /* ----- 调用红黑树的方法 ----- */
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else { // preserve order
                    /* ----- 依次遍历链表节点，拆分为两个链表 ----- */
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        next = e.next;
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    if (hiTail != null) {
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;
}
```

核心步骤如下：

* 如果是初始化。
  * 如果设置过初始化大小，初始化数组大小为threshold。
  * 如果没有设置过，使用默认设置。
*  如果不是初始化，数组扩容一倍。
  * 依次遍历旧Table中的链表，重新计算放入新Table。

