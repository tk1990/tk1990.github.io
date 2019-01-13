---
layout: post
title: 使用Guava RateLimiter限流以及源码解析
categories: java进阶
description: 使用Guava RateLimiter限流以及源码解析
keywords: java进阶，Guava,RateLimiter

---

## 前言

在开发高并发系统时有三把利器用来保护系统：缓存、降级和限流

- `缓存` 缓存的目的是提升系统访问速度和增大系统处理容量
- `降级` 降级是当服务出现问题或者影响到核心流程时，需要暂时屏蔽掉，待高峰或者问题解决后再打开
- `限流` 限流的目的是通过对并发访问/请求进行限速，或者对一个时间窗口内的请求进行限速来保护系统，一旦达到限制速率则可以拒绝服务、排队或等待、降级等处理



## 常用的限流算法

### 漏桶算法

> 漏桶算法思路很简单，水（请求）先进入到漏桶里，漏桶以一定的速度出水，当水流入速度过大会直接溢出，可以看出漏桶算法能强行限制数据的传输速率。

![img](http://images.cnitblog.com/blog/522490/201411/081225378155003.png)

### 令牌桶算法

>   对于很多应用场景来说，除了要求能够限制数据的平均传输速率外，还要求允许某种程度的突发传输。这时候漏桶算法可能就不合适了，令牌桶算法更为适合。如图所示，令牌桶算法的原理是系统会以一个恒定的速度往桶里放入令牌，而如果请求需要被处理，则需要先从桶里获取一个令牌，当桶里没有令牌可取时，则拒绝服务。

![img](http://images.cnitblog.com/blog/522490/201411/081226107372877.png)



## RateLimiter使用以及源码解析

> Google开源工具包Guava提供了限流工具类RateLimiter，该类基于令牌桶算法实现流量限制，使用十分方便，而且十分高效。

### RateLimiter使用

首先简单介绍下RateLimiter的使用，

```java
public void testAcquire() {
      RateLimiter limiter = RateLimiter.create(1);

      for(int i = 1; i < 10; i = i + 2 ) {
          double waitTime = limiter.acquire(i);
          System.out.println("cutTime=" + System.currentTimeMillis() + " acq:" + i + " waitTime:" + waitTime);
      }
  }
```

输出结果：

```shell
cutTime=1535439657427 acq:1 waitTime:0.0
cutTime=1535439658431 acq:3 waitTime:0.997045
cutTime=1535439661429 acq:5 waitTime:2.993028
cutTime=1535439666426 acq:7 waitTime:4.995625
cutTime=1535439673426 acq:9 waitTime:6.999223
```

首先通过`RateLimiter.create(1);`创建一个限流器，参数代表每秒生成的令牌数，通过`limiter.acquire(i);`来以阻塞的方式获取令牌，当然也可以通过`tryAcquire(int permits, long timeout, TimeUnit unit)`来设置等待超时时间的方式获取令牌，如果超timeout为0，则代表非阻塞，获取不到立即返回。



从输出来看，RateLimiter支持预消费，比如在acquire(5)时，等待时间是3秒，是上一个获取令牌时预消费了3个两排，固需要等待3*1秒，然后又预消费了5个令牌，以此类推

**RateLimiter通过限制后面请求的等待时间，来支持一定程度的突发请求(预消费)**，在使用过程中需要注意这一点，具体实现原理后面再分析。



### RateLimiter实现原理

> Guava有两种限流模式，一种为稳定模式(SmoothBursty:令牌生成速度恒定)，一种为渐进模式(SmoothWarmingUp:令牌生成速度缓慢提升直到维持在一个稳定值) 两种模式实现思路类似，主要区别在等待时间的计算上，本篇重点介绍SmoothBursty



##### RateLimiter的创建

通过调用RateLimiter的`create`接口来创建实例，实际是调用的`SmoothBuisty`稳定模式创建的实例。

```java
public static RateLimiter create(double permitsPerSecond) {
    return create(permitsPerSecond, SleepingStopwatch.createFromSystemTimer());
  }

  static RateLimiter create(double permitsPerSecond, SleepingStopwatch stopwatch) {
    RateLimiter rateLimiter = new SmoothBursty(stopwatch, 1.0 /* maxBurstSeconds */);
    rateLimiter.setRate(permitsPerSecond);
    return rateLimiter;
  }
```

`SmoothBursty`中的两个构造参数含义：

- SleepingStopwatch：guava中的一个时钟类实例，会通过这个来计算时间及令牌
- maxBurstSeconds：官方解释，在ReteLimiter未使用时，最多保存几秒的令牌，默认是1

在解析SmoothBursty原理前，重点解释下SmoothBursty中几个属性的含义

```java
/**
 * The work (permits) of how many seconds can be saved up if this RateLimiter is unused?
 * 在RateLimiter未使用时，最多存储几秒的令牌
 * */
 final double maxBurstSeconds;
 

/**
 * The currently stored permits.
 * 当前存储令牌数
 */
double storedPermits;

/**
 * The maximum number of stored permits.
 * 最大存储令牌数 = maxBurstSeconds * stableIntervalMicros(见下文)
 */
double maxPermits;

/**
 * The interval between two unit requests, at our stable rate. E.g., a stable rate of 5 permits
 * per second has a stable interval of 200ms.
 * 添加令牌时间间隔 = SECONDS.toMicros(1L) / permitsPerSecond；(1秒/每秒的令牌数)
 */
double stableIntervalMicros;

/**
 * The time when the next request (no matter its size) will be granted. After granting a request,
 * this is pushed further in the future. Large requests push this further than small requests.
 * 下一次请求可以获取令牌的起始时间
 * 由于RateLimiter允许预消费，上次请求预消费令牌后
 * 下次请求需要等待相应的时间到nextFreeTicketMicros时刻才可以获取令牌
 */
private long nextFreeTicketMicros = 0L; // could be either in the past or future
```

接下来介绍几个关键函数

- setRate

```java
public final void setRate(double permitsPerSecond) {
  checkArgument(
      permitsPerSecond > 0.0 && !Double.isNaN(permitsPerSecond), "rate must be positive");
  synchronized (mutex()) {
    doSetRate(permitsPerSecond, stopwatch.readMicros());
  }
}
```

通过这个接口设置令牌通每秒生成令牌的数量，内部时间通过调用`SmoothRateLimiter`的`doSetRate`来实现



- doSetRate

```java
@Override
  final void doSetRate(double permitsPerSecond, long nowMicros) {
    resync(nowMicros);
    double stableIntervalMicros = SECONDS.toMicros(1L) / permitsPerSecond;
    this.stableIntervalMicros = stableIntervalMicros;
    doSetRate(permitsPerSecond, stableIntervalMicros);
  }
```

这里先通过调用`resync`生成令牌以及更新下一期令牌生成时间，然后更新stableIntervalMicros，最后又调用了`SmoothBursty`的`doSetRate`

- resync

```
/**
 * Updates {@code storedPermits} and {@code nextFreeTicketMicros} based on the current time.
 * 基于当前时间，更新下一次请求令牌的时间，以及当前存储的令牌(可以理解为生成令牌)
 */
void resync(long nowMicros) {
    // if nextFreeTicket is in the past, resync to now
    if (nowMicros > nextFreeTicketMicros) {
      double newPermits = (nowMicros - nextFreeTicketMicros) / coolDownIntervalMicros();
      storedPermits = min(maxPermits, storedPermits + newPermits);
      nextFreeTicketMicros = nowMicros;
    }
}
```

根据令牌桶算法，桶中的令牌是持续生成存放的，有请求时需要先从桶中拿到令牌才能开始执行，谁来持续生成令牌存放呢？

一种解法是，开启一个定时任务，由定时任务持续生成令牌。这样的问题在于会极大的消耗系统资源，如，某接口需要分别对每个用户做访问频率限制，假设系统中存在6W用户，则至多需要开启6W个定时任务来维持每个桶中的令牌数，这样的开销是巨大的。

另一种解法则是延迟计算，如上`resync`函数。该函数会在每次获取令牌之前调用，其实现思路为，若当前时间晚于nextFreeTicketMicros，则计算该段时间内可以生成多少令牌，将生成的令牌加入令牌桶中并更新数据。这样一来，只需要在获取令牌时计算一次即可。



- SmoothBursty的doSetRate

```java
@Override
void doSetRate(double permitsPerSecond, double stableIntervalMicros) {
  double oldMaxPermits = this.maxPermits;
  maxPermits = maxBurstSeconds * permitsPerSecond;
  if (oldMaxPermits == Double.POSITIVE_INFINITY) {
    // if we don't special-case this, we would get storedPermits == NaN, below
    // Double.POSITIVE_INFINITY 代表无穷啊
    storedPermits = maxPermits;
  } else {
    storedPermits =
        (oldMaxPermits == 0.0)
            ? 0.0 // initial state
            : storedPermits * maxPermits / oldMaxPermits;
  }
}
```



桶中可存放的最大令牌数由maxBurstSeconds计算而来，其含义为最大存储maxBurstSeconds秒生成的令牌。
该参数的作用在于，可以更为灵活地控制流量。如，某些接口限制为300次/20秒，某些接口限制为50次/45秒等。也就是流量不局限于qps



##### RateLimiter几个常用接口分析

在了解以上概念后，就非常容易理解RateLimiter暴露出来的接口

```
@CanIgnoreReturnValue
public double acquire() {
  return acquire(1);
}

/**
* 获取令牌，返回阻塞的时间
**/
@CanIgnoreReturnValue
public double acquire(int permits) {
  long microsToWait = reserve(permits);
  stopwatch.sleepMicrosUninterruptibly(microsToWait);
  return 1.0 * microsToWait / SECONDS.toMicros(1L);
}

final long reserve(int permits) {
  checkPermits(permits);
  synchronized (mutex()) {
    return reserveAndGetWaitLength(permits, stopwatch.readMicros());
  }
}
```

`acquire`函数主要用于获取permits个令牌，并计算需要等待多长时间，进而挂起等待，并将该值返回，主要通过`reserve`返回需要等待的时间，`reserve`中通过调用`reserveAndGetWaitLength`获取等待时间

```java
/**
 * Reserves next ticket and returns the wait time that the caller must wait for.
 *
 * @return the required wait time, never negative
 */
final long reserveAndGetWaitLength(int permits, long nowMicros) {
  long momentAvailable = reserveEarliestAvailable(permits, nowMicros);
  return max(momentAvailable - nowMicros, 0);
}
```

最后调用了reserveEarliestAvailable

```java
@Override
final long reserveEarliestAvailable(int requiredPermits, long nowMicros) {
  resync(nowMicros);
  long returnValue = nextFreeTicketMicros;
  double storedPermitsToSpend = min(requiredPermits, this.storedPermits);
  double freshPermits = requiredPermits - storedPermitsToSpend;
  long waitMicros =
      storedPermitsToWaitTime(this.storedPermits, storedPermitsToSpend)
          + (long) (freshPermits * stableIntervalMicros);

  this.nextFreeTicketMicros = LongMath.saturatedAdd(nextFreeTicketMicros, waitMicros);
  this.storedPermits -= storedPermitsToSpend;
  return returnValue;
}
```

	首先通过resync生成令牌以及同步nextFreeTicketMicros时间戳，freshPermits从令牌桶中获取令牌后还需要的令牌数量，通过storedPermitsToWaitTime计算出获取freshPermits还需要等待的时间，在稳定模式中，这里就是(long) (freshPermits * stableIntervalMicros) ，然后更新nextFreeTicketMicros以及storedPermits，这次获取令牌需要的等待到的时间点， reserveAndGetWaitLength返回需要等待的时间间隔。
	
	从`reserveEarliestAvailable`可以看出RateLimiter的预消费原理，以及获取令牌的等待时间时间原理（可以解释示例结果），再获取令牌不足时，并没有等待到令牌全部生成，而是更新了下次获取令牌时的nextFreeTicketMicros，从而影响的是下次获取令牌的等待时间。



     `reserve`这里返回等待时间后，`acquire`通过调用`stopwatch.sleepMicrosUninterruptibly(microsToWait);`进行sleep操作，这里不同于Thread.sleep(), 这个函数的sleep是uninterruptibly的，内部实现：

```java
public static void sleepUninterruptibly(long sleepFor, TimeUnit unit) {
    //sleep 阻塞线程 内部通过Thread.sleep()
  boolean interrupted = false;
  try {
    long remainingNanos = unit.toNanos(sleepFor);
    long end = System.nanoTime() + remainingNanos;
    while (true) {
      try {
        // TimeUnit.sleep() treats negative timeouts just like zero.
        NANOSECONDS.sleep(remainingNanos);
        return;
      } catch (InterruptedException e) {
        interrupted = true;
        remainingNanos = end - System.nanoTime();
        //如果被interrupt可以继续，更新sleep时间，循环继续sleep
      }
    }
  } finally {
    if (interrupted) {
      Thread.currentThread().interrupt();
      //如果被打断过，sleep过后再真正中断线程
    }
  }
}
```

 	sleep之后，`acquire`返回sleep的时间，阻塞结束，获取到令牌。



------

```
public boolean tryAcquire(int permits) {
  return tryAcquire(permits, 0, MICROSECONDS);
}

public boolean tryAcquire() {
  return tryAcquire(1, 0, MICROSECONDS);
}

public boolean tryAcquire(int permits, long timeout, TimeUnit unit) {
  long timeoutMicros = max(unit.toMicros(timeout), 0);
  checkPermits(permits);
  long microsToWait;
  synchronized (mutex()) {
    long nowMicros = stopwatch.readMicros();
    if (!canAcquire(nowMicros, timeoutMicros)) {
      return false;
    } else {
      microsToWait = reserveAndGetWaitLength(permits, nowMicros);
    }
  }
  stopwatch.sleepMicrosUninterruptibly(microsToWait);
  return true;
}

private boolean canAcquire(long nowMicros, long timeoutMicros) {
  return queryEarliestAvailable(nowMicros) - timeoutMicros <= nowMicros;
}

@Override
final long queryEarliestAvailable(long nowMicros) {
  return nextFreeTicketMicros;
}
```

`tryAcquire`函数可以尝试在timeout时间内获取令牌，如果可以则挂起等待相应时间并返回true，否则立即返回false
`canAcquire`用于判断timeout时间内是否可以获取令牌，通过判断当前时间+超时时间是否大于nextFreeTicketMicros 来决定是否能够拿到足够的令牌数，如果可以获取到，则过程同acquire，线程sleep等待，如果通过`canAcquire`在此超时时间内不能回去到令牌，则可以快速返回，不需要等待timeout后才知道能否获取到令牌。



因为SmoothBursty允许一定程度的突发，会有人担心如果允许这种突发，假设突然间来了很大的流量，那么系统很可能扛不住这种突发。因此需要一种平滑速率的限流工具，从而系统冷启动后慢慢的趋于平均固定速率（即刚开始速率小一些，然后慢慢趋于我们设置的固定速率）。Guava也提供了SmoothWarmingUp来实现这种需求，其可以认为是漏桶算法，但是在某些特殊场景又不太一样。

   

SmoothWarmingUp创建方式：

```java
public static RateLimiter create(double permitsPerSecond, long warmupPeriod, TimeUnit unit) {
  checkArgument(warmupPeriod >= 0, "warmupPeriod must not be negative: %s", warmupPeriod);
  return create(
      permitsPerSecond, warmupPeriod, unit, 3.0, SleepingStopwatch.createFromSystemTimer());
}
```

permitsPerSecond表示每秒新增的令牌数，warmupPeriod表示在从冷启动速率过渡到平均速率的时间间隔，大致原理是类似的，这里就先不分析了。



到此，Guava RateLimiter稳定模式的实现原理基本已经清楚，如发现文中错误的地方，劳烦指正！



上述分析主要参考了：https://segmentfault.com/a/1190000012875897，再此基础上做了些笔记补充