___
# 概念
单例模式，是指在任何时候，该类只能被实例化一次，在任何时候，访问该类的对象，对象都是同一个，只有一个
适合用于只需要创建一个实例的对象

___
# 注意事项
在创建单例模式的时候，我们应当注意以下几点：
* 成员变量必须设置为私有变量（十分之基础的问题）
* lazy loading 即延时加载，按需创建
* 是否需要注意线程安全问题，如果有多线程并发，是否会产生多个实例
* 通过反射调用私有构造方法，导致多个实例
* 都需要额外的工作(Serializable、transient、readResolve())来实现序列化，否则每次反序列化一个序列化的对象实例时都会创建一个新的实例。
* 必须将构造器设置为私有方法，否则就不是单例了

synchronized 和 volatile

___
# 实现方式
## 实现一：标准写法
这是最简单的实现，刷刷刷地就写出来了
虽然简单缺点也很明显：
无法延时加载，而是在[[Java/JVM/类加载过程]]的时候就初始化实例了，假如后面都没用到这个单例，那么就白白占用了资源空间
当然，如果确定应用中确实会用到，这种做法还是不错的
```JAVA
public class Singleton {
    private static Singleton singleton = new Singleton();

    /**
     * 注意单例模式的构造函数必须是私有方法
     */
    private Singleton() {

    }

    public static Singleton getInstance() {
        return singleton;
    }
}
```
## 实现二：懒加载
对上面的进行了稍微的改进
解决了懒加载的问题，当需要用到实例的时候才真正去初始化
但是，却导致了个新的问题：
多线程的安全问题，多线程情况下，在实例未初始化时，调用 Singleton.getInstance() 必然会生成多个实例
不过话说回来，如果能够确定在实际应用中不会有多线程访问的话，这种做法还算可以
```JAVA
public class Singleton {
    private static Singleton singleton;

    private Singleton() {

    }
    
    public static Singleton getInstance() {
        if (null == singleton) {
            singleton = new Singleton();
        }
        return singleton;
    }
}
```
## 实现三：同步锁+双重判断
这种写法可以说是相当不错的，通过锁机制和双重判断，集中解决了上面的的实现中出现的两个问题
```JAVA
public class Singleton {
    private volatile static Singleton singleton;

    private Singleton() {

    }

    public static Singleton getInstance() {
        if (null == singleton) {
            synchronized (Singleton.class) {
                if (null == singleton) {
                    singleton = new Singleton();
                }
            }
        }
        return singleton;
    }
}
```
这里不得不说两个地方： volatile 关键字修饰的成员变量和 synchronized 关键字声明的代码块
首先说说 volatile 修饰成员变量，先简单说说这个关键字有什么作用吧（详细的请看 volatile 篇）
1. volatile 关键字具有的两层语义
 * 1）可见性：保证了不同线程对这个变量进行操作时的可见性，即一个线程修改了某个变量的值，这新值对其他线程来说是立即可见的
 * 2）禁止进行指令重排序：在Java内存模型中，允许编译器和处理器对指令进行重排序，但是重排序过程不会影响到单线程程序的执行，却会影响到多线程并发执行的正确性
2. volatile 不具有原子性，并不能保证线程安全
所以讨论下volatile关键字的必要性，如果没有volatile关键字，会怎么样？
在这里问题可能会出在 `singleton = new Singleton();` 这句，在 Java 虚拟机中运行的时候编译成执行指令以后，用伪代码表示
```JAVA
inst = allocat()； // 分配内存
sSingleton = inst;      // 赋值
constructor(inst); // 真正执行构造函数
```
由于虚拟机的优化等导致指令重排了，所以刚好复制操作先执行先赋值了；但还没执行完构造函数，导致其他线程访问得到singleton变量不为null，但初始化还未完成，导致程序崩溃

再说说 synchronized 关键字

## 实现四：静态内部类
不用锁，也可以实现懒加载的且线程安全。通过静态内部类
```JAVA
public class Singleton {
    private Singleton() {

    }

    public static Singleton getInstance() {
        return SingletonCreator.INSTANCE;
    }

    private static class SingletonCreator{
        private static final Singleton INSTANCE = new Singleton();
    }
}
```
只有通过显式调用 getInstance 方法时，才会显式装载 SingletonHolder 类，从而实例化 instance（只有第一次使用这个单例的实例的时候才加载，同时不会有线程安全问题）

## 实现五：枚举类
```JAVA
public enum Singleton {
	INSTANCE;
	public void testMethod() {
	}
}
```
最好的方式，避免了反射、序列化和反序列化