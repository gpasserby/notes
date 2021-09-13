### ThreadLocal是什么？

在ThreadLocal官方源码注释中描述如下：

```
This class provides thread-local variables. These variables differ from their normal counterparts in that each thread that accesses one (via its get or set method) has its own, independently initialized copy of the variable. ThreadLocal instances are typically private static fields in classes that wish to associate state with a thread (e.g., a user ID or Transaction ID).
Each thread holds an implicit reference to its copy of a thread-local variable as long as the thread is alive and the ThreadLocal instance is accessible; after a thread goes away, all of its copies of thread-local instances are subject to garbage collection (unless other references to these copies exist).
```

ThreadLocal 提供了线程本地的实例。它与普通变量的区别在于，每个使用该变量的线程都会初始化一个完全独立的实例副本。ThreadLocal 变量希望被`private static`修饰，那么该变量可以被线程内该类的所有实例所共享。当一个线程结束时，它所有的ThreadLocal 的实例副本都可被回收。

### 使用场景

如上文所述，ThreadLocal 适用于如下两种场景

- 每个线程需要有自己单独的实例
- 实例需要在多个方法中共享，但不希望被多线程共享

对于第一点，每个线程拥有自己实例，实现它的方式很多。例如可以在线程内部构建一个单独的实例。ThreadLocal 可以以非常方便的形式满足该需求。

对于第二点，可以在满足第一点（每个线程有自己的实例）的条件下，通过方法间引用传递的形式实现。ThreadLocal 使得代码耦合度更低，且实现更优雅。

### 使用示例

```java
public class Test {

    public static void main(String[] args) {

        IntStream.range(0, 100).forEach(i -> {
            new Thread(() -> {
                long num = (long) (Math.random() * 10000);

                Counter counter = new Counter();
                long random = (long) (Math.random() * num);
                Counter luckyGuy = null;
                for (long l = 1; l < num; l++) {
                    if (l == random){
                        luckyGuy = new Counter();
                        continue;
                    }
                    new Counter();
                }
                assert luckyGuy != null;
                System.out.println(counter.getCount() + "------" + luckyGuy.getCount());
            }).start();
        });
    }

    private static class Counter{
        private static ThreadLocal<Integer> counter = ThreadLocal.withInitial(() -> 0);

        public Counter() {
            counter.set(counter.get() + 1);
        }

        public Integer getCount(){
            return counter.get();
        }
    }
}
```

### 示例分析

该实例使用了Integer类型的ThreadLocal变量，初始化值为0，用于计数同线程内Counter类的实例化数量。开启100个线程，并发实例化随机数量的Counter实例。随机选取一个幸运儿实例与最后一个实例比较计数的值。

运行结果：

> 1149------1149
> 1240------1240
> 1292------1292
> 2433------2433
> 1970------1970
> 1903------1903
> 2937------2937
> 1334------1334
>
> ....

从输出结果可以看出。每个线程通过ThreadLoacl的get方法获取的counter值都是不一样的。并且，同一线程内，该类所有实例共享ThreadLocal（需要ThreadLocal被`private static`所修饰）。

### ThreadLocal原理

从上述的使用示例中可以看到，ThreadLocal的操作是通过ThreadLocal.get以及ThreadLocal.set来实现的。这说明我们所存储的值是放在ThreadLocal中的吗？ThreadLocal保存着实例副本吗？一探究竟。

#### ThreadLocal.get方法

```java
public T get() {
    // 获取当前线程
    Thread t = Thread.currentThread();
    // 将当前线程传入方法中，返回一个ThreadLocalMap类型的map
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        // 传入当前ThreadLocal，从ThreadLocalMap中获取一个Entry类型的元素
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            // 从元素中获取值
            T result = (T)e.value;
            // 返回
            return result;
        }
    }
    return setInitialValue();
}
```

读取实例时，线程首先通过`getMap(t)`方法获取自身的 ThreadLocalMap。从如下该方法的定义可见，该 ThreadLocalMap 的实例是 Thread 类的一个字段，即由 Thread 维护 ThreadLocal 对象与具体实例的映射。这就是线程自己的一个“本地”实例副本。通过线程，获取ThreadLocalMap，再从中获取元素。

```java
ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}
```

ThreadLocalMap是ThreadLocal的一个静态内部类，维护着某个 ThreadLocal 与具体实例的映射数组。其中每个 Entry 都是一个对 **键** 的弱引用。另外，每个 Entry 都包含了一个对 **值** 的强引用。

使用弱引用的原因在于，当线程结束时，需要保证线程所访问的所有 ThreadLocal 中对应的映射均删除，否则可能会引起内存泄漏。而使用弱引用后，当没有强引用指向 ThreadLocal 变量时，它可被回收，从而避免 ThreadLocal 不能被回收而造成的内存泄漏的问题。

```java
static class ThreadLocalMap {

    // 弱引用 与防止内存泄漏有关
    static class Entry extends WeakReference<ThreadLocal<?>> {
        /** The value associated with this ThreadLocal. */
        Object value;

        Entry(ThreadLocal<?> k, Object v) {
            super(k);
            value = v;
        }
    }
    
    // 维护一个Entry类型的数组
    private Entry[] table;
}
```

但是，这里又可能出现另外一种内存泄漏的问题。ThreadLocalMap 维护 ThreadLocal 变量与具体实例的映射，当 ThreadLocal 变量被回收后，该映射的键变为 null，该 Entry 无法被移除。从而使得实例被该 Entry 引用而无法被回收造成内存泄漏。

**注：**Entry虽然是弱引用，但它是 ThreadLocal 类型的弱引用（也即上文所述它是对 **键** 的弱引用），而非具体实例的的弱引用，所以无法避免具体实例相关的内存泄漏。

#### ThreadLocal.set方法

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

该方法先获取该线程的 ThreadLocalMap 对象，然后直接将 ThreadLocal 对象（即代码中的 this）与目标实例的映射添加进 ThreadLocalMap 中。当然，如果映射已经存在，就直接覆盖。另外，如果获取到的 ThreadLocalMap 为 null，则先创建该 ThreadLocalMap 对象。

#### 防止内存泄漏

对于已经不再被使用且已被回收的 ThreadLocal 对象，它在每个线程内对应的实例由于被线程的 ThreadLocalMap 的 Entry 强引用，无法被回收，可能会造成内存泄漏。

针对该问题，ThreadLocalMap 的 set 方法中，通过 replaceStaleEntry 方法将所有键为 null 的 Entry 的值设置为 null，从而使得该值可被回收。另外，会在 rehash 方法中通过 expungeStaleEntry 方法将键和值为 null 的 Entry 设置为 null 从而使得该 Entry 可被回收。通过这种方式，ThreadLocal 可防止内存泄漏。

```java
private void set(ThreadLocal<?> key, Object value) {

    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1);

    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
        ThreadLocal<?> k = e.get();

        if (k == key) {
            e.value = value;
            return;
        }

        if (k == null) {
            replaceStaleEntry(key, value, i);
            return;
        }
    }

    tab[i] = new Entry(key, value);
    int sz = ++size;
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        rehash();
}
```

