# 理解LruCache

LRU(Least Recently Used)缓存算法，即为最近最少使用算法，它的核心思想是当缓存满时，会优先淘汰那些近期最少使用的缓存对象。

>LruCache是一个泛型类，它内部采用了一个LinkedHashMap以强引用的方式存储外界的缓存对象，其提供get和put方法来完成缓存的获取和添加。

下面我们具体看下LruCache.java这个类：

```
public class LruCache<K, V> {
    private final LinkedHashMap<K, V> map;
    
    private int size;           // 当前大小
    private int maxSize;        // 最大容量

    private int putCount;       // put次数
    private int createCount;    // 创建次数
    private int evictionCount;  // 回收次数
    private int hitCount;       // 命中次数
    private int missCount;      // 未命中次数
    ......
}
```

可以看到LruCache主要维护一个LinkedHashMap，主要算法原理是把最近使用的对象引用存储在 LinkedHashMap 中。
接下来看构造函数：

```
public LruCache(int maxSize) {
    if (maxSize <= 0) {
        throw new IllegalArgumentException("maxSize <= 0");
    }
    this.maxSize = maxSize;
    this.map = new LinkedHashMap<K, V>(0, 0.75f, true);
}
```

主要设置Cache最大值maxSize，初始化一个LinkedHashMap， accessOrder 设置为 true 来实现 LRU，利用访问顺序而不是插入顺序。

```
public void resize(int maxSize) {
    if (maxSize <= 0) {
        throw new IllegalArgumentException("maxSize <= 0");
    }

    synchronized (this) {
        this.maxSize = maxSize;
    }
    trimToSize(maxSize);
}
```

重新设定Cache最大值

```
public final V get(K key) {
    if (key == null) {
        throw new NullPointerException("key == null");
    }

    V mapValue;
    synchronized (this) {
        mapValue = map.get(key);
        if (mapValue != null) {
            hitCount++;
            return mapValue;
        }
        missCount++;
    }

    /*
     * Attempt to create a value. This may take a long time, and the map
     * may be different when create() returns. If a conflicting value was
     * added to the map while create() was working, we leave that value in
     * the map and release the created value.
     */

    V createdValue = create(key);
    if (createdValue == null) {
        return null;
    }

    synchronized (this) {
        createCount++;
        mapValue = map.put(key, createdValue);

        if (mapValue != null) {
            // There was a conflict so undo that last put
            map.put(key, mapValue);
        } else {
            size += safeSizeOf(key, createdValue);
        }
    }

    if (mapValue != null) {
        entryRemoved(false, key, createdValue, mapValue);
        return mapValue;
    } else {
        trimToSize(maxSize);
        return createdValue;
    }
}
```

get方法，如果Cache中存在该item，返回item；如果不存在，则创建item。

```
public final V put(K key, V value) {
    if (key == null || value == null) {
        throw new NullPointerException("key == null || value == null");
    }

    V previous;
    synchronized (this) {
        putCount++;
        size += safeSizeOf(key, value);
        previous = map.put(key, value);
        if (previous != null) {
            size -= safeSizeOf(key, previous);
        }
    }

    if (previous != null) {
        entryRemoved(false, key, previous, value);
    }

    trimToSize(maxSize);
    return previous;
}
```

往Cache中put相应item。

对于get和put的基本原理，这里引用一篇文章中的图片说明[图片来源](https://www.jianshu.com/p/b49a111147ee)：

![LruCache基本原理](https://upload-images.jianshu.io/upload_images/3985563-33560a9500e72780.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000/format/webp)

这张图很好地说明了最近最少的思想。

上面几个方法中都提到了trimToSize：

```
public void trimToSize(int maxSize) {
    while (true) {
        K key;
        V value;
        synchronized (this) {
            if (size < 0 || (map.isEmpty() && size != 0)) {
                throw new IllegalStateException(getClass().getName()
                        + ".sizeOf() is reporting inconsistent results!");
            }

            if (size <= maxSize) {
                break;
            }

            Map.Entry<K, V> toEvict = map.eldest();
            if (toEvict == null) {
                break;
            }

            key = toEvict.getKey();
            value = toEvict.getValue();
            map.remove(key);
            size -= safeSizeOf(key, value);
            evictionCount++;
        }

        entryRemoved(true, key, value, null);
    }
}
```

重新计算Cache大小，将最久没有用到的eldest删除。

```
public final V remove(K key) {
    if (key == null) {
        throw new NullPointerException("key == null");
    }

    V previous;
    synchronized (this) {
        previous = map.remove(key);
        if (previous != null) {
            size -= safeSizeOf(key, previous);
        }
    }

    if (previous != null) {
        entryRemoved(false, key, previous, null);
    }

    return previous;
}
```

删除key

`protected void entryRemoved(boolean evicted, K key, V oldValue, V newValue) {}`

当 item 被回收或者删掉时调用。该方法当 value 被回收释放存储空间时被 remove 调用，或者替换 item 值时 put 调用，默认实现什么都没做，使用时可以根据需要重写。

```
protected V create(K key) {
    return null;
}
```

同样，可以根据需要重写。

```
private int safeSizeOf(K key, V value) {
    int result = sizeOf(key, value);
    if (result < 0) {
        throw new IllegalStateException("Negative size: " + key + "=" + value);
    }
    return result;
}

protected int sizeOf(K key, V value) {
    return 1;
}
```

返回一组key/value的entry大小，默认为1

```
public final void evictAll() {
    trimToSize(-1); // -1 will evict 0-sized elements
    }
```

清空Cache，trimToSize(-1)。

```
/**
 * For caches that do not override {@link #sizeOf}, this returns the number
 * of entries in the cache. For all other caches, this returns the sum of
 * the sizes of the entries in this cache.
 */
public synchronized final int size() {
    return size;
}

/**
 * For caches that do not override {@link #sizeOf}, this returns the maximum
 * number of entries in the cache. For all other caches, this returns the
 * maximum sum of the sizes of the entries in this cache.
 */
public synchronized final int maxSize() {
    return maxSize;
}

/**
 * Returns the number of times {@link #get} returned a value that was
 * already present in the cache.
 */
public synchronized final int hitCount() {
    return hitCount;
}

/**
 * Returns the number of times {@link #get} returned null or required a new
 * value to be created.
 */
public synchronized final int missCount() {
    return missCount;
}

/**
 * Returns the number of times {@link #create(Object)} returned a value.
 */
public synchronized final int createCount() {
    return createCount;
}

/**
 * Returns the number of times {@link #put} was called.
 */
public synchronized final int putCount() {
    return putCount;
}

/**
 * Returns the number of values that have been evicted.
 */
public synchronized final int evictionCount() {
    return evictionCount;
}

/**
 * Returns a copy of the current contents of the cache, ordered from least
 * recently accessed to most recently accessed.
 */
public synchronized final Map<K, V> snapshot() {
    return new LinkedHashMap<K, V>(map);
}

@Override public synchronized final String toString() {
    int accesses = hitCount + missCount;
    int hitPercent = accesses != 0 ? (100 * hitCount / accesses) : 0;
    return String.format("LruCache[maxSize=%d,hits=%d,misses=%d,hitRate=%d%%]",
            maxSize, hitCount, missCount, hitPercent);
}
```

其他一些方法。
