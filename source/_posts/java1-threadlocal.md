---
title: ThreadLocal
date: 2019-10-06 11:42:02
tags: Java
---

## `ThreadLocal`是什么 ？

`ThreadLocal`是线程局部变量，和普通变量的不同在于：**每个线程持有这个变量的一个副本，可以独立修改(set方法)和访问(get方法)这个变量，并且线程之间不会发生冲突。`ThreadLocal`一般会被`private static`修饰**

所以，`ThreadLocal`与线程同步机制不同，线程同步机制是多个线程共享同一个变量，而`ThreadLocal`是为每一个线程创建一个单独的变量副本，故而每个线程都可以单独地改变自己所拥有的变量副本，而不会影响其他线程所对应的副本。可以说`ThreaddLocal`为多线程环境下变量问题提供了另外一种解决思路。


`ThreadLocal`定义了4个方法：
+ `get()`：返回变量在当前线程中的值
+ `initialValue()`：返回变量在当前线程的初始值
+ `remove()`：移除变量在当前线程的值
+ `set(T value)`：设置变量在当前线程的值  

除了这4个方法，`ThreadLocal`内部还有一个静态内部类`ThreadLocalMap`，该内部类才是实现线程隔离机制的关键，`get()`、`set()`、`remove()`都是基于该内部类操作。`ThreadLocalMap`提供了一种用键值对方式存储每一个线程的变量的方法，key为当前`ThreaddLocal`对象，`value`则是对应线程的变量副本。

对于ThreadLocal需要注意的有亮点：
+ `ThreadLocal`示例本身不存储值，它只是提供了一个在当前线程中找到副本值的key
+ 是`ThreadLocal`包含在`Thread`中，而不是`Thread`包含在`ThreadLocal`中

下图是`Thread`、`ThreadLocal`、`ThreadLocalMap`的关系：

{% asset_img java1-threadlocal-1.png %}


## ThreadLocal使用示例

```
public class SeqCount {
    private static ThreadLocal<Integer> seqCount = new ThreadLocal<Integer>() {
        // 实现initialValue()
        @Override
        public Integer initialValue() {
            return 0;
        }
    };


    public int nextSeq() {
        seqCount.set(seqCount.get() + 1);
        return seqCount.get();
    }


    public static void main(String[] args) {
        SeqCount seqCount = new SeqCount();
        SeqThread thread1 = new SeqThread(seqCount);
        SeqThread thread2 = new SeqThread(seqCount);
        SeqThread thread3 = new SeqThread(seqCount);
        SeqThread thread4 = new SeqThread(seqCount);

        thread1.start();
        thread2.start();
        thread3.start();
        thread4.start();
    }


    private static class SeqThread extends Thread {
        private SeqCount seqCount;
        SeqThread(SeqCount seqCount) {
            this.seqCount = seqCount;
        }

        @Override
        public void run() {
            for (int i = 0; i < 3; i++) {
                System.out.println(Thread.currentThread().getName() + " seqCount :" + seqCount.nextSeq());
            }
        }
    }
}
```
运行结果：
```
Thread-0 seqCount :1
Thread-1 seqCount :1
Thread-1 seqCount :2
Thread-1 seqCount :3
Thread-0 seqCount :2
Thread-0 seqCount :3
Thread-2 seqCount :1
Thread-2 seqCount :2
Thread-2 seqCount :3
Thread-3 seqCount :1
Thread-3 seqCount :2
Thread-3 seqCount :3

Process finished with exit code 0
```
从运行结果可以看出，ThreadLocal确实可以达到线程隔离机制，确保变量的安全性。

## `ThreadLocal`源码解析

`ThreadLocal`虽然解决了多线程变量的复杂问题，但是它的源码却是比较简单的。`ThreadLocalMap`是实现`ThreadLocal`的关键。

### `ThreadLocalMap`

`ThreadLocalMap`内部利用Entry来实现key-value的存储，如下：
```
/**
* The entries in this hash map extend WeakReference, using
* its main ref field as the key (which is always a
* ThreadLocal object).  Note that null keys (i.e. entry.get()
* == null) mean that the key is no longer referenced, so the
* entry can be expunged from table.  Such entries are referred to
* as "stale entries" in the code that follows.
*/
static class Entry extends WeakReference<ThreadLocal<?>> {
    /** The value associated with this ThreadLocal. */
    Object value;

    Entry(ThreadLocal<?> k, Object v) {
        super(k);
        value = v;
    }
}
```
从上面代码中可以看出，`Entry`的`key`就是`ThreadLocal`，而`value`就是值。同时，`Entry`也继承`WeakReference`，所以说`Entry`所对应的`key`（ThreadLocal实例）的引用为一个弱引用。

`ThreadLocalMap`的源码稍微多了点，这里只看两个最核心的方法`getEntry()`、`set(ThreadLocal key, Object value)`:  

**`set(ThreadLocal key, Object value)`**

```
/**
* Set the value associated with key.
*
* @param key the thread local object
* @param value the value to be set
*/
private void set(ThreadLocal<?> key, Object value) {

    // We don't use a fast path as with get() because it is at
    // least as common to use set() to create new entries as
    // it is to replace existing ones, in which case, a fast
    // path would fail more often than not.

    Entry[] tab = table;
    int len = tab.length;
    // 根据 ThreadLocal 的散列值，查找对应元素在数组中的位置
    int i = key.threadLocalHashCode & (len-1);

    // 采用“线性探测法”，寻找合适位置
    for (Entry e = tab[i];
            e != null;
            e = tab[i = nextIndex(i, len)]) {
        ThreadLocal<?> k = e.get();

        // key 存在，直接覆盖
        if (k == key) {
            e.value = value;
            return;
        }

        // key == null，但是存在值（因为此处的e != null），说明之前的ThreadLocal对象已经被回收了
        if (k == null) {
            // 用新元素替换陈旧的元素
            replaceStaleEntry(key, value, i);
            return;
        }
    }


    // ThreadLocal对应的key实例不存在也没有陈旧元素，new 一个
    tab[i] = new Entry(key, value);
    int sz = ++size;

    // cleanSomeSlots 清楚陈旧的Entry（key == null） 
    // 如果没有清理陈旧的 Entry 并且数组中的元素大于了阈值，则进行 rehash
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        rehash();
}
```
这个`set()`操作和`Map`中的`put()`方式有点不一样，虽然它们都是key、value结构，不同在于它们解决散列冲突的方式不一样。集合`Map`的`put()`采用的是拉链法，而`ThreadLocalMap`的`set()`则是采用开放定址法。

`set()`操作除了存储元素外，还有一个很重要的作用，就是`replaceStaleEntry()`和`cleanSomeSlots()`，这两个方法可以清除掉`key==null`的实例，防止内存泄漏。在`set()`方法中还有一个变量很重要：`threadLocalHashCode`，定义如下：
```
private final int threadLocalHashCode = nextHashCode();
```
从名字上我们可以看出，`threadLocalHashCode`应该是`ThreadLocal`的散列值，定义为final，表示`ThreadLocal`一旦创建其散列值就已经确定了，生成过程则是调用`nextHashCode()`：
```
/**
    * The next hash code to be given out. Updated atomically. Starts at
    * zero.
    */
private static AtomicInteger nextHashCode =
    new AtomicInteger();

/**
    * The difference between successively generated hash codes - turns
    * implicit sequential thread-local IDs into near-optimally spread
    * multiplicative hash values for power-of-two-sized tables.
    */
private static final int HASH_INCREMENT = 0x61c88647;

/**
    * Returns the next hash code.
    */
private static int nextHashCode() {
    return nextHashCode.getAndAdd(HASH_INCREMENT);
}
```
`nextHashCode`表示分配下一个`ThreaddLocal`实例的`threadLocalHashCode`的值，`HASH_INCREMENT`则表示分配两个`ThreadLocal`实例的`threadLocalHashCode`的增量，从`nextHashCode()`就可以看出它们的定义。

**getEntry()**  
```
/**
    * Get the entry associated with key.  This method
    * itself handles only the fast path: a direct hit of existing
    * key. It otherwise relays to getEntryAfterMiss.  This is
    * designed to maximize performance for direct hits, in part
    * by making this method readily inlinable.
    *
    * @param  key the thread local object
    * @return the entry associated with key, or null if no such
    */
private Entry getEntry(ThreadLocal<?> key) {
    int i = key.threadLocalHashCode & (table.length - 1);
    Entry e = table[i];
    if (e != null && e.get() == key)
        return e;
    else
        return getEntryAfterMiss(key, i, e);
}
```
由于采用了开放定址法，所以当前key的散列值和元素在数组的索引并不是完全对应的。首先取一个探测数（key的散列值），如果所对应的key就是我们所要找的元素，则返回，否则调用`getEntryAfterMiss()`，如下：
```
/**
    * Version of getEntry method for use when key is not found in
    * its direct hash slot.
    *
    * @param  key the thread local object
    * @param  i the table index for key's hash code
    * @param  e the entry at table[i]
    * @return the entry associated with key, or null if no such
    */
private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
    Entry[] tab = table;
    int len = tab.length;

    while (e != null) {
        ThreadLocal<?> k = e.get();
        if (k == key)
            return e;
        if (k == null)
            expungeStaleEntry(i);
        else
            i = nextIndex(i, len);
        e = tab[i];
    }
    return null;
}
```
这里有一个重要的地方，当`key == null`时，调用了`expungeStaleEntry()`方法，该方法用于处理`key == null`，有利于GC回收，能够有效地避免内存泄漏。

**`get()`**  
返回当前线程所对应的线程变量。
```
    /**
     * Returns the value in the current thread's copy of this
     * thread-local variable.  If the variable has no value for the
     * current thread, it is first initialized to the value returned
     * by an invocation of the {@link #initialValue} method.
     *
     * @return the current thread's value of this thread-local
     */
    public T get() {
        // 获取当前线程
        Thread t = Thread.currentThread();

        // 获取当前线程的成员变量threadLocal
        ThreadLocalMap map = getMap(t);
        if (map != null) {

            // 从当前程程的ThreadLocalMap获取相对应的Entry
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                // 获取目标值
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }
```
首先通过当前线程获取所对应的成员变量`ThreadLocalMap`，然后通过`ThreadLocalMap`获取当前`ThreadLocal`的`Entry`，最后通过所获取的`Entry`获取目标值value。

`getMap()`方法可以获取当前线程所对应的`ThreadLocalMap`，如下：
```
/**
    * Get the map associated with a ThreadLocal. Overridden in
    * InheritableThreadLocal.
    *
    * @param  t the current thread
    * @return the map
    */
ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}
```

**`set(T value)`**  
设置当前线程的线程局部变量的值。  
```
/**
    * Sets the current thread's copy of this thread-local variable
    * to the specified value.  Most subclasses will have no need to
    * override this method, relying solely on the {@link #initialValue}
    * method to set the values of thread-locals.
    *
    * @param value the value to be stored in the current thread's copy of
    *        this thread-local.
    */
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}
```
获取当前线程所对应的`ThreadLocalMap`，如果不为空，则调用`ThreadLocalMap`的`set()`方法，key就是当前`ThreadLocal`；如果不存在，就调用`createMap()`方法新建一个，如下：
```
/**
    * Create the map associated with a ThreadLocal. Overridden in
    * InheritableThreadLocal.
    *
    * @param t the current thread
    * @param firstValue value for the initial entry of the map
    */
void createMap(Thread t, T firstValue) {
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}
```

**`initValue()`**  
返回该线程局部变量的初始值。  
```
/**
    * Returns the current thread's "initial value" for this
    * thread-local variable.  This method will be invoked the first
    * time a thread accesses the variable with the {@link #get}
    * method, unless the thread previously invoked the {@link #set}
    * method, in which case the {@code initialValue} method will not
    * be invoked for the thread.  Normally, this method is invoked at
    * most once per thread, but it may be invoked again in case of
    * subsequent invocations of {@link #remove} followed by {@link #get}.
    *
    * <p>This implementation simply returns {@code null}; if the
    * programmer desires thread-local variables to have an initial
    * value other than {@code null}, {@code ThreadLocal} must be
    * subclassed, and this method overridden.  Typically, an
    * anonymous inner class will be used.
    *
    * @return the initial value for this thread-local
    */
protected T initialValue() {
    return null;
}
```
该方法定义为`protected`级别且返回为`null`，很明显是要子类实现它的，所以我们在使用`ThreadLocal`的时候一般都应该覆盖该方法。该方法不能显示调用，只有在第一次调用`get()`或者`set()`方法时才会被执行，并且仅执行1次。

**`remove()`**  
将当前程程局部变量的值删除。
```
/**
    * Removes the current thread's value for this thread-local
    * variable.  If this thread-local variable is subsequently
    * {@linkplain #get read} by the current thread, its value will be
    * reinitialized by invoking its {@link #initialValue} method,
    * unless its value is {@linkplain #set set} by the current thread
    * in the interim.  This may result in multiple invocations of the
    * {@code initialValue} method in the current thread.
    *
    * @since 1.5
    */
    public void remove() {
        ThreadLocalMap m = getMap(Thread.currentThread());
        if (m != null)
            m.remove(this);
    }
```
该方法的目的是减少内存的占用。当然，我们不需要显示调用该方法，因为一个线程结束后，它所对应的局部变量就会被垃圾回收。


## `ThreadLocal`为什么会内存泄漏

前面提到每个`Thread`都有一个`ThreadLocal.ThreadLocalMap`的map，该map的key为`ThreadLocal`实例，它为一个弱引用，我们知道弱引用有利于GC回收。当`ThreadLocal的key == null`时，GC就会回收这部分空间，但是value却不一定能够被回收，因为他还与Current Thread存在一个强引用关系，如下：

{% asset_img java1-threadlocal-2.png %}

由于存在这个强引用关系，会导致value无法回收。如果这个线程对象不会销毁那么这个强引用关系则会一直存在，就会出现内存泄漏情况。所以说只要这个线程对象能够及时被GC回收，就不会出现内存泄漏。如果碰到线程池，那就更坑了。

**那么要怎么避免这个问题呢？**  
在前面提过，在`ThreadLocalMap`中的`setEntry()`、`getEntry()`，如果遇到`key == null`的情况，会对`value`设置为`null`。当然我们也可以显示调用`ThreadLocal的remove()`方法进行处理。

## 总结

+ ThreadLocal 不是用于解决共享变量的问题的，也不是为了协调线程同步而存在，而是为了方便每个线程处理自己的状态而引入的一个机制。这点至关重要。
+ 每个Thread内部都有一个ThreadLocal.ThreadLocalMap类型的成员变量，该成员变量用来存储实际的ThreadLocal变量副本。
+ ThreadLocal并不是为线程保存对象的副本，它仅仅只起到一个索引的作用。它的主要木得视为每一个线程隔离一个类的实例，这个实例的作用范围仅限于线程内部。
