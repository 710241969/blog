___
# 概述
就是为了实现某个对象的方法增强

___
# 静态代理
功能接口
```java
public interface InterfaceObject {
    void doSomething();
}
```

真实被代理对象
```java
public class TargetObject implements InterfaceObject {
    public void doSomething() {
        System.out.println("被代理的真实对象 TargetObject 的 doSomething 方法");
    }
}
```

代理类
```java
public class ProxyObject implements InterfaceObject {

    private TargetObject targetObject;

    public ProxyObject(TargetObject targetObject) throws Exception {
        if (null == targetObject) {
            throw new Exception("传入为 null");
        }
        this.targetObject = targetObject;
    }

    public void doSomething() {
        System.out.println("代理对象执行 before doSomething");
        targetObject.doSomething();
        System.out.println("代理对象执行 after doSomething");
    }
}
```

___
# JDK 代理
代理类必须实现 InvocationHandler 接口，我们要实现 invoke 方法，在 invoke 方法内进行被代理类方法的调用，以及实现代理的功能

调用 Proxy.newProxyInstance 生成代理对象

如果想要实现JDK动态代理那么代理类必须实现接口，否则不能使用

功能接口
```java
public interface InterfaceObject {
    void doSomething1();

    void doSomething2();
}
```
被代理类
```java
public class TargetObject implements InterfaceObject {
    public void doSomething1() {
        System.out.println("被代理的真实对象 TargetObject 的 doSomething1 方法");
    }

    public void doSomething2() {
        System.out.println("被代理的真实对象 TargetObject 的 doSomething2 方法");
    }
}
```
## 方式一，静态工厂方法
通过静态工厂方法，调用 Proxy.newProxyInstance 生成代理对象，匿名实现 InvocationHandler 接口，得到代理类
这个过程就不存在代理类的定义
```java
public class ProxyFactory {
    public static Object getProxyInstance(Object targetObject) {
        return Proxy.newProxyInstance(
                targetObject.getClass().getClassLoader(),
                targetObject.getClass().getInterfaces(),
                (proxy, method, args) -> {
                    System.out.println("代理对象执行 before " + method.getName());
                    Object methodResult = method.invoke(targetObject, args);
                    System.out.println("代理对象执行 after " + method.getName());
                    return methodResult;
                });
    }
}
```
使用
```java
        InterfaceObject realObject = new TargetObject();
        InterfaceObject proxyObject = (InterfaceObject) ProxyFactory.getProxyInstance(realObject);
        proxyObject.doSomething1();
        proxyObject.doSomething2();
```
输出
```
代理对象执行 before doSomething1
被代理的真实对象 TargetObject 的 doSomething1 方法
代理对象执行 after doSomething1
代理对象执行 before doSomething2
被代理的真实对象 TargetObject 的 doSomething2 方法
代理对象执行 after doSomething2
```
## 方式二，定义代理类

```java
public class ProxyHandler implements InvocationHandler {
    private InterfaceObject targetObject;

    private ProxyHandler(InterfaceObject targetObject) {
        this.targetObject = targetObject;
    }

    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("代理对象执行 before " + method.getName());
        Object methodResult = method.invoke(targetObject, args);
        System.out.println("代理对象执行 after " + method.getName());
        return methodResult;
    }

    public static Object getProxyInstance(InterfaceObject targetObject) {
        ProxyHandler proxyHandler = new ProxyHandler(targetObject);
        return Proxy.newProxyInstance(
                targetObject.getClass().getClassLoader(),
                targetObject.getClass().getInterfaces(),
                proxyHandler);
    }
}
```
使用
```java
        InterfaceObject realObject = new TargetObject();
        InterfaceObject proxyObject = (InterfaceObject) ProxyHandler.getProxyInstance(realObject);
        proxyObject.doSomething1();
        proxyObject.doSomething2();
```
输出
```
代理对象执行 before doSomething1
被代理的真实对象 TargetObject 的 doSomething1 方法
代理对象执行 after doSomething1
代理对象执行 before doSomething2
被代理的真实对象 TargetObject 的 doSomething2 方法
代理对象执行 after doSomething2
```

___
# CGLIB 代理
CGLib 代理 CGLib 是针对类实现代理，主要是对指定的类生成一个子类，覆盖其中的方法（继承）
Cglib 包的底层是通过使用一个小而快的字节码处理框架 ASM 来转换字节码并生成新的类
需要注意的是，CGLib 不能对声明为 final 的类和方法进行代理，因为 final 的类和方法不能继承，虽然不会报错，但是会调用原来的函数，代理会失效

代理类必须实现 MethodInterceptor 接口，我们要实现 intercept 方法，在 intercept 方法内进行被代理类方法的调用，以及实现代理的功能

调用 Proxy.newProxyInstance 生成代理对象

代理类必须实现 MethodInterceptor 接口，我们要实现 intercept 方法，在 intercept 方法内进行被代理类方法的调用，以及实现代理的功能
使用 Enhancer 类，指定父类，指定代理增强的实现类，最后调用 Enhancer 的 create 方法创建对象

## 被代理类， doSomething2 函数被定义为 final
```java
public class TargetObject {
    public void doSomething1() {
        System.out.println("被代理的真实对象 TargetObject 的 doSomething1 方法");
    }

    public final void doSomething2() {
        System.out.println("被代理的真实对象 TargetObject 的 doSomething2 方法");
    }
}
```
## 方式一，静态工厂方法
通过静态工厂方法，调用 Proxy.newProxyInstance 生成代理对象，匿名实现 InvocationHandler 接口，得到代理类
这个过程就不存在代理类的定义
```java
public class ProxyFactory {
    public static Object getProxyInstance(Object targetObject) {
        Enhancer enhancer = new Enhancer();
        enhancer.setSuperclass(targetObject.getClass());
        enhancer.setCallback((MethodInterceptor) (obj, method, args, proxy) -> {
            System.out.println("代理对象执行 before " + method.getName());
        /*
          如果我们通过反射 methodProxy.invoke(arg0, ...) 这种方式是无法调用到父类的方法的
          子类有方法重写，隐藏了父类的方法，父类的方法已经不可见
          如果调 methodProxy.invoke(arg0, ...) 会造成死循环，不停调用当前函数，发生 StackOverflowError
         */
            Object methodResult = proxy.invokeSuper(obj, args);
            System.out.println("代理对象执行 after " + method.getName());
            return methodResult;
        });
        return enhancer.create();
    }
}
```
使用
```java
        TargetObject targetObject = new TargetObject();
        TargetObject proxyObject = (TargetObject) ProxyFactory.getProxyInstance(targetObject);
        proxyObject.doSomething1();
        proxyObject.doSomething2();
```
输出
```
代理对象执行 before doSomething1
被代理的真实对象 TargetObject 的 doSomething1 方法
代理对象执行 after doSomething1
被代理的真实对象 TargetObject 的 doSomething2 方法
```
可以看到`doSomething2`方法并没有得到增强

## 方式二，定义代理类

```java
public class ProxyHandler implements MethodInterceptor {
    private ProxyHandler() {
    }

    public Object intercept(Object object, Method method, Object[] args, MethodProxy methodProxy) throws Throwable {
        System.out.println("代理对象执行 before " + method.getName());
        /*
          如果我们通过反射 methodProxy.invoke(arg0, ...) 这种方式是无法调用到父类的方法的
          子类有方法重写，隐藏了父类的方法，父类的方法已经不可见
          如果调 methodProxy.invoke(arg0, ...) 会造成死循环，不停调用当前函数，发生 StackOverflowError
         */
        Object methodResult = methodProxy.invokeSuper(object, args);
        System.out.println("代理对象执行 after " + method.getName());
        return methodResult;
    }

    public static Object getProxyInstance(Object targetObject) {
        Enhancer enhancer = new Enhancer();
        enhancer.setSuperclass(targetObject.getClass());
        enhancer.setCallback(new ProxyHandler());
        return enhancer.create();
    }
}
```
使用
```java
        TargetObject targetObject = new TargetObject();
        TargetObject proxyObject = (TargetObject) ProxyHandler.getProxyInstance(targetObject);
        proxyObject.doSomething1();
        proxyObject.doSomething2();
```
输出
```
代理对象执行 before doSomething1
被代理的真实对象 TargetObject 的 doSomething1 方法
代理对象执行 after doSomething1
被代理的真实对象 TargetObject 的 doSomething2 方法
```
可以看到`doSomething2`方法并没有得到增强



















