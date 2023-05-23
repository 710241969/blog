AOP，Aspect-Oriented Programming
面向切面编程
能够对我们设计的业务类的对象方法进行统一的增强。将那些与业务无关，却为业务模块所共同调用的逻辑或责任（例如事务处理、日志管理、权限控制等）封装起来，便于减少系统的重复代码，降低模块间的耦合度，并有利于未来的可拓展性和可维护性

Spring AOP 代理是通过动态代理实现的，具体实现分为 JDK 代理和 Cglib 代理
通过动态代理的方式，实现被代理类方法的增强，比如加日志，事务处理等

# 源码分析
具体实现分为 JDK 代理和 Cglib 代理，那怎么决定具体使用哪一种动态代理呢
Spring AOP 决定使用的代理方式，主要代码在 `DefaultAopProxyFactory.createAopProxy`
```java
/**
 * Default {@link AopProxyFactory} implementation, creating either a CGLIB proxy
 * or a JDK dynamic proxy.
 *
 * <p>Creates a CGLIB proxy if one the following is true for a given
 * {@link AdvisedSupport} instance:
 * <ul>
 * <li>the {@code optimize} flag is set
 * <li>the {@code proxyTargetClass} flag is set
 * <li>no proxy interfaces have been specified
 * </ul>
 *
 * <p>In general, specify {@code proxyTargetClass} to enforce a CGLIB proxy,
 * or specify one or more interfaces to use a JDK dynamic proxy.
 *
 * @author Rod Johnson
 * @author Juergen Hoeller
 * @since 12.03.2004
 * @see AdvisedSupport#setOptimize
 * @see AdvisedSupport#setProxyTargetClass
 * @see AdvisedSupport#setInterfaces
 */
@SuppressWarnings("serial")
public class DefaultAopProxyFactory implements AopProxyFactory, Serializable {
    ...
	@Override
	public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
		if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
			Class<?> targetClass = config.getTargetClass();
			if (targetClass == null) {
				throw new AopConfigException("TargetSource cannot determine target class: " +
						"Either an interface or a target is required for proxy creation.");
			}
			if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
				return new JdkDynamicAopProxy(config);
			}
			return new ObjenesisCglibAopProxy(config);
		}
		else {
			return new JdkDynamicAopProxy(config);
		}
	}
    ...
}
```
config.isOptimize：控制通过 cglib 创建的代理是否使用激进的优化策略，默认是 false ，一般也不会改变
config.isProxyTargetClass：是否使用cglib来创建代理对象，默认是 false ，可以通过如下方式设置为true
```java
@EnableAspectJAutoProxy(proxyTargetClass = true)
```
**根据代码可以看到，使用 cglib 代理的时候，当代理对象是个接口，或者是代理对象是 jdk 动态代理对象时，依旧会使用 jdk 动态代理**