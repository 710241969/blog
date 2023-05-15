
# 和 ThreadLocal 异同
和 ThreadLocal 不同点是在子线程创建时，会复制给子线程
其他完全一样

# 组织关系
```java
public class Thread implements Runnable {
	/*  
	* InheritableThreadLocal values pertaining to this thread. This map is  
	* maintained by the InheritableThreadLocal class.  
	*/  
	ThreadLocal.ThreadLocalMap inheritableThreadLocals = null;
}
```
Thread 类中定义了默认权限的成员变量 `inheritableThreadLocals`
inheritableThreadLocals 和 threadLocals 基本一致，用来将父线程的数据传递给子线程，父线程创建子线程时，会自动将 inheritableThreadLocals 的值浅拷贝到子线程的 inheritableThreadLocals 中，而且只有在创建时复制，所以父线程对 inheritableThreadLocals 值的改动子线程不可见

