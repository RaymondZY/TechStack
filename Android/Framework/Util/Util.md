# Util

## ArrayMap

继承自`SimpleArrayMap`，实现了`Map`接口，提供泛型支持的Key-value存储。

```java
public class ArrayMap<K, V> extends SimpleArrayMap<K, V> implements Map<K, V> {
}
```

`Map`功能的实现基本上全部依赖`SimpleArrayMap`，`ArrayMap`类只是实现了标准的`Map`接口。

### 存储

内部存储主要依靠两个数组，一个`int[]`数组用于存储哈希值，一个`Object[]`数组用于存储Key和Value对象。

```java
int[] mHashes;
Object[] mArray;
int mSize;
```

两个数组的对应关系是：

* `mHashs[index]`：哈希值

* `mArray[index << 1]`：Key

* `mArray[(index << 1) + 1]`：Value

### Index

查找Key对应的Index使用的是二分查找的方式。

```java
int indexOf(Object key, int hash) {
    final int N = mSize;

    // 如果Map内容是空的的，返回第0个位置进行插入
    // 取反是为了标记没有找到对应的相同的Key
    // 如果调用方获取到一个负数就知道没找到，再取反一次就得到了希望插入的位置
    if (N == 0) {
        return ~0;
    }

    // 通过二分查找在mHash数组中寻找
    int index = binarySearchHashes(mHashes, N, hash);

    // 如果返回的是一个负数，也代表了需要插入的位置
    // 也就是数组中第一个比hash值大的位置
    // 插入到这里保证mHash数组还是单调递增的
    if (index < 0) {
        return index;
    }

    // 如果Key.equals()说明找到了
    // 直接返回
    if (key.equals(mArray[index<<1])) {
        return index;
    }

    // 在index之后寻找
    // 并用一个变量end记录第一个碰到的Hash值不同的位置
    // 用于返回在数组中插入的位置
    int end;
    for (end = index + 1; end < N && mHashes[end] == hash; end++) {
        if (key.equals(mArray[end << 1])) return end;
    }

    // 在index之前寻找
    for (int i = index - 1; i >= 0 && mHashes[i] == hash; i--) {
        if (key.equals(mArray[i << 1])) return i;
    }

    // 取反返回在数组中需要返回的位置
    return ~end;
}
```

### Put

```java
public V put(K key, V value) {
    final int osize = mSize;
    final int hash;
    int index;
    if (key == null) {
        // null值hash值对应为0
        hash = 0;
        index = indexOfNull();
    } else {
        // 其它情况取Object.hashCode()
        hash = key.hashCode();
        index = indexOf(key, hash);
    }
    if (index >= 0) {
        // 如果找到了对应的Key
        // 直接返回Value值
        index = (index<<1) + 1;
        final V old = (V)mArray[index];
        mArray[index] = value;
        return old;
    }

    // 取反找到需要插入的位置
    index = ~index;
    if (osize >= mHashes.length) {
        // 如果Hash数组已经满了开始进行扩容
        
        // 不足4，扩容到4
        // 不足8，扩容到8
        // 8和8以上，扩容1.5倍
        final int n = osize >= (BASE_SIZE*2) ? (osize+(osize>>1))
            : (osize >= BASE_SIZE ? (BASE_SIZE*2) : BASE_SIZE);

        if (DEBUG) System.out.println(TAG + " put: grow from " + mHashes.length + " to " + n);

        final int[] ohashes = mHashes;
        final Object[] oarray = mArray;
        
        // 重新分配
        allocArrays(n);

        if (CONCURRENT_MODIFICATION_EXCEPTIONS && osize != mSize) {
            throw new ConcurrentModificationException();
        }

        // 拷贝原始值
        if (mHashes.length > 0) {
            if (DEBUG) System.out.println(TAG + " put: copy 0-" + osize + " to 0");
            System.arraycopy(ohashes, 0, mHashes, 0, ohashes.length);
            System.arraycopy(oarray, 0, mArray, 0, oarray.length);
        }

        // 释放临时数组
        freeArrays(ohashes, oarray, osize);
    }

    if (index < osize) {
        if (DEBUG) System.out.println(TAG + " put: move " + index + "-" + (osize-index)
                                      + " to " + (index+1));
        // 将Hash数组index以后的所有元素往后移动1位
        System.arraycopy(mHashes, index, mHashes, index + 1, osize - index);
        // 将Array数组index << 1以后的所有元素往后移动2位
        System.arraycopy(mArray, index << 1, mArray, (index + 1) << 1, (mSize - index) << 1);
    }

    if (CONCURRENT_MODIFICATION_EXCEPTIONS) {
        if (osize != mSize || index >= mHashes.length) {
            throw new ConcurrentModificationException();
        }
    }

    // 存放新的值
    mHashes[index] = hash;
    mArray[index<<1] = key;
    mArray[(index<<1)+1] = value;
    mSize++;
    return null;
}
```

### Remove

```java
public V removeAt(int index) {
    final Object old = mArray[(index << 1) + 1];
    final int osize = mSize;
    final int nsize;
    if (osize <= 1) {
        // Now empty.
        if (DEBUG) System.out.println(TAG + " remove: shrink from " + mHashes.length + " to 0");
        freeArrays(mHashes, mArray, osize);
        mHashes = ContainerHelpers.EMPTY_INTS;
        mArray = ContainerHelpers.EMPTY_OBJECTS;
        nsize = 0;
    } else {
        nsize = osize - 1;
        if (mHashes.length > (BASE_SIZE*2) && mSize < mHashes.length/3) {
            // 如果空的内容比较多
            // 重新压缩数组大小
            final int n = osize > (BASE_SIZE*2) ? (osize + (osize>>1)) : (BASE_SIZE*2);

            if (DEBUG) System.out.println(TAG + " remove: shrink from " + mHashes.length + " to " + n);

            final int[] ohashes = mHashes;
            final Object[] oarray = mArray;
            allocArrays(n);

            if (CONCURRENT_MODIFICATION_EXCEPTIONS && osize != mSize) {
                throw new ConcurrentModificationException();
            }

            if (index > 0) {
				// 拷贝数组
                System.arraycopy(ohashes, 0, mHashes, 0, index);
                System.arraycopy(oarray, 0, mArray, 0, index << 1);
            }
            if (index < nsize) {
                // 平移数组进行删除
                System.arraycopy(ohashes, index + 1, mHashes, index, nsize - index);
                System.arraycopy(oarray, (index + 1) << 1, mArray, index << 1,
                                 (nsize - index) << 1);
            }
        } else {
            if (index < nsize) {
                // 平移数组进行删除
                System.arraycopy(mHashes, index + 1, mHashes, index, nsize - index);
                System.arraycopy(mArray, (index + 1) << 1, mArray, index << 1,
                                 (nsize - index) << 1);
            }
            mArray[nsize << 1] = null;
            mArray[(nsize << 1) + 1] = null;
        }
    }
    if (CONCURRENT_MODIFICATION_EXCEPTIONS && osize != mSize) {
        throw new ConcurrentModificationException();
    }
    mSize = nsize;
    return (V)old;
}
```



## SparseArray

