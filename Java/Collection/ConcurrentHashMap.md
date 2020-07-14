# ConcurrentHashMap

## 1.7

### Unsafe

```c++
void sun::misc::Unsafe::putObject (jobject obj, jlong offset, jobject value)
{
    jobject *addr = (jobject *) ((char *) obj + offset);
    *addr = value;
}
```

普通的`putObject()`方法，用于对比。

```c++
void sun::misc::Unsafe::putObjectVolatile (jobject obj, jlong offset, jobject value)
{
    write_barrier ();
    volatile jobject *addr = (jobject *) ((char *) obj + offset);
    *addr = value;
}
```

* `write_barrier()`保证指令有序。
* `volatile`保证线程可见性。

加起来实现类似Java的`volatile`关键字的效果。

```c++
jobject sun::misc::Unsafe::getObjectVolatile (jobject obj, jlong offset)
{
    volatile jobject *addr = (jobject *) ((char *) obj + offset);
    jobject result = *addr;
    read_barrier ();
    return result;
}
```

* `read_barrier()`保证指令有序。
* `volatile`保证线程可见性。

加起来实现类似Java的`volatile`关键字的效果。

```c++
void sun::misc::Unsafe::putOrderedObject (jobject obj, jlong offset, jobject value)
{
    volatile jobject *addr = (jobject *) ((char *) obj + offset);
    *addr = value;
}
```

只保证了有序性，没有保证指令重排，可能是在加锁情况下进行优化。

### HashEntry

总结：

* 链表
* 记录了hash值
* `setNext()`是通过Unsafe方法

```java
static final class HashEntry<K,V> {
    final int hash;
    final K key;
    volatile V value;
    volatile HashEntry<K,V> next;

    HashEntry(int hash, K key, V value, HashEntry<K,V> next) {
        this.hash = hash;
        this.key = key;
        this.value = value;
        this.next = next;
    }

    final void setNext(HashEntry<K,V> n) {
        UNSAFE.putOrderedObject(this, nextOffset, n);
    }

    static final sun.misc.Unsafe UNSAFE;
    static final long nextOffset;
    static {
        try {
            UNSAFE = sun.misc.Unsafe.getUnsafe();
            Class k = HashEntry.class;
            // 获取当前class的next变量的偏移
            nextOffset = UNSAFE.objectFieldOffset
                (k.getDeclaredField("next"));
        } catch (Exception e) {
            throw new Error(e);
        }
    }
}
```

### Segment

#### 成员变量

```java
static final class Segment<K,V> extends ReentrantLock implements Serializable {

    private static final long serialVersionUID = 2249069246763182397L;

    // 自旋次数
    static final int MAX_SCAN_RETRIES =
        Runtime.getRuntime().availableProcessors() > 1 ? 64 : 1;

	// 数组+链表存储结构
    transient volatile HashEntry<K,V>[] table;

	// 数量
    transient int count;

    transient int modCount;

	// 扩容阈值
    transient int threshold;

	// 扩容因子
    final float loadFactor;
}
```

#### scanAndLockForPut()

```java
private HashEntry<K,V> scanAndLockForPut(K key, int hash, V value) {
    HashEntry<K,V> first = entryForHash(this, hash);
    HashEntry<K,V> e = first;
    HashEntry<K,V> node = null;
    int retries = -1; // negative while locating node
    //如果尝试加锁失败，那么就对segment[hash]对应的链表进行遍历找到需要put的这个entry所在的链表中的位置，
    //这里之所以进行一次遍历找到坑位，主要是为了通过遍历过程将遍历过的entry全部放到CPU高速缓存中，
    //这样在获取到锁了之后，再次进行定位的时候速度会十分快，这是在线程无法获取到锁前并等待的过程中的一种预热方式。
    while (!tryLock()) {
        HashEntry<K,V> f; // to recheck first below
        //获取锁失败，初始时retries=-1必然开始先进入第一个if
        if (retries < 0) {//<1>
            if (e == null) { //<1.1>
                //e=null代表两种意思，第一种就是遍历链表到了最后，仍然没有发现指定key的entry；
                //第二种情况是刚开始时确实太过entryForHash找到的HashEntry就是空的，即通过hash找到的table中对应位置链表为空
                //当然这里之所以还需要对node==null进行判断，是因为有可能在第一次给node赋值完毕后，然后预热准备工作已经搞定，
                //然后进行循环尝试获取锁，在循环次数还未达到<2>以前，某一次在条件<3>判断时发现有其它线程对这个segment进行了修改，
                //那么retries被重置为-1，从而再一次进入到<1>条件内，此时如果再次遍历到链表最后时，因为上一次遍历时已经给node赋值过了，
                //所以这里判断node是否为空，从而避免第二次创建对象给node重复赋值。
                if (node == null) // speculatively create node
                    node = new HashEntry<K,V>(hash, key, value, null);
                retries = 0;
            }
            else if (key.equals(e.key))//<1.2>   遍历过程发现链表中找到了我们需要的key的坑位
                retries = 0;
            else//<1.3>   当前位置对应的key不是我们需要的，遍历下一个
                e = e.next;
        }
        else if (++retries > MAX_SCAN_RETRIES) {//<2>
            // 尝试获取锁次数超过设置的最大值，直接进入阻塞等待，这就是所谓的有限制的自旋获取锁，
            //之所以这样是因为如果持有锁的线程要过很久才释放锁，这期间如果一直无限制的自旋其实是对系统性能有消耗的，
            //这样无限制的自旋是不利的，所以加入最大自旋次数，超过这个次数则进入阻塞状态等待对方释放锁并获取锁。
            lock();
            break;
        }
        else if ((retries & 1) == 0 &&
                 (f = entryForHash(this, hash)) != first) {//<3>
            // 遍历过程中，有可能其它线程改变了遍历的链表，这时就需要重新进行遍历了。
            e = first = f; // re-traverse if entry changed
            retries = -1;
        }
    }
    return node;
}
```

#### put()

总结：

* 尝试获取锁，如果没有获取到，进行`tryLock()`自旋获取。

* 依次遍历链表，查看有没有Key相同（==或者equals）的节点，有进行值的覆盖。
* 没有找到Key相同的，插入链表头部。
* 超过阈值进行`rehash()`操作，没超过设置到数组中。

```java
final V put(K key, int hash, V value, boolean onlyIfAbsent) {
    //先尝试对segment加锁，如果直接加锁成功，那么node=null；如果加锁失败，则会调用scanAndLockForPut方法去获取锁，
    //在这个方法中，获取锁后会返回对应HashEntry（要么原来就有要么新建一个）
    HashEntry<K,V> node = tryLock() ? null :
        scanAndLockForPut(key, hash, value);
    V oldValue;
    try {
        //这里是一个优化点，由于table自身是被volatile修饰的，然而put这一块代码本身是加锁了的，所以同一时间内只会有一个线程操作这部分内容，
        //所以不再需要对这一块内的变量做任何volatile修饰，因为变量加了volatile修饰后，变量无法进行编译优化等，会对性能有一定的影响
        //故将table赋值给put方法中的一个局部变量，从而使得能够减少volatile带来的不必要消耗。
        HashEntry<K,V>[] tab = table;
        int index = (tab.length - 1) & hash;
        //这里有一个问题：为什么不直接使用数组下标获取HashEntry，而要用entryAt来获取链表？
        //这里结合网上内容个人理解是：由于Segment继承的是ReentrantLock，所以它是一个可重入锁，那么是否存在某种场景下，
        //会导致同一个线程连续两次进入put方法，而由于put最终使用的putOrderedObject只是禁止了写写重排序无法保证内存可见性，
        //所以这种情况下第二次put在获取链表时必须用entryAt中的volatile语义的get来获取链表，因为这种情况下下标获取的不一定是最新数据。
        //不过并没有想到哪里会存在这种场景，有谁能想到的或者是我的理解有误请指出！
        HashEntry<K,V> first = entryAt(tab, index);//先获取需要put的<k,v>对在当前这个segment中对应的链表的表头结点。

        for (HashEntry<K,V> e = first;;) {//开始遍历first为头结点的链表
            if (e != null) {//<1>
                //e不为空，说明当前键值对需要存储的位置有hash冲突，直接遍历当前链表，如果链表中找到一个节点对应的key相同，
                //依据onlyIfAbsent来判断是否覆盖已有的value值。
                K k;
                if ((k = e.key) == key ||
                    (e.hash == hash && key.equals(k))) {
                    //进入这个条件内说明需要put的<k,y>对应的key节点已经存在，直接判断是否更新并最后break退出循环。
                    oldValue = e.value;
                    if (!onlyIfAbsent) {
                        e.value = value;
                        ++modCount;
                    }
                    break;
                }
                e = e.next;//未进入上面的if条件中，说明当前e节点对应的key不是需要的，直接遍历下一个节点。
            }
            else {//<2>
                //进入到这个else分支，说明e为空，对应有两种情况下e可能会为空，即：
                // 1>. <1>中进行循环遍历，遍历到了链表的表尾仍然没有满足条件的节点。
                // 2>. e=first一开始就是null（可以理解为即一开始就遍历到了尾节点）
                if (node != null) //这里有可能获取到锁是通过scanAndLockForPut方法内自旋获取到的，这种情况下依据找好或者说是新建好了对应节点，node不为空
                    node.setNext(first);
                else// 当然也有可能是这里直接第一次tryLock就获取到了锁，从而node没有分配对应节点，即需要给依据插入的k,v来创建一个新节点
                    node = new HashEntry<K,V>(hash, key, value, first);
                int c = count + 1; //总数+1 在这里依据获取到了锁，即是线程安全的！对应了上述对count变量的使用规范说明。
                if (c > threshold && tab.length < MAXIMUM_CAPACITY)//判断是否需要进行扩容
                    //扩容是直接重新new一个新的HashEntry数组，这个数组的容量是老数组的两倍，
                    //新数组创建好后再依次将老的table中的HashEntry插入新数组中，所以这个过程是十分费时的，应尽量避免。
                    //扩容完毕后，还会将这个node插入到新的数组中。
                    rehash(node);
                else
                    //数组无需扩容，那么就直接插入node到指定index位置，这个方法里用的是UNSAFE.putOrderedObject
                    //网上查阅到的资料关于使用这个方法的原因都是说因为它使用的是StoreStore屏障，而不是十分耗时的StoreLoad屏障
                    //给我个人感觉就是putObjectVolatile是对写入对象的写入赋予了volatile语义，但是代价是用了StoreLoad屏障
                    //而putOrderedObject则是使用了StoreStore屏障保证了写入顺序的禁止重排序，但是未实现volatile语义导致更新后的不可见性，
                    //当然这里由于是加锁了，所以在释放锁前会将所有变化从线程自身的工作内存更新到主存中。
                    //这一块对于putOrderedObject和putObjectVolatile的区别有点混乱，不是完全理解，网上也没找到详细解答，查看了C源码也是不大确定。
                    //希望有理解的人看到能指点一下，后续如果弄明白了再更新这一块。
                    setEntryAt(tab, index, node);
                ++modCount;
                count = c;
                oldValue = null;
                break;
            }
        }
    } finally {
        unlock();
    }
    return oldValue;
}
```

#### reshash()

总结：

* 将table中所有的节点重新放置。
  * 找到原链表中尾部的一段，它们index值相同，直接复用这部分链表结构，放置到新的槽位中。
  * 其它部分重新计算槽位进行分配。
* 将新插入的节点放到对应位置的链表头部。

```java
/**
 * Doubles size of table and repacks entries, also adding the
 * given node to new table
 * 对数组进行扩容，由于扩容过程需要将老的链表中的节点适用到新数组中，所以为了优化效率，可以对已有链表进行遍历，
 * 对于老的oldTable中的每个HashEntry，从头结点开始遍历，找到第一个后续所有节点在新table中index保持不变的节点fv，
 * 假设这个节点新的index为newIndex，那么直接newTable[newIndex]=fv，即可以直接将这个节点以及它后续的链表中内容全部直接复用copy到newTable中
 * 这样最好的情况是所有oldTable中对应头结点后跟随的节点在newTable中的新的index均和头结点一致，那么就不需要创建新节点，直接复用即可。
 * 最坏情况当然就是所有节点的新的index全部发生了变化，那么就全部需要重新依据k,v创建新对象插入到newTable中。
*/
@SuppressWarnings("unchecked")
private void rehash(HashEntry<K,V> node) {
    HashEntry<K,V>[] oldTable = table;
    int oldCapacity = oldTable.length;
    int newCapacity = oldCapacity << 1;
    threshold = (int)(newCapacity * loadFactor);
    HashEntry<K,V>[] newTable =
        (HashEntry<K,V>[]) new HashEntry[newCapacity];
    int sizeMask = newCapacity - 1;
    for (int i = 0; i < oldCapacity ; i++) {
        HashEntry<K,V> e = oldTable[i];
        if (e != null) {
            HashEntry<K,V> next = e.next;
            int idx = e.hash & sizeMask;
            if (next == null)   //  Single node on list 只有单个节点
                newTable[idx] = e;
            else { // Reuse consecutive sequence at same slot
                HashEntry<K,V> lastRun = e;
                int lastIdx = idx;
                for (HashEntry<K,V> last = next;
                     last != null;
                     last = last.next) {
                    int k = last.hash & sizeMask;
                    if (k != lastIdx) {
                        lastIdx = k;
                        lastRun = last;
                    }
                }//这个for循环就是找到第一个后续节点新的index不变的节点。
                newTable[lastIdx] = lastRun;
                // Clone remaining nodes
                //第一个后续节点新index不变节点前所有节点均需要重新创建分配。
                for (HashEntry<K,V> p = e; p != lastRun; p = p.next) {
                    int h = p.hash;
                    int k = h & sizeMask;
                    HashEntry<K,V> n = newTable[k];
                    newTable[k] = new HashEntry<K,V>(h, p.key, p.value, n);
                }
            }
        }
    }
    int nodeIndex = node.hash & sizeMask; // add the new node
    node.setNext(newTable[nodeIndex]);
    newTable[nodeIndex] = node;
    table = newTable;
}
```

### ConcurrentHashMap

#### 成员变量

```java
public class ConcurrentHashMap<K, V> extends AbstractMap<K, V>
    implements ConcurrentMap<K, V>, Serializable {

    /**
     * 在构造函数未指定初始大小时，默认使用的map大小
     */
    static final int DEFAULT_INITIAL_CAPACITY = 16;

    /**
     * 默认的扩容因子，当初始化构造器中未指定时使用。
     */
    static final float DEFAULT_LOAD_FACTOR = 0.75f;

    /**
     * 默认的并发度，这里所谓的并发度就是能同时操作ConcurrentHashMap（后文简称为chmap）的线程的最大数量，
     * 由于chmap采用的存储是分段存储，即多个segement，加锁的单位为segment，所以一个cmap的并行度就是segments数组的长度，
     * 故在构造函数里指定并发度时同时会影响到cmap的segments数组的长度，因为数组长度必须是大于并行度的最小的2的幂。
     */
    static final int DEFAULT_CONCURRENCY_LEVEL = 16;

    /**
     * 最大容量
     */
    static final int MAXIMUM_CAPACITY = 1 << 30;

    /**
     * 每个分段最小容量
     */
    static final int MIN_SEGMENT_TABLE_CAPACITY = 2;

    /**
     * 分段最大的容量
     */
    static final int MAX_SEGMENTS = 1 << 16; // slightly conservative

    /**
     * 默认自旋次数，超过这个次数直接加锁，防止在size方法中由于不停有线程在更新map
     * 导致无限的进行自旋影响性能，当然这种会导致ConcurrentHashMap使用了这一规则的方法
     * 如size、clear是弱一致性的。
     */
    static final int RETRIES_BEFORE_LOCK = 2;

    /**
     * 计算segment位置时&操作的mask值。
     */
    final int segmentMask;

    /**
     * 计算segment位置时hash值右移的偏移。
     */
    final int segmentShift;

    /**
     * The segments, each of which is a specialized hash table.
     */
    final Segment<K,V>[] segments;

    transient Set<K> keySet;
    transient Set<Map.Entry<K,V>> entrySet;
    transient Collection<V> values;
}
```

#### 构造函数

* 根据`concurrencyLevel`计算`segments`的数组大小。默认16。
* 根据`initalCapacity`计算每个`segment`中`table`的大小。默认`capacity`为16，最小`table`大小2，因此`table`大小至少为2。
* 只创建一个`segment`，其它`segment`延迟到使用过程中调用`ensureSegment`进行。

```java
public ConcurrentHashMap(int initialCapacity,
                         float loadFactor, int concurrencyLevel) {
    if (!(loadFactor > 0) || initialCapacity < 0 || concurrencyLevel <= 0)
        throw new IllegalArgumentException();
    if (concurrencyLevel > MAX_SEGMENTS)
        concurrencyLevel = MAX_SEGMENTS;
    // Find power-of-two sizes best matching arguments
    int sshift = 0;
    int ssize = 1;
    while (ssize < concurrencyLevel) {
        ++sshift;
        ssize <<= 1;
    }
    this.segmentShift = 32 - sshift;
    this.segmentMask = ssize - 1;
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    int c = initialCapacity / ssize;
    if (c * ssize < initialCapacity)
        ++c;
    int cap = MIN_SEGMENT_TABLE_CAPACITY;
    while (cap < c)
        cap <<= 1;
    // create segments and segments[0]
    Segment<K,V> s0 =
        new Segment<K,V>(loadFactor, (int)(cap * loadFactor),
                         (HashEntry<K,V>[])new HashEntry[cap]);
    Segment<K,V>[] ss = (Segment<K,V>[])new Segment[ssize];
    UNSAFE.putOrderedObject(ss, SBASE, s0); // ordered write of segments[0]
    this.segments = ss;
}
```

#### ensureSegment()

总结：

* 利用`segment0`作为prototype创建新的`table`。
* 自旋进行cas设置`table`，这步骤只有一个线程最终能完成数组的初始化。

```java
private Segment<K,V> ensureSegment(int k) {
    final Segment<K,V>[] ss = this.segments;
    long u = (k << SSHIFT) + SBASE; // raw offset
    Segment<K,V> seg;
    if ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u)) == null) {
        Segment<K,V> proto = ss[0]; // use segment 0 as prototype
        int cap = proto.table.length;
        float lf = proto.loadFactor;
        int threshold = (int)(cap * lf);
        HashEntry<K,V>[] tab = (HashEntry<K,V>[])new HashEntry[cap];
        if ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u))
            == null) { // recheck
            Segment<K,V> s = new Segment<K,V>(lf, threshold, tab);
            while ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u))
                   == null) {
                if (UNSAFE.compareAndSwapObject(ss, u, null, seg = s))
                    break;
            }
        }
    }
    return seg;
}
```

#### get()

总结：

* 通过`segmentShift`和`segmentMask`获取到`segments`数组的内存偏移量。
* 通过`Unsafe`的`getObjectValatile`方法进行volatile语音的读，获得segment。
* 通过`Unsafe`获取链表头进行遍历。

* 弱一致性。`Unsafe`操作获取到的数据是主存中最新的，但是后续遍历过程中，数据可能是被更改的，所以是弱一致性。

```java
public V get(Object key) {
    Segment<K,V> s; // manually integrate access methods to reduce overhead
    HashEntry<K,V>[] tab;
    int h = hash(key);//获取key对应hash值
    long u = (((h >>> segmentShift) & segmentMask) << SSHIFT) + SBASE;//获取对应h值存储所在segments数组中内存偏移量
    if ((s = (Segment<K,V>)UNSAFE.getObjectVolatile(segments, u)) != null &&
        (tab = s.table) != null) {
        //通过Unsafe中的getObjectVolatile方法进行volatile语义的读，获取到segments在偏移量为u位置的分段Segment，
        //并且分段Segment中对应table数组不为空
        for (HashEntry<K,V> e = (HashEntry<K,V>) UNSAFE.getObjectVolatile
             (tab, ((long)(((tab.length - 1) & h)) << TSHIFT) + TBASE);
             e != null; e = e.next) {//获取h对应这个分段中偏移量为xxx下的HashEntry的链表头结点，然后对链表进行 遍历
            //###这里第一次初始化通过getObjectVolatile获取HashEntry时，获取到的是主存中最新的数据，但是在后续遍历过程中，有可能数据被其它线程修改
            //从而导致其实这里最终返回的可能是过时的数据，所以这里就是ConcurrentHashMap所谓的弱一致性的体现，containsKey方法也一样！！！！
            K k;
            if ((k = e.key) == key || (e.hash == h && key.equals(k)))
                return e.value;
        }
    }
    return null;
}
```

#### size()

总结：

* 尝试最多两次获取size，比较`modCount`是否和获取前一致。一致说明没有其它线程进行过操作，跳出循环。
* 超过两次，进行全部`segment`上的加锁。

```java
public int size() {
    // Try a few times to get accurate count. On failure due to
    // continuous async changes in table, resort to locking.
    final Segment<K,V>[] segments = this.segments;
    int size;
    boolean overflow; // true if size overflows 32 bits
    long sum;         // sum of modCounts
    long last = 0L;   // previous sum
    int retries = -1; // first iteration isn't retry
    try {
        for (;;) {
            if (retries++ == RETRIES_BEFORE_LOCK) {
                for (int j = 0; j < segments.length; ++j)
                    ensureSegment(j).lock(); // force creation
            }
            sum = 0L;
            size = 0;
            overflow = false;
            for (int j = 0; j < segments.length; ++j) {
                Segment<K,V> seg = segmentAt(segments, j);
                if (seg != null) {
                    sum += seg.modCount;
                    int c = seg.count;
                    if (c < 0 || (size += c) < 0)
                        overflow = true;
                }
            }
            if (sum == last)
                break;
            last = sum;
        }
    } finally {
        if (retries > RETRIES_BEFORE_LOCK) {
            for (int j = 0; j < segments.length; ++j)
                segmentAt(segments, j).unlock();
        }
    }
    return overflow ? Integer.MAX_VALUE : size;
}
```

### 总结

* `put()`操作在`segement`上加锁。
* `get()`操作用`Unsafe`进行`validate`语义获取，但是没法保证完全的可见性，弱一致性。
* `size()`操作尝试自旋2次计算，如果`modCount`有问题，全段加锁。
* `rehash()`操作进行扩容，在`put()`内，所以也是`segment`上加锁，尝试复用旧的链表。



## 1.8

### initTable()

总结：

* 没有初始化好一直自旋
* 判断`sizectl`有没有被其它线程设置成-1，如果其它线程正在初始化，释放cpu时间片，退出竞争。
* 使用cas将`sizectl`设置为-1，如果设置成功，就使用当前线程进行初始化。

```java
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    // 当Table没有初始化好的时候一直自旋
    while ((tab = table) == null || tab.length == 0) {
        // 如果sizeCtl小于0说明其它线程正在进行初始化
        // 退出cpu时间片
        if ((sc = sizeCtl) < 0)
            Thread.yield(); // lost initialization race; just spin
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
            // 尝试使用cas设置sizectrl为-1，标记当前正在进行Table的初始化
            try {
                // double-check
                if ((tab = table) == null || tab.length == 0) {
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;                    
                    @SuppressWarnings("unchecked")
                    // 创建新的Table
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    table = tab = nt;
                    sc = n - (n >>> 2);
                }
            } finally {
                sizeCtl = sc;
            }
            break;
        }
    }
    return tab;
}
```

### put()

总结：

* 一直自旋直到插入完成
* 如果链表是空的使用cas设置链表头为新的节点
* 如果链表不为空在链表头节点上进行`synchronized`加锁
  * 循环链表进行插入
  * 或者使用红黑树进行插入
* 如果正在进行扩容，让当前线程帮助进行扩容
* 进行扩容

```java
final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException();
    int hash = spread(key.hashCode());
    int binCount = 0;
    // 一直自旋
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        if (tab == null || (n = tab.length) == 0)
            // 如果Table是空的，进行初始化
            tab = initTable();
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            // <1>
            // 如果链表是空的使用cas进行插入新节点到链表头
            if (casTabAt(tab, i, null,
                         new Node<K,V>(hash, key, value, null)))
                break;                   // no lock when adding to empty bin
        }
        else if ((fh = f.hash) == MOVED)
            // 如果正在扩容过程中
            // 让当前线程帮助最初扩容的线程一起进行数据的转移
            tab = helpTransfer(tab, f);
        else {
            // 如果当前链表不为空
            // 在链表头加锁
            V oldVal = null;
            synchronized (f) {
                // double check链表头指针
                // 在<1>中虽然是volatile语义的获取了表头，但是现在可能已经失效了
                // 失效了就需要再次回到for循环自旋了
                if (tabAt(tab, i) == f) {
                    if (fh >= 0) {
                        binCount = 1;
                        // 循环查找链表
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) {
                                // 如果Key匹配上了进行赋值并退出整个for循环
                                oldVal = e.val;
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            Node<K,V> pred = e;
                            // 如果没有找到Key相同的节点就在尾部新增一个节点
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key,
                                                          value, null);
                                break;
                            }
                        }
                    }
                    else if (f instanceof TreeBin) {
                        // 如果链表头部是一个TreeBin，使用TreeBin的put方法
                        Node<K,V> p;
                        binCount = 2;
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                              value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                }
            }
            if (binCount != 0) {
                // 转化为树
                if (binCount >= TREEIFY_THRESHOLD)
                    treeifyBin(tab, i);
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    // 进行扩容
    addCount(1L, binCount);
    return null;
}
```