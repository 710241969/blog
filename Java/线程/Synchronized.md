# Synchronized 关键字

## 作用
synchronized 关键字保证在多线程环境中，只有一个线程可以执行某个方法或者某个代码块，同时还有另一个作用就是保证可见性（一个线程操作的共享数据变化，可以被其他线程看到，也就是 volatile 的保证可见性功能）

## 三种使用方式
synchronized 关键字的使用方式总共就以下三种：
1. 修饰代码块
2. 修饰静态方法（类方法）
3. 修饰实例方法

## 原理

### java对象头和 monitor
![[Pasted image 20230519011028.png]]
在 JVM 中，对象在内存中分为三块区域：
* 对象头
    * Mark Word（标记字段）：默认存储对象的HashCode，分代年龄和锁标志位信息。它会根据对象的状态复用自己的存储空间，也就是说在运行期间Mark Word里存储的数据会随着锁标志位的变化而变化。mark word的位长度为JVM的一个Word大小，也就是说32位JVM的Mark word为32位，64位JVM为64位。其中 ptr_to_heavyweight_monitor：指向monitor对象（也称为管程或监视器锁）的起始地址，每个对象都存在着一个monitor与之关联，对象与其monitor之间的关系有存在多种实现方式，如monitor对象可以与对象一起创建销毁或当前线程试图获取对象锁时自动生，但当一个monitor被某个线程持有后，它便处于锁定状态。
    * Klass Point（类型指针）：对象指向它的类元数据的指针，虚拟机通过这个指针来确定这个对象是哪个类的实例。
* 实例数据
这部分主要是存放类的数据信息，父类的信息。
* 填充
由于虚拟机要求对象起始地址必须是8字节的整数倍，填充数据不是必须存在的，仅仅是为了字节对齐。

JAVA 虚拟机的指令集中有 monitorenter 和 monitorexit 两个指令来支持 synchronized 关键字的语义，这两个字节码都需要一个 reference 类型的参数来指明要锁定和解锁的对象。如果 Java 程序中的 synchronized 明确指定了对象参数，那就是这个对象的 reference ；如果没有明确指定，那就根据 synchronized 修饰的是实例方法还是类方法，去取对应的对象实例或 Class 对象来作为锁对象。

synchronized 无论是同步方法还是同步代码块， JVM 实现语义都是由 monitor 来支持的，当jvm执行到monitorenter指令时，当前线程试图获取monitor对象的所有权，如果未加锁或者已经被当前线程所持有，就把锁的计数器+1；当执行monitorexit指令时，锁计数器-1；当锁计数器为0时，该锁就被释放了。如果获取monitor对象失败，该线程则会进入阻塞状态，直到其他线程释放锁。

在Java虚拟机（HotSpot）中，monitor是由ObjectMonitor实现的，位于HotSpot虚拟机源码ObjectMonitor.hpp文件，C++实现的

![[Pasted image 20230519012140.png]]
ObjectMonitor中有两个队列，`_WaitSet` 和 `_EntryList`，用来保存ObjectWaiter对象列表( 每个等待锁的线程都会被封装成ObjectWaiter对象)，`_owner`指向持有ObjectMonitor对象的线程，当多个线程同时访问一段同步代码时，首先会进入 `_EntryList` 集合，当线程获取到对象的monitor 后进入 `_Owner` 区域并把monitor中的owner变量设置为当前线程同时monitor中的计数器count加1，若线程调用 wait() 方法，将释放当前持有的monitor，owner变量恢复为null，count自减1，同时该线程进入 WaitSe t集合中等待被唤醒。若当前线程执行完毕也将释放monitor(锁)并复位变量的值，以便其他线程进入获取monitor(锁)

### 同步代码块
首先写简单的一段同步代码块方法
```JAVA
public class TestSynchronized {

    public void testMethod() {
        synchronized (this) {
            System.out.println("hello world");
        }
    }
}
```
先编译出 .class 文件，然后对 .class 文件进行 javap -c 查看反编译字节码
![[Pasted image 20230519012548.png]]
可以看到有 monitorenter 和 monitorexit 两个指令

在虚拟机规范对 monitorenter 和 monitorexit 的行为描述中，有两点是需要特别注意的：
1. synchronized 同步块对同一条线程来说是可重入的，不会出现自己把自己锁死的问题。
2. 同步块在已进入的线程执行完之前，会阻塞后面其他线程的进入。

### 同步方法
```JAVA
    public synchronized void synchronizedMethod() {
        System.out.println("hello world");
    }
```
我们可以看到它的反编译结果并没有看到 monitorenter 和 monitorexit，而是有 ACC_SYNCHRONIZED

同步方法依赖flags标志ACC_SYNCHRONIZED实现，字节码中没有具体的逻辑
ACC_SYNCHRONIZED标志表示方法为同步方法
无 ACC_STATIC 标志为普通同步方法
有 ACC_STATIC 标志为静态同步方法

**方法的同步是隐式的，不需要通过字节码来控制，它的实现在方法调用和返回操作之中，会去隐式调用刚才的两个指令：monitorenter和monitorexit**

> 虚拟机可以从方法常量池中的方法表结构(method_info structure)中的 ACC_SYNCHRONIZED 访问标志区分一个方法是否是同步方法。当调用方法时，调用指令将会检查方法的 ACC_SYNCHRONIZED 访问标志是否设置了，如果设置了，执行线程将先持有同步锁，然后执行方法，最后在方法完成（无论是正常完成还是非正常完成）时释放同步锁。在方法执行期间，执行线程持有了同步锁，其他任何线程都无法再获得同一个锁。



## JVM 对 synchronized 的优化
Java 的线程是映射到操作系统的原生线程之上的，如果要阻塞或唤醒一个线程，都需要操作系统来帮忙完成，这就需要从用户态转到核心态中，因此状态转换需要耗费很多的处理器时间。对于代码简单的同步块（如 synchronized 修饰的 getter() 或 setter() 方法），状态转换消耗的时间可能比用户代码执行的时间还要长。所以 synchronized 是 Java 语言中一个重量级 (Heavyweight) 的操作。而虚拟机也会对此做一些优化

### 自旋锁与自适应自旋 Adaptive Spinning
互斥同步对性能最大的影响是阻塞的实现，挂起线程和回复线程的操作都需要转入内核态中完成，这些操作给系统的并发性能带来了很大的压力。在许多应用上，共享数据的锁定状态只会持续很短一段时间，为了这段时间去挂起和恢复线程不值得。如果物理机器有一个以上的处理器，能让两个或两个以上的线程同时并行执行，我们就可以让后面请求锁的那个线程“稍等一下”，但不放弃处理器的执行时间，看看持有锁的线程是否很快就会释放锁。为了让线程等待，我们只需让线程执行一个忙循环（自旋），这就是所谓的自旋锁。
自旋锁在 JDK1.4.2 中就已经引入，只不过默认是关闭的，可以使用 `-XX:+UseSpining` 参数来开启，在 JDK 1.6 中就已经改为默认开启了。自旋等待本身虽然避免了线程切换的开销，但它是要占用处理器时间的，如果锁占用的时间很短，自旋等待的效果就会非常好，反之，如果锁被占用的时间很长，那么自旋的线程就会白白消耗处理器资源，而不会做任何有用的工作，反而会带来性能上的浪费。因此，自旋等待的时间必须要有一定的限度，如果自旋超过了限定的次数仍然没有成功获得锁，就应当使用传统的方式去挂起线程了。自旋次数默认值是 10 次，用户可以使用参数 -XX:PreBlockSpin 来更改。
在 JDK1.6 中引入了自适应自旋锁。自适应意味着自旋的时间不再固定了，而是由前一次在同一个锁上的自旋时间及锁的拥有者的状态来决定。如果在同一个锁对象上，自旋等待刚刚成果获得过锁，并且持有锁的线程正在运行中，那么虚拟机就会认为这次自旋也很有可能再次成功，进而它将允许自旋等待持续相对更长的时间，比如 100 个循环。另外，如果对于某个锁，自旋很少成功获得过，那在以后要获取这个锁时将可能省略掉自旋过程，以避免浪费处理器资源。

### 锁消除 Lock Elimination
锁消除是指虚拟机即时编译器在运行时，对一些代码上要求同步，但是被检测到不可能存在共享数据竞争的锁进行消除。锁消除的主要判定依据来源于逃逸分析技术的数据支持，如果判断在一段代码中，堆上的所有数据都不会逃逸出去从而被其他线程访问到，那就可以把它们当做栈上数据对待，认为它们是线程私有的，同步加锁自然就无序进行。
例如下面一段代码
```JAVA
public String concatString(String s1, String s2, String s3) {
    return s1 + s2 + s3;
}
```
Java 编译器会对String 连接做自动优化。在 JDK1.5 之前会转化为 StringBuffer 对象的连续 append() 操作，在 JDK 1.5 及以后的版本中会转化为 StringBuilder 对象的连续 append() 操作。
JDK 1.5 以前 Javac 转化后的字符串连接操作
```JAVA
public String concatString(String s1, String s2, String s3) {
    StringBuffer sb = new StringBuffer();
    sb.append(s1);
    sb.append(s2);
    sb.append(s3);
    return sb.toString();
}
```
每个 StringBuffer.append() 方法中都有一个同步块，锁就是 sb 对象，虚拟机观察变量 sb，很快会发现它的动态作用域被限制在 concatString() 方法内部。也就是说， sb 的所有引用拥有不会“逃逸”到 concatString() 方法之外，其他线程无法访问到它，因此，虽然这里有锁，但是可以被安全地消除掉，在即时编译之后，这段代码就会忽略掉所有的同步而直接执行了（当然 JDK 1.5 之前的版本的虚拟机是不会的，依然是会有锁，而 JDK 1.5 以及后面的版本直接转化为非线程安全的 StringBuilder 来完成字符串拼接，并不会加锁。这里只是作为一个例子解释锁消除）

### 锁粗化 Lock Coarsening
如果一系列连续操作都对同一个对象反复加锁和解锁，甚至加锁操作是出现在循环体中的，那即使没有线程竞争，频繁地进行互斥同步操作也会导致不必要的性能损耗。
上面的代码 StringBuffer 连续的 append() 方法就属于这类情况。如果虚拟机探测到有这样一串零碎的操作都对同一个对象加锁，将会把加锁同步的范围扩展（粗化）到整个操作序列的外部。还是以上面的代码为例，就是扩展到第一个 append() 操作之前直至最后一个 append() 操作之后，这样只需要加锁一次就可以了。


### 轻量级锁
还是跟Mark Work 相关，如果这个对象是无锁的，jvm就会在当前线程的栈帧中建立一个叫锁记录（Lock Record）的空间，用来存储锁对象的Mark Word 拷贝，然后把Lock Record中的owner指向当前对象。

JVM接下来会利用CAS尝试把对象原本的Mark Word 更新会Lock Record的指针，成功就说明加锁成功，改变锁标志位，执行相关同步操作。

如果失败了，就会判断当前对象的Mark Word是否指向了当前线程的栈帧，是则表示当前的线程已经持有了这个对象的锁，否则说明被其他线程持有了，继续锁升级，修改锁的状态，之后等待的线程也阻塞

### 偏向锁
之前我提到过了，对象头是由Mark Word和Klass pointer 组成，锁争夺也就是对象头指向的Monitor对象的争夺，一旦有线程持有了这个对象，标志位修改为1，就进入偏向模式，同时会把这个线程的ID记录在对象的Mark Word中。

这个过程是采用了CAS乐观锁操作的，每次同一线程进入，虚拟机就不进行任何同步的操作了，对标志位+1就好了，不同线程过来，CAS会失败，也就意味着获取锁失败









