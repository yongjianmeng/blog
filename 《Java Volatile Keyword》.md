# Java volatile 关键字 #

本文翻译自[《Java Volatile Keyword》](http://tutorials.jenkov.com/java-concurrency/volatile.html)，是我看过的解释Java中**volatile**关键字较为详尽的一篇文章，在此做给它做一个译文，并结合一些其他资料作为补充，可能也会对文章做出一些修改，最终希望能把volatile关键字解释地更清楚。

在Java中，volatile关键字的作用是用于标记一个变量为“存储在主存储器（主存）中”，更确切地说，每一次读取volatile变量都是从主存中读取而不是从CPU缓存<sup>[1]</sup>中读取，每一次写入volatile变量都会写入主存中而不是只写入CPU缓存中<sup>[2]</sup>。事实上，自JDK 1.5之后，volatile关键字不仅仅能保证被标记的变量的读取和写入操作能直接作用于主存，它还有更多的功能，我会在后面的文章做出解释。


## 变量可见性问题 ##

在Java中，volatile关键字保证在一个线程上修改一个变量的操作对于其它线程是可见的（原文：guarantees visibility of changes to variables across threads），这可能听上去有一些抽象，那就让我更详细地解释一下。

在一个多线程的应用中，当要用多个线程去操作一个非volatile的共享变量时，每个线程都会先从主存中拷贝变量到CPU缓存中，然后再在CPU缓存中操作该共享变量，这样做能提高性能。如果你的电脑包含有多个CPU<sup>[3]</sup>，并且每个线程在不同的CPU上执行代码，这也就意味着，每个线程都会拷贝共享变量到各自的CPU缓存中，如下图所示：

![](https://github.com/yongjianmeng/blog/blob/master/images/Java%20volatile%20%E5%85%B3%E9%94%AE%E5%AD%97%20-%201.png)

使用非volatile共享变量就不能保证Java虚拟机在何时会从主存中读取数据到CPU缓存中，或者何时会把CPU缓存中的数据写入到主存中，这将会导致一些问题，在后面的章节我会做出解释。

想象一个场景，我们使用两个线程去访问一个共享对象，这个共享对象包含一个计数器变量，在Java中它的声明如下：
```
public class SharedObject {

    public int counter = 0;

}
```
现在我们用其中一个线程（Thread 1）去增加计数器变量的值，另一个线程（Thread 2）会时不时地去读取这个计数器变量的值。

如果这个计数器变量没有声明为volatile，那么就无法保证计数器变量的值何时会被从CPU缓存中同步（写入、回写）到主存中，也就是说，计数器变量在CPU缓存中的值与在主存中的值不同，如下图所示<sup>[4]</sup>：

![](https://github.com/yongjianmeng/blog/blob/master/images/Java%20volatile%20%E5%85%B3%E9%94%AE%E5%AD%97%20-%202.png)

这里存在的问题是Thread 2没有看到计数器变量最新的值，即使Thread 1已经把它更新到了7，但由于Thread 1没有把它同步（写入、回写）到主存中，所以Thread 1对计数器变量的修改对Thread 2是不可见的，这也就是我们要说的“可见性”问题：一个线程对共享变量做的更新操作对其他线程是不可见的。

## volatile 可见性保证 ##

### 完全volatile可见性保证 ###
TODO change a name ?

## 指令重排带来的挑战 ##

## volatile Happpends-Before 保证 ##

### volatile的局限性 ###

## 什么时候可以用volatile ##

## volatile的性能问题 ##

## 译者注 ##

[1] 关于主存储器、CPU缓存等关系如下表所示：
![](https://github.com/yongjianmeng/blog/blob/master/images/Java%20volatile%20%E5%85%B3%E9%94%AE%E5%AD%97%20-%200.jpg)

[2] 不使用volatile变量之前写操作只写入CPU缓存中，之后再同步到主存中，而使用了volatile变量后，写操作不仅会写入CPU缓存中，也会同时写入到主存中。
[3] 或者是单个CPU多个核。
[4] 图中Thread 1已经把计数器变量的值更新到了7，但是并没有将其同步到主存中，导致同一个计数器变量在CPU缓存中的值与在主存中的值不同，Thread 2只能读到主存中的值，也就是0.
[5] 
