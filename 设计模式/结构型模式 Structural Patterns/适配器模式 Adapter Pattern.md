___
# 适配器模式是怎么回事
无非是我们需要一只鸭，但是我们只有一只鸡，这个时候就需要定义一个适配器，由这个适配器来充当鸭，但是适配器里面的方法还是由鸡来实现的。

适配器模式总体来说分三种：默认适配器模式、对象适配器模式、类适配器模式。先不急着分清楚这几个，先看看例子再说。

___
# 代码实现
我们有一个鸭子实现
```java
/**
 * 鸭子类的具体实现
 *
 * @author wangyiming
 */
public class Duck {
    public void duckFly() {
        System.out.println("鸭子飞");
    }
}
```
有一个鸡子的接口，或者是抽象类，或者是实现类，都行的
```java
/**
 * 目标抽象类定义客户所需接口，可以是一个抽象类或接口，也可以是具体类
 * 鸡类的接口
 *
 * @author wangyiming
 */
public interface InterfaceChicken {
    void fly();
}
```

我们想要鸡子，但是想用鸭子的方式来飞

## 对象适配器模式
```java
/**
 * 对象适配器模式
 * 我们已经有 鸭 的实现，但是我们希望得到一只 鸡，并且这只 鸡 能用 鸭 的方式来飞
 * 但是 鸡 和 鸭 没有任何继承关系
 * 此时就可以通过对象适配器，实现或继承 鸡 但是内部调用的其实是 鸭 的方法
 */
public class ObjectAdapter implements InterfaceChicken {
    private Duck adapter;

    public ObjectAdapter(Duck adapter) {
        this.adapter = adapter;
    }

    @Override
    public void fly() {
        adapter.duckFly();
    }
}
```

## 类适配器模式
```java
/**
 * 类适配器
 * <p>
 * 用类继承的方式做适配器
 *
 * @author wangyiming
 */
public class ClassAdapter extends Duck implements InterfaceChicken {

    @Override
    public void fly() {
        super.duckFly();
    }
}
```











