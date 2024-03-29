在开始文章之前，我这里暂且认为大家已经明白了JVM创建对象分配内存地址的流程，也知道JVM内存划分。基于人道主义我还是放一张图吧，大家对照着看。

<br/>

<div style="text-align:center; font-weight:bold;">JVM内存结构</div>

![](https://files.mdnice.com/user/2735/aecf71bc-dc8c-4097-bb67-ce4eabd7a32d.png)

<br/>

<div style="text-align:center; font-weight:bold;">堆内存划分结构</div>

![](https://files.mdnice.com/user/2735/c508f260-fb6a-4cb3-aa17-aa5f1adabcb2.png)

## 堆区分配内存是否存在多线程安全问题？

答：**可能存在**；

new Object();

上述操作我们都知道它最终需要在堆内存中开辟一块内存空间，那么想这么一个问题，堆区是所有线程共享的，那么在JVM频繁创建对象的时候，并发情况下在堆内存中开辟空间是不是存在安全问题。

那么为了解决这个问题我们首先想到的就是加锁，但是加锁存在一个问题，就是影响性能。

## TLAB出现（Thread Local Allocation Buffer）

基于上面的问题，从而引出了TLAB，强行翻译一下就是**线程本地分配缓冲区**，首先呢先看张图

![](https://files.mdnice.com/user/2735/6162e4f7-c246-4532-88a9-e2745ffdf9b3.png)

**声明**：在堆内存中分配空间，首先是在eden区进行分配，并不是直接分配在老年代，内存分配结束之后，没进行一次Yong GC，如果对象没有被回收，那么他的存活次数就会 +1，如果这个次数达到15次，那么这个对象晋升到老年代。


那么我们知道了对象分配首先是在eden区进行的，那么也不难理解上面的图，我们在eden区域划分出来一块区域，我们称之为TLAB，每一个TLAB都是现成私有的，那么并发创建对象的时候其实也就不需要进行加锁这样的操作了，这样现成安全问题就解决了。

如果分配的这些TLAB空间被使用完了或者对象所需要额内存空间大于TLAB所能提供的空间，那么只能在公用的eden区或者老年代分配内存空间了。

## 总结

- 1、JVM首选TLAB进行内存空间的分配；
- 2、TLAB占用整个eden区域的1%，这个值也可以通过参数自定义；

通过这个问题也可以推理出另外一个问题，**堆区在严格意义上说不是线程共享的**。