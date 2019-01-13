---
layout: post
title: 死磕Java源码之ThreadLocal实现分析
categories: java进阶
description: 死磕Java源码之ThreadLocal实现分析
keywords: java进阶，ThreadLocal

---

### 死磕Java源码之ThreadLocal实现分析

通俗的讲， `ThreadLocal`是Java里一种特殊的变量。每个线程都有一个ThreadLocalMap，用来存放ThreadLocal变量表，当然这里不是直接通过Map的方式存储，而是通过一个table和Entry结构存储

因为`ThreadLocalMap`变量是跟线程绑定的，所以不存在多线程共享变量之间的并发问题，所以`ThreadLocal`也就是线程安全的变量。

![屏幕快照 2018-09-06 上午11.20.55.png](https://upload-images.jianshu.io/upload_images/1272254-a5d21452f0ecb469.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


具体的结构，在源码部分说明

### ThreadLocal的使用

ThreadLocal主要有以下几个方法：

```java
public T get() { } // 用来获取ThreadLocal在当前线程中保存的变量副本
public void set(T value) { } //set()用来设置当前线程中变量的副本
public void remove() { } //remove()用来移除当前线程中变量的副本
protected T initialValue() { } //initialValue()是一个protected方法，一般是用来在使用时进行重写的
```



写一个demo，在`main`线程和新建线程中，对同一个`ThreadLocal`变量进行修改，看下修改后的结果：

```java
public class ThreadLocalDemo {

    public static ThreadLocal<String> threadLocal = new ThreadLocal<>();

    public static void main(String[] args) {


        ThreadLocalDemo.threadLocal.set("hello world main");
        System.out.println(ThreadLocalDemo.threadLocal.get());


        try {
            Thread thread = new Thread() {
                public void run() {
                    ThreadLocalDemo.threadLocal.set("hello world thread");
                    System.out.println(ThreadLocalDemo.threadLocal.get());
                };
            };
            thread.start();
            thread.join();
        } catch (Exception ex) {
            System.out.println(ex);
        }

        System.out.println(ThreadLocalDemo.threadLocal.get());

    }

}
```

执行输出：

```java
hello world main
hello world thread
hello world main
```

不难看出，我们在`new Thread()`中对`ThreadLocal`的变量`threadLocal`进行修改后，在main线程中再次输出，其值并没有收到影响，他们修改的分别是各自的副本，不会对其他副本有影响。

当然这里完整的逻辑是应该在使用完调用`remove`方法删除`threadLocal`副本，以防内存泄露。

具体原理见下文



### ThreadLocal的内存泄漏与源码分析

ThreadLocal的结构如图：

![image](http://upload-images.jianshu.io/upload_images/1272254-df040bae132e019b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### ThreadLocal为什么会内存泄漏

每个`Thread` 维护一个 `ThreadLocalMap` 映射表，这个映射表的 `key` 是 `ThreadLocal`实例本身，`value` 是真正需要存储的 `Object`。

也就是说 `ThreadLocal` 本身并不存储值，它只是作为一个 `key` 来让线程从 `ThreadLocalMap` 获取 `value`。值得注意的是图中的虚线，表示 `ThreadLocalMap` 是使用 `ThreadLocal` 的弱引用作为 `Key` 的，弱引用的对象在 GC 时会被回收。

`ThreadLocalMap`使用`ThreadLocal`的弱引用作为`key`，如果一个`ThreadLocal`没有外部强引用来引用它，那么系统 GC 的时候，这个`ThreadLocal`势必会被回收，这样一来，`ThreadLocalMap`中就会出现`key`为`null`的`Entry`，就没有办法访问这些`key`为`null`的`Entry`的`value`，如果当前线程再迟迟不结束的话，这些`key`为`null`的`Entry`的`value`就会一直存在一条强引用链：`Thread Ref -> Thread -> ThreaLocalMap -> Entry -> value`永远无法回收，造成内存泄漏。

其实，`ThreadLocalMap`的设计中已经考虑到这种情况，也加上了一些防护措施：在`ThreadLocal`的`get()`,`set()`,`remove()`的时候都会清除线程`ThreadLocalMap`里所有`key`为`null`的`value`，当然这只是一种防护措施，最好的使用方法是在用完了ThreadLocal变量时，调用remove()方法主动将ThreadLocal和value释放。

关于为什么`ThreadLocalMap`使用`ThreadLocal`的弱引用，这就跟弱引用的机制有关，若引用的对象在JVM执行GC的时候就会被回收掉。通过gc前后查看table中对应entry对象的referent即可查看是否被回收（示例的前提是ThreadLocal的强引用对象已经释放）：

GC之前：

![![屏幕快照 2018-09-05 下午5.03.21.png](https://upload-images.jianshu.io/upload_images/1272254-4a2814486c210166.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
](https://upload-images.jianshu.io/upload_images/1272254-f429ce0cc63d4533.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


GC之后：



此时Entry的referent=null，当再次通过调用get、set、remove方法是，ThreadLocal会有各自的机制，将Map中key(referent)为空的Entry移除，并释放其中的value，一定成都避免了内存泄漏，此机制源码分析阶段说明。

#### ThreadLocal源码分析

ThreadLocal中的关键属性

```java
//创建ThreadLocal时，复制的HashCode
//HashCode是在全局静态的nextHashCode基础上增加一个HASH_INCREMENT而来
private final int threadLocalHashCode = nextHashCode();
```

下面分析几个具体的函数：

- get方法的实现

```java
/**
 * Returns the value in the current thread's copy of this
 * thread-local variable.  If the variable has no value for the
 * current thread, it is first initialized to the value returned
 * by an invocation of the {@link #initialValue} method.
 *
 * @return the current thread's value of this thread-local
 */
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

首先是当前线程，然后通过`getMap(t)`方法获取到一个map，map的类型为`ThreadLocalMap`。然后接着下面获取到<key,value>键值对，注意这里获取键值对传进去的是  this，而不是当前线程t。如果获取成功，则返回value值。如果map为空，则调用`setInitialValue`方法返回value。

- setInitialValue方法的实现

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

```
/**
* 构造ThreadLocalMap
**/
ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
    table = new Entry[INITIAL_CAPACITY];
    int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
    table[i] = new Entry(firstKey, firstValue);
    size = 1;
    setThreshold(INITIAL_CAPACITY);
}
```

首先是通过调用`initialValue`，`initialValue`是protected方法，初始化`ThreadLocal`时可以重写此函数，相当于延迟加载，然后通过getMap创建`threadLocals`，如果`threadLocals`不存在时，会调用createMap创建一个初始大小为16的Entry数组table，并新建一个Entry存入table中。这个threadLocals就是用来存储实际的变量副本的，键值为当前`ThreadLocal`变量，value为变量副本（即T类型的变量）



这里重点看下Entry类

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

`Entry`类是集成自`WeakReference`，然后使用ThreadLocal作为了键，也就是说这里的ThreadLocal是一个弱引用在GC的时候会被回收。

接上文，如果map存在，则会调用map的`getEntry`方法，`getEntry`方法实现：

```java
private Entry getEntry(ThreadLocal<?> key) {
    //通过hash算出数组下标
    int i = key.threadLocalHashCode & (table.length - 1);
    Entry e = table[i];
    if (e != null && e.get() == key)
        //如果取出Entry，并且e.get也就是referent与threadLocal相同，则说明是需要的值，返回Entry对象e ，判断e.get() = key 是解决hash碰撞的情况
        return e;
    else
        //如果下标i的Entry不存在或者 其threadLocal不相同，则执行此
        return getEntryAfterMiss(key, i, e);
}


private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
    Entry[] tab = table;
    int len = tab.length;
	
    while (e != null) {
        //说明有此entry，可能是hash碰撞的结果
        ThreadLocal<?> k = e.get();
        if (k == key)
            return e;
        if (k == null)
            //处理已无引用的ThreadLocal变量等，解决内存泄漏的机制之一
            expungeStaleEntry(i);
        else
            //下标+1 
            i = nextIndex(i, len);
        e = tab[i];
    }
    //如果getEntry中获取的entry=null，则说明无此ThreadLocal变量，返回null
    return null;
}
```

expungeStaleEntry 方法

```java
//删除可以释放的Entry
private int expungeStaleEntry(int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;

    // expunge entry at staleSlot
    tab[staleSlot].value = null;
    tab[staleSlot] = null;
    size--;

    // Rehash until we encounter null
    Entry e;
    int i;
    for (i = nextIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = nextIndex(i, len)) {
        ThreadLocal<?> k = e.get();
        if (k == null) {
            //如果发现ThreadLocal已经被释放掉，则通过这里来释放value的引用，以及删除数组table中的Entry
            e.value = null;
            tab[i] = null;
            size--;
        } else {
            int h = k.threadLocalHashCode & (len - 1);
            if (h != i) {
                //重新设置Entry在table中的位置
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

通过对get方法的大致分析，可以分为几个阶段：

1）判断Map是否存在，如果不存在初始化Map以及table等

2）如果已存在，并且获取到Entry，则返回

3）如果不存在，则调用`expungeStaleEntry`清除需要释放的`ThreadLocal`、释放对value的一用，从table中删除相应下标的Entry，以及重新设置元素在table中的位置



- set方法的实现

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



set方法中的createMap与上文的createMap相同，不在做说明，重点看下map.set(this, value);  这里直接在温中锋

```java
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
           //如果通过hash计算的下标取出的entry的key与设置的相同，则更新value
            e.value = value;
            return;
        }

        if (k == null) {
            //和HashMap不一样，由于Entry key继承了软引用，会出现k是null的情况！所以会接着在replaceStaleEntry重新循环寻找相同的key
            replaceStaleEntry(key, value, i);
            return;
        }
    }

    //如果key!= null  并且 k != key 说明存在hash鹏铮
    tab[i] = new Entry(key, value);
    int sz = ++size;
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        //调用cleanSomeSlots()对table进行清理，如果没有任何Entry被清理，并且表的size超过了阈值，就会调用rehash()方
        rehash();
}
```

  hash散列的键值数据在存储过程中可能会发生碰撞，大家知道HashMap存储的是一个Entry链，当hash发生冲突后，将新的Entry存放在链表最前端。但是`ThreadLocalMap`不一样，采用index+1作为重散列的hash值写入。另外有一点需要注意key出现null的原因是由于Entry的key是继承了软引用，在下一次GC时不管它有没有被引用都会被回收掉而Value没有被回收。当出现null时，会调用`replaceStaleEntry()`方法接着循环寻找相同的key，如果存在，直接替换旧值。如果不存在，则在当前位置上重新创建新的Entry.



- remove方法的实现

- ```java
  //ThreadLocal
  public void remove() {
           ThreadLocalMap m = getMap(Thread.currentThread());
           if (m != null)
               m.remove(this);
       }
  /**
   * ThreadeLocalMap
   * Remove the entry for key.
   */
  private void remove(ThreadLocal<?> key) {
      Entry[] tab = table;
      int len = tab.length;
      int i = key.threadLocalHashCode & (len-1);
      for (Entry e = tab[i];
           e != null;
           e = tab[i = nextIndex(i, len)]) {
          if (e.get() == key) {
              e.clear();
              expungeStaleEntry(i);
              return;
          }
      }
  }
  ```

  remove方法相对简单，通过hashcode计算出下标，然后判断key与要删除的ThreadLocal是否一致，如果一致，释放掉相应的引用，并调用expungeStaleEntry方法清理其他的可以释放的对象。


### ThreadLocal的使用场景

- 每个线程自己独享的数据，比如session数据
- 实例需要在多个方法中共享，但不希望被多线程共享

比如在Dubbo中的`RpcContext`实例，在`RpcContext.java`文件中，通过静态的`ThreadLocal`变量，为每个线程持有一个RpcContext对象，这个`RpcContext`对象只有在此线程的不同方法中共享使用，在多线程中不会共享，是一种典型的应用，包括重写了`initialValue`方法

```java
rivate static final ThreadLocal<RpcContext> LOCAL = new ThreadLocal<RpcContext>() {
   @Override
   /**
   * 重新initialValue方法，当get时为null时，通过回调此方法获取RpcContext实例
   **/
   protected RpcContext initialValue() {
      return new RpcContext();
   }
};

/**
 * get context.
 * 
 * @return context
 */
public static RpcContext getContext() {
    return LOCAL.get();
}

/**
 * remove context.
 * 
 * @see com.alibaba.dubbo.rpc.filter.ContextFilter
 */
public static void removeContext() {
    LOCAL.remove();
}
```



### 总结

- `ThreadLocal` 并不解决线程间共享数据的问题，通过使用ThreadLocal是使数据在不同线程有不同的副本，不会有多线程共享数据也就不需要解决共享数据的问题
- 每个线程持有一个 Map 并维护了 `ThreadLocal` 对象与具体实例的映射，该 Map 由于只被持有它的线程访问，故不存在线程安全以及锁的问题
- `ThreadLocalMap` 的 `Entry` 对 `ThreadLocal` 的引用为弱引用，避免了因ThreadLocalMap强引用 ThreadLocal 对象在线程回收之前无法被回收的问题
- `ThreadLocalMap` 的 set 方法通过调用 replaceStaleEntry 方法回收键为 null 的 Entry 对象的值（即为具体实例）以及 Entry 对象本身从而防止内存泄漏
- `ThreadLocalMap` 的 get 方法通过调用 `expungeStaleEntry` 方法回收键为 null 的 Entry 对象的值（即为具体实例）以及 Entry 对象本身从而防止内存泄漏
- `ThreadLocalMap`当hash发生冲突后，并不是与HashMap一样采用的Entry链表将新的Entry存放在链表最前端。而是采用index+1作为重散列的hash值来重新存储Entry值



文章部分自己理解，部分借鉴了大牛们的文章，在此表示感谢！如有bug，劳烦指正！



参考：

<https://blog.csdn.net/liulongling/article/details/50607802>

<http://www.importnew.com/21206.html>

<http://www.jasongj.com/java/threadlocal/>

<http://www.importnew.com/22039.html>

<https://blog.csdn.net/liulongling/article/details/50607802