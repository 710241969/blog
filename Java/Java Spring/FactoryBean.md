

[[IOC 容器#BeanFactory]]
BeanFactory是个Factory，也就是IOC容器或对象⼯⼚，是 Spring IOC 容器最顶层的接口。

FactoryBean是个Bean。在Spring中，所有的Bean都是由BeanFactory(也就是IOC容器)的实现类来进⾏管理的。但对FactoryBean⽽⾔，这个Bean不是简单的Bean，⽽是**⼀个能⽣产或者修饰对象⽣成的⼯⼚Bean**，它的实现与设计模式中的⼯⼚模式和修饰器模式类似。

我们可以实现 FactoryBean 的接口，定义我们对某个类（比如 A.Class）的实例的创建方式。
通过@Component可以把我们实现的 FactoryBean 加入到 IOC 容器。
之后通过 IOC 容器的 getBean 方法去获取 A.Class ，就会调用到我们自己实现的`getObject()`。默认是单例模式，这样这个 A.Class 的实例会加入到 IOC 容器中。