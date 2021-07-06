# ThreadLocal详解



## 什么是threadLocal？

> This class provides thread-local variables.  These variables differ from their normal counterparts in that each thread that accesses one has its own, independently initialized copy of the variable.

threadLocal是用于线程内部存储的类，通过threadLocal可以实现线程独享的存储空间。不同于线程同步的概念，threadLocal是让每个线程用于独立的数据。

## 使用示例

### 示例代码：

```java
public class ThreadLocalExample {
    static final ThreadLocal<Human> holder = new ThreadLocal<>();

    public static void main(String[] args) {
        Thread t1 = new Thread(()->{
            holder.set(new Human("张三", 21, 0));
            System.out.println(Thread.currentThread().getName() + ":" + holder.get());
        }, "t1");

        Thread t2 = new Thread(()->{
            holder.set(new Human("李四", 22, 0));
            System.out.println(Thread.currentThread().getName() + ":" + holder.get());
        }, "t2");

        t1.start();
        t2.start();

        try{
            t1.join();
            t2.join();
            System.out.println(Thread.currentThread().getName() + ":" + holder.get());
        }catch (Exception e){
            e.printStackTrace();
        }
    }
}
```

### 运行结果：

```
t2:Human{name='李四', age=22, gender=0}
t1:Human{name='张三', age=21, gender=0}
main:null
```

可以发现不同的线程使用同一个threadLocal对象的get方法获得的结果是不一样的。这就表示这是数据是在线程中单独存储的。



## 实现原理

### set（T value）

```java
public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
}
```

从set方法可以发现value实际上是存储在一个叫ThreadLocalMap的对象中的，而这个对象是与当前线程关联的。set方法会先拿到这个map，然后将this和value作为键值对存储。注意，这里的键值对的key是threadLocal对象本身。

如果map为null的话，会先创建map并添加第一个键值对。



### getMap(Thread t)

```java
ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}
```

threadLocal是从线程对象获取到的map对象。

在Thread类中这个map对象定义如下：

```java
/* ThreadLocal values pertaining to this thread. This map is maintained
 * by the ThreadLocal class. */
ThreadLocal.ThreadLocalMap threadLocals = null;
```



### get()

```java
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue();
}
```

与set相同，get同样是从线程对象获取到的map。然后用threadLocal对象作为key来获取entry对象，这里的entry对象与HashMap的entry类似，记录了一个键值对。

可以发现，如果线程的map为null时，会进行一次初始化。

```java
private T setInitialValue() {
    T value = initialValue();
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
    return value;
}
```

### 创建map

通过set和get的分析，threadLocalMap采用了一种**懒汉式**的初始化方法，threadLocalMap的创建其实是在第一次set或get的时候完成的。



## ThreadLocalMap

通过对threadLocal的get、set分析，我们发现这两个方法都是用threadLocal对象作为key在线程对象所拥有的ThreadLocalMap对象上做键值对操作。接下来就来看看ThreadLocalMap是什么。



### 数据结构之 Hash表



Hash表是使用hashcode作为数组下标来存放数据的数据结构。它会先计算出对象的hashcode，然后根据hashcode找到槽位(slot)放入。访问时会计算hashcode然后直接根据下标找到对象。这样使得访问的时间复杂度变成了O（1）。

因为hash算法，可能会出现不同对象拥有相同hashcode 的场景，明显冲突时你不能直接替换槽位里现有的对象。所以需要解决hash冲突的方法：

1. **拉链法**：拉链法是在槽位中构建链表，这样一个槽位就可以存放多个hash冲突的对象。不过因为链表访问时间复杂度位O（N），这种方式降低了访问效率。java中的HashMap等都采用了拉链法。
2. **开放地址法**：开放地址法是在冲突后，从冲突位置向后寻找空位放入。这种方式实现简单，但是使访问和添加效率都降低了。ThreadLocalMap使用的是这种方式。

![两种hash表](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/94d4524579ac43f0b0e6ad3b41d44d03~tplv-k3u1fbpfcp-watermark.image)

### Entry

```java
static class Entry extends WeakReference<ThreadLocal<?>> {
    /** The value associated with this ThreadLocal. */
    Object value;

    Entry(ThreadLocal<?> k, Object v) {
        super(k);
        value = v;
    }
}
```

Entry继承了**WeakReference**，代表Entry对象是一个弱引用。弱引用不管内存空间是否充足，只要发生垃圾回收就会被清除。至于使用弱引用的原因，我会在后面关于内存泄漏的解决中解释。

- **强引用**：使用new创建的对象都是强引用，强引用不会被垃圾回收
- **软引用**：使用SoftReference创建软引用，如果一个对象只有软引用指向，在内存不足时触发的垃圾回收中会被回收
- **弱引用**：使用WeakReference创建弱引用，只有弱引用的对象只要发生垃圾回收就会被回收
- **虚引用**：虚引用主要用来跟踪对象被垃圾回收器回收的活动。虚引用与软引用和弱引用的一个区别在于：虚引用必须和引用队列 （ReferenceQueue）联合使用。

除了弱引用，还可以发现entry不同于HashMap的Entry，它只有value属性，没有key属性。这是因为ThreadLocalMap是使用threadLocal作为key，这里的key-value关系体现在弱引用是引用的ThreadLocal类型。即这个弱引用本身就是key。



### set(ThreadLocal key, T value)

```java
private void set(ThreadLocal<?> key, Object value) {

        // We don't use a fast path as with get() because it is at
        // least as common to use set() to create new entries as
        // it is to replace existing ones, in which case, a fast
        // path would fail more often than not.

        Entry[] tab = table;
        int len = tab.length;
    	// 用threadLocal对象的hashcode计算数组槽位
        int i = key.threadLocalHashCode & (len-1);
    
		// 从i位置开始，寻找空槽位
        for (Entry e = tab[i];
             e != null;
             e = tab[i = nextIndex(i, len)]) {
            
            ThreadLocal<?> k = e.get();
			// key相同，替换值
            if (k == key) {
                e.value = value;
                return;
            }
			// key为null，是脏entry
            if (k == null) {
                // 替换脏entry
                replaceStaleEntry(key, value, i);
                return;
            }
        }
		
        tab[i] = new Entry(key, value);
        nt sz = ++size;
        if (!cleanSomeSlots(i, sz) && sz >= threshold)
            rehash();
}
```

ThreadLocalMap的set方法不同于HashMap的set方法，它不涉及链表或红黑树，因为它是使用开放地址法解决的hash冲突。

之前说过ThreadLocalMap使用ThreadLocal对象作为key，所以第一步会从ThreadLocal对象获取到hashcode并计算出在数组中的下标。

因为使用的是开放地址发解决hash冲突，所以会从这个计算出的下标开始遍历数组。

### nextIndex(int i, int len)

```java
private static int nextIndex(int i, int len) {
	return ((i + 1 < len) ? i + 1 : 0);
}
```

当下标超过len的时候回到0从头开始

### getEntry(ThreadLocal<?> key)

```java
private Entry getEntry(ThreadLocal<?> key) {
        int i = key.threadLocalHashCode & (table.length - 1);
        Entry e = table[i];
    	// key命中
        if (e != null && e.get() == key)
            return e;
        else
            // 没有命中，通过线性探测法向后寻找
            return getEntryAfterMiss(key, i, e);
    }
```

getEntry首先会用ThreadLocal对象的hashcode计算出数组下标，然后判断该下标的entry是否命中。没有命中的话会采用**线性探测法**向后寻找。

### getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e)

```java
private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
        Entry[] tab = table;
        int len = tab.length;
		// 从i位置开始线性探测
        while (e != null) {
            ThreadLocal<?> k = e.get();
            // key命中
            if (k == key)
                return e;
            // 发现k为null的脏entry
            if (k == null)
                expungeStaleEntry(i);
            // 向后遍历
            else
                i = nextIndex(i, len);
            e = tab[i];
        }
        return null;
    }
```

getEntryAfterMiss是getEntry中没有命中后的线性探测。它会从i位置开始向后搜索，如果有命中则返回value，如果找到key为null的值会视其为脏entry而清理掉。



## 内存泄漏

### 什么是内存泄漏

用人话说，内存泄漏就是一块内存无法被访问到，但是又没有被回收清除。比如以下代码中HashMap的内存泄漏：

```java
static HashMap<Object, Object> map = new HashMap<>(16);
    public static void main(String[] args) {
        memoryLeak();
        // 现在如何访问到v1？
    }

    private static void memoryLeak(){
        Object k1 = new Object();
        Object v1 = new Object();
        map.put(k1, v1);
    }
```

因为k1的作用域只有memoryLeak这个方法，方法结束后我们将无法访问到v1对象，但是v1对象因为HashMap的引用并没有被回收，这就导致了一个无法访问的内存空间，也就是内存泄漏。

### ThreadLocalMap如何解决内存泄漏

### ThreadLocal的内存泄漏

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a385bf3d1b8f4d928fa5214c99bc2e23~tplv-k3u1fbpfcp-watermark.image)

ThreadLocal的整个引用关系如图，可以发现ThreadLocal的对象是具有一个强引用的，我们通过这个引用去访问entry。如果这个引用断开了，这个entry将无法被访问到，但是有因为它被ThreadLocalMap强引用，所以没被回收，导致了内存泄漏。

### 弱引用

```java
static class Entry extends WeakReference<ThreadLocal<?>>
```

之前有提到ThreadLocalMap的Entry继承了**WeakReference**。之所以使用WeakReference就是在一定程度上解决内存泄漏。

- WeakReference通过get()方法获取它引用的对象，如果该对象已经被回收了就会返回**null**。
- ThreadLocalMap通过ThreadLocal对象访问value，如果我们没有ThreadLocal对象就访问不到value，也就导致了内存泄漏
- 可以知道弱引用被回收后get()返回null，那么当我们没有ThreadLocal对象的引用时，ThreadLocal对象只有弱引用，它将被回收，此时get(）返回null

**综上**，如果ThreadLocal对象被回收了，那么Entry的弱引用会返回null。这样只要发现null，我们就将这个Entry称为**脏entry（StaleEntry）**，即发生了内存泄漏的entry，需要即时清理。接下来就介绍何时发现null，又如何清理。



### expungeStaleEntry

getEntry时的清理

```java
ThreadLocal<?> k = e.get();
// key命中
if (k == key)
	return e;
// 发现k为null的脏entry
if (k == null)
    // 清理脏entry
	expungeStaleEntry(i);
```

getEntry的时候，当发现e.get()为null是会调用expungeStaleEntry清理脏entry。具体清理过程如下：

```java
private int expungeStaleEntry(int staleSlot) {
            Entry[] tab = table;
            int len = tab.length;
			
    		// part I
            tab[staleSlot].value = null;
            tab[staleSlot] = null;
            size--;

            // part II
            Entry e;
            int i;
            for (i = nextIndex(staleSlot, len);
                 (e = tab[i]) != null;
                 i = nextIndex(i, len)) {
                ThreadLocal<?> k = e.get();
                if (k == null) {
                    e.value = null;
                    tab[i] = null;
                    size--;
                } else {
                    int h = k.threadLocalHashCode & (len - 1);
                    if (h != i) {
                        tab[i] = null;

                        // Unlike Knuth 6.4 Algorithm R, we must scan until
                        // null because multiple entries could have been stale.
                        while (tab[h] != null)
                            h = nextIndex(h, len);
                        tab[h] = e;
                    }
                }
            }
            return i;
        }
```

我们将清理的代码分为Part I,II两个部分。

- **Part I**：这一部分代码将该下标的entry的value设置成了null，这样使得value对象没有引用指向可以被回收。最后将该下标设为null，变成了空位。
- **Part II**：这部分代码是在将后面的entry前移，因为现在清楚了脏entry就多出了一个空位，所以要将后面的非空entry向前移动。这样做的好处是避免了用hashcode没有命中entry时向后线性探测遇到null结束。

### replaceStaleEntry

set()时清理脏entry

```java
ThreadLocal<?> k = e.get();
// key相同，替换值
if (k == key) {
	e.value = value;
	return;
}
// key为null，是脏entry
if (k == null) {
	// 替换脏entry
	replaceStaleEntry(key, value, i);
	return;
}
```

与getEntry时相似，也是在线性探测的过程中发现了k==null触发的清理。

```java
private void replaceStaleEntry(ThreadLocal<?> key, Object value,
                               int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;
    Entry e;
	
    // ---------------- Part I ----------------------------------- 
    
    // Back up to check for prior stale entry in current run.
    // We clean out whole runs at a time to avoid continual
    // incremental rehashing due to garbage collector freeing
    // up refs in bunches (i.e., whenever the collector runs).
    int slotToExpunge = staleSlot;
    for (int i = prevIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = prevIndex(i, len))
        if (e.get() == null)
            slotToExpunge = i;
    
	// ---------------- Part II ----------------------------------- 
    // Find either the key or trailing null slot of run, whichever
    // occurs first
    for (int i = nextIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = nextIndex(i, len)) {
        ThreadLocal<?> k = e.get();

        // If we find key, then we need to swap it
        // with the stale entry to maintain hash table order.
        // The newly stale slot, or any other stale slot
        // encountered above it, can then be sent to expungeStaleEntry
        // to remove or rehash all of the other entries in run.
        if (k == key) {
            e.value = value;

            tab[i] = tab[staleSlot];
            tab[staleSlot] = e;

            // Start expunge at preceding stale entry if it exists
            if (slotToExpunge == staleSlot)
                slotToExpunge = i;
            cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
            return;
        }

        // If we didn't find stale entry on backward scan, the
        // first stale entry seen while scanning for key is the
        // first still present in the run.
        if (k == null && slotToExpunge == staleSlot)
            slotToExpunge = i;
    }

    // If key not found, put new entry in stale slot
    tab[staleSlot].value = null;
    tab[staleSlot] = new Entry(key, value);
	
    // If there are any other stale entries in run, expunge them
    if (slotToExpunge != staleSlot)
        cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
}
```

同样，我们把代码分为两部分。

- Part I：这一部分代码的作用是从当前脏entry位置向前找到最靠前的脏entry，至于具体的原因这里涉及到了垃圾回收器的工作原理，所以不多介绍
- Part II：依然是使用线性探测法向后遍历，当找到目标的key后会把目标key和脏entry交换

### 总结（*）

ThreadLocalMap使用了弱引用和线性探测的方法避免内存泄漏。不过这并不能完全避免内存泄漏，因为只有在使用了get和set，并且遍历到了发生内存泄漏的下标才会清理，如果发生泄露后我们没有调用相关方法去解决，那么内存泄漏依然存在。

要完全解决内存泄漏有两种方式：

- **结束线程，释放掉ThreadLocalMap的内存**：也就是打断之前图中的Thread到entry的强引用链，使垃圾回收清除entry。
- **手动调用remove方法**：在使用完后使用remove移出entry
