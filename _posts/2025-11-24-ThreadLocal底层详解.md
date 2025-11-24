---
layout: post
categories: 技术文章
author: chen
title: ThreadLocal底层详解
---
## 一、什么是ThreadLocal

多线程操作同一变量，会产生并发安全问题，ThreadLocal提供了一种能力，使得每个线程对于同一变量有自己的专属本地变量，这样多线程操作就互不干扰，见以下示例

```
public class ThreadLocalExample {
    private static ThreadLocal<Integer> threadLocal = ThreadLocal.withInitial(() -> 0);

    public static void main(String[] args) {
        Runnable task = () -> {
            int value = threadLocal.get();
            value += 1;
            threadLocal.set(value);
            System.out.println(Thread.currentThread().getName() + " Value: " + threadLocal.get());
        };

        Thread thread1 = new Thread(task, "Thread-1");
        Thread thread2 = new Thread(task, "Thread-2");

        thread1.start(); // 输出: Thread-1 Value: 1
        thread2.start(); // 输出: Thread-2 Value: 1
    }
}
```

## 二、ThreadLocal底层原理

能够独享本地变量的原理实际非常简单，每一个线程Thread类里有一个成员变量threadLocals存储了本地数据，而ThreadLocal对象本身不负责任何存储的工作，可以把它当成一个转换器。

```
public class Thread implements Runnable {
    //......
    //与此线程有关的ThreadLocal值。由ThreadLocal类维护
    ThreadLocal.ThreadLocalMap threadLocals = null;
}
```

![ThreadLocal 数据结构](https://raw.githubusercontent.com/chen-1110/image/main/threadlocal-data-structure.png)

### set源码

```
public void set(T value) {
    //获取当前请求的线程
    Thread t = Thread.currentThread();
    //取出 Thread 类内部的 threadLocals 变量(哈希表结构)
    ThreadLocalMap map = getMap(t);
    if (map != null)
        // 将需要存储的值放入到这个哈希表中
        map.set(this, value);
    else
        createMap(t, value);
}
```

有以下几个步骤

1.getMap拿到threadLocals

```java
ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}
```

2.set进这个ThreadLocalMap，threadLocalMap的底层数据结构类似jdk7的hashMap，用数组存储，key是threadLocal对象的hash值，value是要存的值。

```
static class ThreadLocalMap {

        static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;
            
            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }
        
        private void set(ThreadLocal<?> key, Object value) {

            // We don't use a fast path as with get() because it is at
            // least as common to use set() to create new entries as
            // it is to replace existing ones, in which case, a fast
            // path would fail more often than not.

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
}
```

### ThreadLocal内存泄漏及解决

理解了底层原理，这个非常好理解，内存泄漏发生的条件有以下两个

1.创建的ThreadLocal对象为非静态变量，在其生命周期结束后，对象会被gc回收掉

2.多线程长期存活（如线程池内对象）

3.ThreadLocal对象set后，不做remove等清除操作

此时由于ThreadLocal对象使用结束，底层thread对象的threadLocals成员变量key为弱引用，会触发gc回收变为null，但是value值会一直保留下，不会被gc，从而导致value值持续堆积，最终可能会OOM，示例代码如下：

```
public class LeakDemo {
	//一个线程，无界阻塞队列（Integer.MAX）
    private static ExecutorService pool = Executors.newFixedThreadPool(1);

    public static void main(String[] args) {
        for (int i = 0; i < 10000; i++) {
            pool.execute(() -> {
                ThreadLocal<byte[]> tl = new ThreadLocal<>();
                tl.set(new byte[1024 * 1024]); // 1MB
                //没有remove();
            });
        }
    }
}

```

解决方法：用完就remove，推荐用try-finally

### 跨父子线程传递threadLocal

Thread类里有一个和threadLocals底层一样的InheritableThreadLocals，在创建子线程的时候在线程init方法里会传递该值，逻辑很简单。

```
// Thread 的构造方法会调用 init() 方法
private void init(/* ... */) {
	// 1、获取父线程
    Thread parent = currentThread();
    // 2、将父线程的 inheritableThreadLocals 赋值给子线程
    if (inheritThreadLocals && parent.inheritableThreadLocals != null)
        this.inheritableThreadLocals =
        	ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);
}
```

