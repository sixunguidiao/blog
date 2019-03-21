# WeakHashMap

## 1. 存储结构

WeakHashMap 的 Entry 继承自 WeakReference，被 WeakReference 关联的对象在下一次垃圾回收时会被回收 (而HashMap 中由于每个元素都有一个桶去引用，因此不会被垃圾回收器回收)。

更直观的说，当使用 WeakHashMap 时，即使没有显示的添加或删除任何元素，也可能发生如下情况：

- 调用两次`size()`方法返回不同的值；
- 两次调用`isEmpty()`方法，第一次返回`false`，第二次返回`true`；
- 两次调用`containsKey()`方法，第一次返回`true`，第二次返回`false`，尽管两次使用的是同一个`key`；
- 两次调用`get()`方法，第一次返回一个`value`，第二次返回`null`，尽管两次使用的是同一个对象。

遇到这么奇葩的现象，你是不是觉得使用者一定会疯掉？其实不然，**WeekHashMap 的这个特点特别适用于需要缓存的场景**。在缓存场景下，由于内存是有限的，不能缓存所有对象；对象缓存命中可以提高系统效率，但缓存 MISS 也不会造成错误，因为可以通过计算重新得到。

要明白 WeekHashMap 的工作原理，还需要引入一个概念：**弱引用（WeakReference）**。我们都知道 Java 中内存是通过 GC 自动管理的，GC 会在程序运行过程中自动判断哪些对象是可以被回收的，并在合适的时机进行内存释放。GC 判断某个对象是否可被回收的依据是，**是否有有效的引用指向该对象**。如果没有有效引用指向该对象（基本意味着不存在访问该对象的方式），那么该对象就是可回收的。这里的**“有效引用”**并不包括**弱引用**。也就是说，**虽然弱引用可以用来访问对象，但进行垃圾回收时弱引用并不会被考虑在内，仅有弱引用指向的对象仍然会被 GC 回收**。

```java
private static class Entry<K,V> extends WeakReference<Object> implements Map.Entry<K,V>
```

## 2. Tomcat 中的 ConcurrentCache (*)

Tomcat 中的 ConcurrentCache 使用了 WeakHashMap 来实现缓存功能。

ConcurrentCache 采取的是分代缓存：

- 经常使用的对象放入 eden 中，eden 使用 ConcurrentHashMap 实现，不用担心会被回收（伊甸园）；
- 不常用的对象放入 longterm，longterm 使用 WeakHashMap 实现，这些老对象会被垃圾收集器回收。
- 当调用 get() 方法时，会先从 eden 区获取，如果没有找到的话再到 longterm 获取，当从 longterm 获取到就把对象放入 eden 中，从而保证经常被访问的节点不容易被回收。
- 当调用 put() 方法时，如果 eden 的大小超过了 size，那么就将 eden 中的所有对象都放入 longterm 中，利用虚拟机回收掉一部分不经常使用的对象。

```java
public final class ConcurrentCache<K, V> {

    private final int size;

    private final Map<K, V> eden;

    private final Map<K, V> longterm;

    public ConcurrentCache(int size) {
        this.size = size;
        this.eden = new ConcurrentHashMap<>(size);
        this.longterm = new WeakHashMap<>(size);
    }

    public V get(K k) {
        V v = this.eden.get(k);
        if (v == null) {
            v = this.longterm.get(k);
            if (v != null)
                this.eden.put(k, v);
        }
        return v;
    }

    public void put(K k, V v) {
        if (this.eden.size() >= size) {
            this.longterm.putAll(this.eden);
            this.eden.clear();
        }
        this.eden.put(k, v);
    }
}
```
