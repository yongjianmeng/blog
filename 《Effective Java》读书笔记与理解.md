# 《Effective Java》读书笔记与理解 - 持续更新 #

## 创建与销毁对象 ##

### 1. 考虑用静态工厂方法替代构造器 ###

* 静态工厂方法与构造器不同的第一大优势在于，**它们有名称**。

 **构造器方式**
```
// Constructs a randomly generated positive BigInteger that is probably prime, with the specified bitLength.
public BigInteger(int bitLength,
                  int certainty,
                  Random rnd)
```

 **静态工厂方法方式**
```
// Returns a positive BigInteger that is probably prime, with the specified bitLength. 
public static BigInteger probablePrime(int bitLength,
                                       Random rnd)
```

 两段代码做的事情相同，但显然静态工厂方法更能清楚地表达意图，更容易让用户记住，而不需要进一步去参考文档。

 **参考设计模式**：工厂方法

* 静态工厂方法与构造器不同的第二大优势在于，**不必在每次调用它们的时候都创建一个新对象**。
 **举例**
```
public static Boolean valueOf(boolean b) {
    return b ? Boolean.TRUE : Boolean.FALSE;
}
```
Boolean.valueOf(boolean)方法从来不创建对象。

 **参考设计模式**：单例模式、享元模式

* 静态工厂方法与构造器不同的第三大优势在于，**它们可以返回原返回类型的任何子类型的对象**。
 **举例**：
	- Java Collections Framework
	- EnumSet（RegalarEnumSet、JumboEnumSet）
	- Service Provider Framework  - JDBC

* 静态工厂方法与构造器不同的第三大优势在于，**在创建参数化类型实例的时候，它们使代码变得更加简单**。

 **构造器方式举例**
```
Map<String, List<String>> m = new HashMap<String, List<String>>();
```
 **静态工厂方法方式举例**
```
public static <K, V> HashMap<K, V> newInstance() {
    return new HashMap<K, V>();
}
Map<String, List<String>> m = HashMap.newInstance();
```
上面的静态工厂方法方式举例只是作者的猜想，现版本的Java（1.8）中并没有这个方法，但是自Java 1.7之后，我们可以以更简便的方式来构造：
```
Map<String, List<String>> m = new HashMap<>();
```
上面语句会在语言级别进行**类型推导**。

* 静态工厂方法的主要缺点在于，**类如果不含有公有的或受保护的构造器，就不能被子类化**。  
其实也不能算是缺点，因为我们建议使用复合（Composition），而不是继承。

* 静态工厂方法的第二个缺点在于，**它们与其他的静态方法实际上没有任何区别**。  
在文档中，它们没有像构造器那样在文档中明确标识出来。
**静态工厂方法惯用名称**：**valueOf**（类型转换方法）、**of**（valueOf的一种简洁的替代）、**getInstance**（每次返回单例）、**newInstance**（每次返回新实例）、**getType**（像getInstance一样，每次返回单例的类型Type），**newType**（像newInstance一样，每次返回不一样的类型Type）。

----------
## 2. 遇到多个构造器参数时要考虑用构建器 ##

* 静态工厂方法和构造器有个共同的局限性：它们都不能很好地扩展到大量的可选参数。

* 重叠构造器模式不适用大量的可选参数：客户端代码会很难编写，并且容易出错，有时编译器也不会报错，结果造成意想不到的错误。

* JavaBean模式弥补了重叠构造器的不足，但仍然有一些缺点：
	- 构造过程被分到了几个调用中，在构造过程中JavaBean可能处于不一致状态。
	- JavaBean模式阻止了把类做成不可变的可能。   

* 如果类的构造器或者静态工厂方法中具有多个参数，设计这种类时，Builder模式就是种不错的选择，特别是当大多数参数都是可选的时候。与传统的重叠构造器模式相比，使用Builder模式的客户端代码将易于阅读和编写，构建器也比JavaBeans更加安全。 

----------
## 3. 用私有构造器或者枚举类型强化Singleton属性 ##

* 用私有构造器强化Singleton属性
```
// Singleton with public final field
public class Elvis {
    public static final Elvis INSTANCE = new Elvis();
    private Elvis() { ... }

    // ...
}
```
享有特权的客户端可以借助AccessibleObject.setAccessible方法，通过反射机制调用私有构造器，可以通过修改构造器，让它在被要求创建第二个实例的时候抛出异常。

* 公有的成员是个静态工厂方法：
```
// Singleton with static factory
public class Elvis {
    private static final Elvis INSTANCE = new Elvis();
    private Elvis() { ... }
    public static Elvis getInstance() { return INSTANCE; }

    // ...
}
```
公有域方法的主要好处在于，**组成类的成员的声明很清楚地表明了这个类是一个Singleton**。现代的JVM实现几乎能够将静态工厂方法的调用内联化。静态工厂方法更加灵活，在不改变其API的前提下，可以改变该类是否应该为Singleton的想法。为了维护并保证Singleton，必须声明所有的实例都是瞬时（**transient**）**的并提供一个readResolve方法**，否则，每次反序列化一个序列化的实例时，都会创建一个新的实例。
```
// readResolve method to preserve singleton property
private Object readResolve() {
	// Return the one true Elvis and let the garbage collector take care of the Elvis impersonator.
    return INSTANCE;
}
```
* 包含单个元素的枚举类型实现单例模式
```
public enum Elvis {
	INSTANCE;

    // ...
}
```
这种方法在功能上与 公有域方法相近，但是它更加简洁，**无偿地提供了序列化机制**，**绝对防止多次实例化**，**即使是在面对复杂的序列化或者反射攻击的时候**。单元素的枚举类型已经成为实现Singleton的最佳方法。

----------
## 4. 通过私有构造器强化不可实例化的能力 ##

* 通过私有构造器强化不可实例化的能力
```
// Noninstantiable utility class
public class UtilityClass {
    // Suppress default constructor for noninstantiability
    private UtilityClass() {
        throw new AssertionError();
    }
    // ...
}
```
**副作用**：使得该类无法被子类化。

----------
## 5. 避免创建不必要的对象 ##

----------
## 6. 消除过期的对象引用 ##

----------
## 7. 避免使用终结方法 ##

----------
## 8. 覆盖equals时请遵守通用约定 ##

----------
## 9. 覆盖equals时总要覆盖hashCode ##

----------
## 10. 始终要覆盖toString ##

----------
## 11. 谨慎地覆盖clone ##

----------
## 12. 考虑实现Comparable接口 ##

----------
## 13. 使类和成员地可访问性最小化 ##

----------
## 14. 在公有类中使用访问方法而非公有域 ##

----------
## 15. 使可变性最小化 ##

----------
## 16. 复合优先于继承 ##

----------
## 17. 要么为继承而设计，并提供文档说明，要么禁止继承 ##

----------