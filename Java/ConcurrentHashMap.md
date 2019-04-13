# ConcurrentHashMap

## 1. 存储结构

HashEntry 的结构 和 HashMap 中的 Entry 类似，唯一的区别就是其中的核心数据如 value ，以及 next 都是用 volatile 修饰的，保证了获取时的可见性。

```java
static final class HashEntry<K,V> {
    final int hash;
    final K key;
    volatile V value;
    volatile HashEntry<K,V> next;
}
```

ConcurrentHashMap 和 HashMap 实现上类似，最主要的差别是 ConcurrentHashMap 采用了分段锁（Segment），每个分段锁维护着几个桶（HashEntry），多个线程可以同时访问不同分段锁上的桶，从而使其并发度更高（并发度就是 Segment 的个数）。

Segment 继承自 ReentrantLock。

```java
static final class Segment<K,V> extends ReentrantLock implements Serializable {

    private static final long serialVersionUID = 2249069246763182397L;

    static final int MAX_SCAN_RETRIES =
        Runtime.getRuntime().availableProcessors() > 1 ? 64 : 1;

    transient volatile HashEntry<K,V>[] table;

    transient int count;

    transient int modCount;

    transient int threshold;

    final float loadFactor;
}
```

```java
final Segment<K,V>[] segments;
```

默认的并发级别为 16，也就是说默认创建 16 个 Segment。

![img](https://cyc2018.github.io/CS-Notes/pics/deb18bdb-b3b3-4660-b778-b0823a48db12.jpg)

## 2. size 操作

每个 Segment 维护了一个 count 变量来统计该 Segment 中的键值对个数。

```java
/**
 * The number of elements. Accessed only either within locks
 * or among other volatile reads that maintain visibility.
 */
transient int count;
```

put、remove 和 get 操作只需要关心一个 Segment，而 size 操作需要遍历所有的 Segment 才能算出整个 Map 的大小。一个简单的方案是，先锁住所有 Sgment，计算完后再解锁。但这样做，在做 size 操作时，不仅无法对 Map 进行写操作，同时也无法进行读操作，不利于对 Map 的并行操作。

为更好支持并发操作，ConcurrentHashMap 会在不上锁的前提逐个 Segment 计算3次 size，如果某相邻两次计算获取的所有 Segment 的更新次数 (每个 Segment 都与 HashMap 一样通过 modCount 跟踪自己的修改次数，Segment 每修改一次其 modCount 加一) 相等，说明这两次计算过程中无更新操作，则这两次计算出的总 size 相等，可直接作为最终结果返回。如果这三次计算过程中 Map 有更新，则对所有 Segment 加锁重新计算Size。

```java
/**
 * Number of unsynchronized retries in size and containsValue
 * methods before resorting to locking. This is used to avoid
 * unbounded retries if tables undergo continuous modification
 * which would make it impossible to obtain an accurate result.
 */
static final int RETRIES_BEFORE_LOCK = 2;

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
            // 超过尝试次数，则对每个 Segment 加锁
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
            // 连续两次得到的结果一致，则认为这个结果是正确的
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

## 3. put 操作

1. 通过 key 定位到 Segment，在对应的 Segment 中进行具体的 put。

2. 尝试获取锁，如果获取失败肯定就有其他线程存在竞争，则利用 scanAndLockForPut() 自旋获取锁，如果重试的次数达到了 MAX_SCAN_RETRIES 则改为阻塞锁获取，保证能获取成功。

3. 通过 key 的 hashcode 定位到当前 Segment 中的 table 中的 HashEntry。

4. 遍历该 HashEntry，如果不为空则判断传入的 key 和当前遍历的 key 是否相等，相等则覆盖旧的 value。

5. 为空则需要新建一个 HashEntry 并加入到 Segment 中，**在插入之前会先判断是否需要扩容**。

6. 最后解除所获取的当前 Segment 的锁。

## 4. get 操作

get 逻辑比较简单，只需要将 key 通过 Hash 之后定位到具体的 Segment ，再定位到具体的 HashEntry。由于 HashEntry 中的 value 属性是用 volatile 关键词修饰的，保证了内存可见性，所以每次获取时都是最新值。

ConcurrentHashMap 的 get 方法是非常高效的，**因为整个过程都不需要加锁**。

## 5. JDK 1.8 的改动

### put 操作

1.8 的 put 操作采用 CAS + synchronized 的方式：如果 Key 对应的数组元素为 null，则通过 CAS 操作将其设置为当前值。如果 Key 对应的数组元素 (也即链表表头或者树的根元素) 不为 null，则对该元素使用 synchronized 关键字申请锁 (这样也可以看出在新版的 JDK 中对 synchronized 优化是很到位的)，然后进行操作。如果该 put 操作使得当前链表长度超过一定阈值，则将该链表转换为树，从而提高寻址效率。

### size 操作

put 方法和 remove 方法都会通过 addCount 方法维护 Map 的 size。size 方法通过 sumCount 获取由 addCount 方法维护的 Map 的 size。

```java
public int size() {
    long n = sumCount();
    return ((n < 0L) ? 0 :
            (n > (long)Integer.MAX_VALUE) ? Integer.MAX_VALUE :
            (int)n);
}
```

## 参考资料

[ConcurrentHashMap](https://cyc2018.github.io/CS-Notes/#/notes/Java%20%E5%AE%B9%E5%99%A8)

[HashMap? ConcurrentHashMap? 相信看完这篇没人能难住你！](https://crossoverjie.top/2018/07/23/java-senior/ConcurrentHashMap/)

