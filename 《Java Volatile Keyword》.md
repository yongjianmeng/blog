# Java volatile 关键字 #

本文翻译自[《Java Volatile Keyword》](http://tutorials.jenkov.com/java-concurrency/volatile.html)，是我看过的解释**volatile**关键字较为详尽的一篇文章，在此给它做一个译文，也会对文章做出一些修改，并结合一些资料作为补充，希望能把volatile关键字解释地更清楚。

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

## Volatile 可见性保证 ##

volatile关键字被设计的目的就是用来解决可见性问题的。当用volatile关键字修饰一个计数器变量时<sup>[5]</sup>，所有的写操作都会被立即地回写到主存中，同样的，所有的读操作都会直接从主存中读取。

下面这段代码是用来说明如何用volatile修饰一个变量的：
```
public class SharedObject {

    public volatile int counter = 0;

}
```

当声明一个变量为volatile后，它能保证一个线程对这个变量的写操作对其他的线程都是可见的。

在上面的场景中，一个线程（Thread 1）更改计数器的值，另一个线程（Thread 2）读取计数器的值（但从不修改它），当我们把计数器变量用volatile修饰，那么这足以保证T1对计数器的写操作对于T2来说是可见的<sup>[6]</sup>。

当然，如果Thread 1和Thread 2同时修改计数器变量的值，即使把计数器变量声明为volatile也不能保证可见性，后面会详细说明。

### 完全volatile可见性保证 ###

事实上，volatile的可见性保证远不止volatile关键字本身的定义，它的可见性保证如下：

* 如果线程A写入一个volatile变量，随后线程B读取同一个volatile变量，那么在线程A写入该volatile变量**之前**的所有其他变量的可见性，对于线程B读取该volatile变量**之后**是可见的。

* 如果线程A读取一个volatile变量，那么所有其他变量的可见性对于线程A读取volatile变量**之后**是可见的。

是不是都点晦涩难懂？让我用下面的代码来解释说明一下：
```
public class MyClass {
    private int years;
    private int months
    private volatile int days;


    public void update(int years, int months, int days){
        this.years  = years;
        this.months = months;
        this.days   = days;
    }
}
```

update() 方法写入三个变量，其中只有**days**是volatile的。

对于volatile关键字，完全volatile可见性保证意味着：在一个线程中，当一个值被写入days变量时，那么所有对于该线程都是可见的变量也会和days变量一起被写入到主存中。

当我们要读取**years**，**months**和**days**变量时，我们可能会用如下方式读取：

```
public class MyClass {
    private int years;
    private int months
    private volatile int days;

    public int totalDays() {
        int total = this.days;
        total += months * 30;
        total += years * 365;
        return total;
    }

    public void update(int years, int months, int days){
        this.years  = years;
        this.months = months;
        this.days   = days;
    }
}
```

注意到，在totalDays()方法中，对于**days**变量值的读取并赋值给**total**变量是放在方法的最开头，当读取days变量的值时，**months**和**years**变量的值也会从主存中直接读取，因此，如果按照上面的读取顺序，你可以保证拿到**days**，**months**和**years**变量的最新值。

## 指令重排带来的挑战 ##

Java虚拟机和CPU为了提高性能，会对程序中的指令进行重排，只要指令的语义和原来的相同即可。例如，看下面这些指令：

```
int a = 1;
int b = 2;

a++;
b++;
```

这些指令可以被重排为下面这种顺序，它在程序中的语义没有被改变：

```
'
int a = 1;
a++;

int b = 2;
b++;
```

然而，指令重排给其中一个变量是volatile的代码逻辑带来挑战，让我们看看之前提过的MyClass类：

```
public class MyClass {
    private int years;
    private int months
    private volatile int days;


    public void update(int years, int months, int days){
        this.years  = years;
        this.months = months;
        this.days   = days; // -> 把days、months和years变量的值回写到主存
    }
}
```

当update()方法写入一个值到**days**变量时，对于**years**和**months**变量新写入的值也会被回写到主存中。但是，如果Java虚拟机对上述指令进行重排，如下所示，重排后的指令顺序：

```
public void update(int years, int months, int days){
    this.days   = days; // -> 把days、months和years变量的值回写到主存
    this.months = months;
    this.years  = years;
}
```

在**days**变量被修改后，**months**和**years**变量的值还是会被回写到主存，但是这次回写到主存是发生在新的值被赋值给**months**和**years**变量**之前**，也因此，months和years变量的新值对于其他线程是不可见的。此时，指令重排序改变了语义。

Java对于这个问题有解决方案，我们在接下来的章节会看到。

## Volatile Happpends-Before 保证 ##

为了解决指令重排带来的挑战，volatile关键句给出了“happens-before”保证作为可见性保证的补充。“happens-before”保证作出了如下的保证：

* 当读取/写入其他变量的操作原先位于写入一个volatile变量的操作之前，那么读取/写入其他变量的操作就不能被重排到写入这个volatile变量的操作之后。
在写入一个volatile变量之前的所有读取/写入操作能保证“happens-before”于写入该volatile变量的操作。 注意到，有一种情况还是有可能会发生的：位于写入一个volatile变量的操作**之后**的读取/写入其他变量的操作被重排到发生于写入一个volatile变量的操作之前。从**之后**重排到**之前**是允许的，但从**之前**到**之后**是不允许的<sup>[7]</sup>。

* 当读取/写入其他变量的操作原先位于读取一个volatile变量的操作之后，那么读取/写入其他变量的操作就不能被重排到读取这个volatile变量的操作之前。
注意到，有一种情况还是有可能会发生的：位于读取一个volatile变量的操作**之前**的读取其他变量的操作被重排到发生于读取一个volatile变量的操作之后。从**之前**重排到**之后**是允许的，但从**之后**到**之前**是不允许的<sup>[8]</sup>。

上述“happens-before”保证机制确保了volatile关键字的可见性保证是被强制的，即使在有指令重排的挑战下。

### Volatile的局限性 ###

尽管volatile关键字保证了所有的读取操作都直接从主存中读取，所有的写操作都直接被写入到主存中，但在一些场景下，仅仅声明volatile关键字是不够的。

在之前的场景中只有一个线程（Thread 1）会对共享计数器变量进行写操作，把该计数器变量声明为volatile可以确保另一个线程（Thread 2）可以一直看到被Thread 1写入的最新值。

事实上，多个线程可以同时地写入一个共享的volatile变量，并且也能保证主存中存储的是正确的值，但是有个前提，就是写入的新值不依赖于这个共享的volatile变量之前的值，换句话说，如果一个线程写入新值到一个共享的volatile变量之前不需要先读取这个共享的volatile变量的值并推算出新值，那么这样是允许多个线程同时地写入这个共享的volatile变量而不会出错。

一旦线程需要先读取这个共享的volatile变量的值，然后根据这个读取的值去生成新的值，最后再写入这个共享的volatile变量，那么volatile关键字是不能够再对修饰的变量保证其正确的可见性。在读取这个volatile变量的旧值和写入这个变量的新值之间是存在一个很短的时间间隔的，这个很短的时间间隔就会产生竟态条件：多个线程可能就会读取到相同的volatile变量的旧值，然后都根据旧值为这个共享的volatile变量生成新值，最后都在写入操作时回写到主存中，然后它们就会覆写掉对方的值。

多个线程同时增加一个共享的volatile变量的值，这种场景下volatile变量是不能保证可见性的后面的章节会对这个场景做出详细的解释。

想象一下，线程1和线程2同时地（两个不同的CPU，可以并行地执行）从主存中**读取**一个共享的volatile计数器变量到它们各自的CPU缓存中，并各自根据读取的值计算出下一个值（读取的值都是0，所以两个CPU计算出的下一个值都是1）<sup>[9]</sup>，这个场景可以用下图表示：

![](https://github.com/yongjianmeng/blog/blob/master/images/Java%20volatile%20%E5%85%B3%E9%94%AE%E5%AD%97%20-%203.png)

线程1和线程2现在实际上已经不同步了。共享的volatile计数器变量的正确的值应该是2，但是这个变量在两个线程各自的CPU缓存中的值都是1，而主存中这个变量的值仍是0，这已经混乱了！尽管两个线程最后会把它们所持有的这个共享计数器变量的值回写到主存中，但仍然是错误的<sup>[10]</sup>。

## 什么时候可以用volatile ##

就像我之前提及的那样，如果两个线程同时对同一个共享变量做读取和写入操作<sup>[11]</sup>，那么使用volatile变量是不能够保证这个变量的正确性的，你需要在那种情况下使用**synchronized**来保证读取和写入一个变量是原子性的。读取/写入一个volatile变量并不会阻塞其他线程读取/写入。针对这种场景，你必须使用synchronized关键字去把**关键区域**（原文：critical sections）包围住<sup>[12]</sup>。

除了使用synchronized块外，另外一种方法是使用java.util.concurrent 包下其中一个原子数据类型。例如AtomicLong或AtomicReference或其他。

如果只有一个线程读取和写入一个共享的volatile变量而其他线程只读这个变量，那么就能够保证读取线程读到的值是最新被写入的值，如果没有用volatile修饰这个变量，这个就不能够得到保证。

volatile关键字也对32位和64位的关键字作出保证<sup>[13]</sup>。

## Volatile的性能问题 ##

读取/写入volatile变量会导致从主存中读取/写入该变量，从主存中读取/写入比从CPU缓存中访问更消耗性能，访问volatile变量也会阻止指令重排，而指令重排技术的目的是用来提升性能的，因此，你最好只在确实需要用volatile变量保证变量的可见性时使用volatile修饰变量<sup>[14]</sup>。

## 译者注 ##

[1] 关于主存储器、CPU缓存等关系如下表所示：
![](https://github.com/yongjianmeng/blog/blob/master/images/Java%20volatile%20%E5%85%B3%E9%94%AE%E5%AD%97%20-%200.jpg)

[2] 不使用volatile变量之前写操作只写入CPU缓存中，之后再同步到主存中，而使用了volatile变量后，写操作不仅会写入CPU缓存中，也会同时写入到主存中。

[3] 或者是单个CPU多个核。

[4] 图中Thread 1已经把计数器变量的值更新到了7，但是并没有将其同步到主存中，导致同一个计数器变量在CPU缓存中的值与在主存中的值不同，Thread 2只能读到主存中的值，也就是0.

[5] 仅限于只有一个线程会改变这个计数器变量的值，而其他的线程都只有读取的操作，文章后面会有说明。

[6] 因为Thread 1和Thread 2都操作计数器变量，当Thread 1修改计数器的值，计数器的最新值会被立刻从CPU缓存中写入到主存里，而当Thread 2去读取计数器的值时，它会直接从主存中读取（而不是从CPU缓存中读取），所以可见性得到了保证。

[7] 这里的**之前**和**之后**是以前面说的**写入一个volatile变量的操作**作为界限，**读取/写入其他变量的操作**如果发生于**写入一个volatile变量的操作**之前，那么**读取/写入其他变量的操作**不允许被重排到**写入一个volatile变量的操作**之后。**读取/写入其他变量的操作**如果发生于**写入一个volatile变量的操作**之后，那么**读取/写入其他变量的操作**就允许被重排到**写入一个volatile变量的操作**之前。我们用下面的代码说明：
```
public class Test {
	private volatile int x;
	private int y;
	private int z;

	public void test() {
		int m = this.y; // -> 读取其他变量的操作
		this.z = 1;     // -> 写入其他变量的操作
		this.x = 1;     // -> 写入一个volatile变量的操作
	}
}
```
上述代码**读取/写入其他变量的操作**发生于**写入一个volatile变量的操作**之前，所以读取**y**和写入**z**这两个操作是不能被重排到写入**x**操作之后的，例如，下面的重排序是**不允许**的：
```
public void test() {
	this.x = 1;     // -> 写入一个volatile变量的操作
	int m = this.y; // -> 读取其他变量的操作（被重排序）
	this.z = 1;     // -> 写入其他变量的操作（被重排序）
}
```
再看个允许重排的例子：
```
public class Test {
	private volatile int x;
	private int y;
	private int z;

	public void test() {
		this.x = 1;     // -> 写入一个volatile变量的操作
		int m = this.y; // -> 读取其他变量的操作
		this.z = 1;     // -> 写入其他变量的操作
	}
}
```
上述代码**读取/写入其他变量的操作**发生于**写入一个volatile变量的操作**之后，所以读取**y**和写入**z**这两个操作是允许被重排到写入**x**操作之前的，例如，下面的重排序是**允许**的：
```
public void test() {
	int m = this.y; // -> 读取其他变量的操作
	this.z = 1;     // -> 写入其他变量的操作
	this.x = 1;     // -> 写入一个volatile变量的操作
}
```

[8] 和[7]类似，这里的**之前**和**之后**是以前面说的**读取一个volatile变量的操作**作为界限，**读取/写入其他变量的操作**如果发生于**读取一个volatile变量的操作**之后，那么**读取/写入其他变量的操作**不允许被重排到**读取一个volatile变量的操作**之前。**读取其他变量的操作**如果发生于**读取一个volatile变量的操作**之前，那么**读取其他变量的操作**就允许被重排到**读取一个volatile变量的操作**之后。

[9] 这里原文写得不是很正确，所以我按照自己的理解写出它想表达的意思，原文如下：
Imagine if Thread 1 reads a shared counter variable with the value 0 into its CPU cache, increment it to 1 and not write the changed value back into main memory. Thread 2 could then read the same counter variable from main memory where the value of the variable is still 0, into its own CPU cache. Thread 2 could then also increment the counter to 1, and also not write it back to main memory. 

[10] 之前说的很短的时间间隔就是在**读取**一个共享的volatile变量的旧值和**写入**一个共享的volatile变量的新值之间，如果多个线程同时做这两个操作，就会因为这个时间间隔而产生**竟态条件**，也就是我们描述的场景那样，两个线程同时地读取了一个共享的volatile变量的值，并根据读到的值计算出新值，最后再回写到主存中，这时候结果就是错误的，谁都不能保证在这个事件间隔内，其他线程会对同一个共享的volatile变量做出什么事，即使这个时间间隔很短。

[11] 这里还需要有一个前提条件，就是两个线程同时地读取，**并根据读取到的值计算出新值**，并回写新值到主存中。

[12] 其实不一定使用synchronized关键字，使用原子变量也是可以的，或者其他锁机制也可以。

[13] Java虚拟机规范定义：**所有对基本类型的操作，除了某些对long类型和double类型的操作之外，都必须是原子级的**。由于规范没有规定如何实现，那么当今所知的虚拟机对这条规则的实现都是把32位值做为原子性对待，而不是64位做为原子性。那么，当线程把主存中的long/double类型的值读到线程内存中时，可能是两次32位值的写操作，显而易见，如果几个线程同时操作，那么就可能会出现高低2个32位值出错的情况发生，所以现在，java程序必须确保通过同步来操作共享的long和double。

[14] 在当前大多数处理器架构上，读取volatile变量的开销只比读取非volatile变量的开销略高一些。

> 欢迎转载，注明出处即可。
