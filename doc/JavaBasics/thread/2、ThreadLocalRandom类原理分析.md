#### 1、Random类及其局限性

```java
public int nextInt(int bound) {
    if (bound <= 0)
        throw new IllegalArgumentException(BadBound);
    // 计算新的种子
    int r = next(31);
    int m = bound - 1;
    // 根据新的种子计算随机数
    if ((bound & m) == 0)  // i.e., bound is a power of 2
        r = (int)((bound * (long)r) >> 31);
    else {
        for (int u = r;
             u - (r = u % bound) + m < 0;
             u = next(31))
            ;
    }
    return r;
}
```

```java
protected int next(int bits) {
    long oldseed, nextseed;
    // 这是一个原子性的变量
    AtomicLong seed = this.seed;
    do {
        // (1)、获取老的种子
        oldseed = seed.get();
        // (2)、计算出新的种子
        nextseed = (oldseed * multiplier + addend) & mask;
    // (3)、CAS操作更新老的种子
    } while (!seed.compareAndSet(oldseed, nextseed));
    return (int)(nextseed >>> (48 - bits));
}
```


**Random小结：**

- **面试**：多线程下Random存在什么样的问题？
> 每个Random实例里面都有一个**原子性**的种子变量用来记录当前的种子值，当要生成新的随机数时需要根据当前的种子计算新的种子并更新种子变量。当在多线程环境下，多个线程会竞争同一个原子变量的更新操作，由于原子变量的更新时CAS操作，同时只有一个线程会成功，所以会造成大量线程进行自旋重试，从而降低并发性能。

**可能出现的症状：**
如果并发请求非常多，自旋锁一直重试，那么CPU会一直飙升。

#### 2、ThreadLocalRandom

```java
public static ThreadLocalRandom current() {
    if (UNSAFE.getInt(Thread.currentThread(), PROBE) == 0)
        localInit();
    return instance;
}
```

```java
static final void localInit() {
    int p = probeGenerator.addAndGet(PROBE_INCREMENT);
    int probe = (p == 0) ? 1 : p; // skip 0
    long seed = mix64(seeder.getAndAdd(SEEDER_INCREMENT));
    Thread t = Thread.currentThread();
    UNSAFE.putLong(t, SEED, seed);
    UNSAFE.putInt(t, PROBE, probe);
}
```


这个方法用来创建ThreadLocalRandom随机数生成器，如果当前线程中threadLocalRandomProbe的变量值为0，则说明是第一次调用current方法，那么就调用localInit方法初始化种子变量。

这里使用了延迟初始化，在localInit方法中，并没有初始化种子变量，而是在需要生成随机数的时候再生成种子变量，这是一种优化。


```java
public int nextInt(int bound) {
    if (bound <= 0)
        throw new IllegalArgumentException(BadBound);
    // 生成种子
    int r = mix32(nextSeed());
    int m = bound - 1;
    if ((bound & m) == 0) // power of two
        r &= m;
    else { // reject over-represented candidates
        for (int u = r >>> 1;
             u + m - (r = u % bound) < 0;
             u = mix32(nextSeed()) >>> 1)
            ;
    }
    return r;
}
```

```java
final long nextSeed() {
    Thread t; long r; // read and update per-thread seed
    // 生成新种子（获取当前线程种子 + 种子增量）
    UNSAFE.putLong(t = Thread.currentThread(), SEED,
                   r = UNSAFE.getLong(t, SEED) + GAMMA);
    return r;
}
```

mix32是一个固定的算法，这里重点看下nextSeed方法，当第一次调用的时候进行初始化，获取当前线程threadLocalRandomSeed的值（第一次默认值为0） + 种子增量，如果不是第一次那么获取旧种子的值 + 种子增量生成新的种子变量并设置回去。这样的话多线程环境下就避免了竞争，因为threadLocalRandomSeed是Thread的一个变量，属于线程级别。