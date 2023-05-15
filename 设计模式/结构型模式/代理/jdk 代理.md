# 概述
代理类必须实现 InvocationHandler 接口，我们要实现 invoke 方法，在 invoke 方法内进行被代理类方法的调用，以及实现代理的功能

调用 Proxy.newProxyInstance 生成代理对象

如果想要实现JDK动态代理那么代理类必须实现接口，否则不能使用

# 代码实现
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

