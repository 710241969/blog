___
# DI
DI Dependency Injection 依赖注入

依赖注入，是 IOC 的一个方面，是个通常的概念，它有多种解释。这概念是说你不用直接创建对象，而只需要描述它如何被创建，以及他们的依赖关系。
不在代码里直接组装你的组件和服务，但是要在配置文件里或注解描述哪些组件需要哪些服务，之后一个容器（IOC 容器）负责把他们生成实例并根据依赖的定义组装起来

___
# IOC
IOC Inversion of Control 控制反转
控制反转是从容器的角度在描述依赖注入，由 IOC 容器来负责完成 DI 过程，并管理所有创建出来的实例

___
# BeanFactory
BeanFactory，以Factory结尾，表示它是一个工厂类(接口)，它负责生产和管理bean的一个工厂。

在Spring中，BeanFactory是IOC容器的核心接口，它的职责包括：实例化、定位、配置应用程序中的对象及建立这些对象间的依赖。作为最顶层的一个接口类，它定义了 IOC 容器的基本功能规范

BeanFactory只是个接口，并不是IOC容器的具体实现。BeanFactory有3个重要的子类：ListableBeanFactory、HierarchicalBeanFactory 和 AutowireCapableBeanFactory。

ListableBeanFactory：表示这些Bean是可列表化的；
HierarchicalBeanFactory：表示的是这些Bean是有继承关系的，也就是每个Bean有可能有父Bean；
AutowireCapableBeanFactory：定义Bean的自动装配规则。

这三个类最终的默认实现类是 DefaultListableBeanFactory，它实现了所有的接口

___
# ApplicationContext
应用上下文 ApplicationContext 接口继承 BeanFactory 接口
同时还继承了许多接口来对 BeanFactory 进行功能扩展，提供了更多面向应用的功能
1. 支持信息源，可以实现国际化。（实现 MessageSource 接口）
2. 访问资源。(实现 ResourcePatternResolver 接口)
3. 支持应用事件。(实现 ApplicationEventPublisher 接口)

## BeanFactory 接口和 ApplicationContext 接口有什么区别
我们一般称 BeanFactory 为 IoC 容器，而称 ApplicationContext 为应用上下文。但有时为了行文方便，我们也将 ApplicationContext 称为 Spring 容器
BeanFactory是Sping框架的基础设施，而面向sping本身。
ApplicationContext面向使用Sping框架的开发者，几乎所有的应用场合我们都是直接使用ApplicationContet而非底层的BeanFactory。

通常的，ApplicationContext 的实现类在加载配置文件的时候就会创建对象。BeanFactory 的实现类在使用对象的时候才会创建对象。不过这也取决于具体实现了。

___
# 启动流程
主要是 org.springframework.context.support.AbstractApplicationContext#refresh 方法完成

## 源码分析

### refresh
```java
	@Override
	public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			// Prepare this context for refreshing.
			prepareRefresh();

			// Tell the subclass to refresh the internal bean factory.
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

			// Prepare the bean factory for use in this context.
			prepareBeanFactory(beanFactory);

			try {
				// Allows post-processing of the bean factory in context subclasses.
				postProcessBeanFactory(beanFactory);

				// Invoke factory processors registered as beans in the context.
				invokeBeanFactoryPostProcessors(beanFactory);

				// Register bean processors that intercept bean creation.
				registerBeanPostProcessors(beanFactory);

				// Initialize message source for this context.
				initMessageSource();

				// Initialize event multicaster for this context.
				initApplicationEventMulticaster();

				// Initialize other special beans in specific context subclasses.
				onRefresh();

				// Check for listener beans and register them.
				registerListeners();

				// 完成所有单例 Bean 的实例化、依赖注入等初始化操作，也就是完成我们注解的所有单例 Bean 的实例化
				finishBeanFactoryInitialization(beanFactory);

				// Last step: publish corresponding event.
				finishRefresh();
			}

			catch (BeansException ex) {
				if (logger.isWarnEnabled()) {
					logger.warn("Exception encountered during context initialization - " +
							"cancelling refresh attempt: " + ex);
				}

				// Destroy already created singletons to avoid dangling resources.
				destroyBeans();

				// Reset 'active' flag.
				cancelRefresh(ex);

				// Propagate exception to caller.
				throw ex;
			}

			finally {
				// Reset common introspection caches in Spring's core, since we
				// might not ever need metadata for singleton beans anymore...
				resetCommonCaches();
			}
		}
	}
```


