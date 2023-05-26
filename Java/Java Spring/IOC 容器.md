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

___
# 源码分析

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
prepareRefresh
obtainFreshBeanFactory
prepareBeanFactory
postProcessBeanFactory
invokeBeanFactoryPostProcessors
registerBeanPostProcessors
initMessageSource
initApplicationEventMulticaster
onRefresh
registerListeners
finishBeanFactoryInitialization
finishRefresh
步骤2-6是创建并准备beanFactory对象
步骤7-12是完善applicationContext的其他功能
步骤11是创建非懒加载的单例对象

prepareRefresh准备上下文环境，系统环境变量

obtainFreshBeanFactory创建BeanFactory，为beanFactory中的成员变量beanDefinitionMap（**是个容器**）进行初始化，该map的作用是保存BeanDefinition对象。BeanDefinition作为bean的设计蓝图，规定了bean的特征，如单例、依赖关系、初始销毁方法。后续 bean 的创建就是根据这个来的

prepareBeanFactory进一步完善beanFactory。为它的其他各项成员变量赋值

postProcessBeanFactory 是一个空实现，留给子类扩展。这里体现了模板方法设计模式

invokeBeanFactoryPostProcessors 会调用**所有的**beanFactory后置处理器，作用是充当**beanFactory的扩展点**，可以**添加或修改BeanDefinition**

registerBeanPostProcessors  beanDefinitionMap中找出所有的bean后置处理器，生成实例对象并保存到beanPostProcessors集合。就是找出实现了 BeanPostProcessors 接口的类

initMessageSource 向ApplicationContext添加messageSource成员，实现国际化功能

initApplicationContextEventMulticaster 向ApplicationContext的成员变量applicationEventMulticaster赋值（初始化）。事件广播器，它里面维护了一个集合，该集合保存了spring容器的所有监听器。如果需用用事件广播器发生事件，则事件广播器就会遍历它所维护的集合，然后给相应的监听器发送事件

onRefresh 这一步也是一个空实现，留给子类扩展。模板方法模式。比如 spring boot Tomcat 就在这里面启动

registerListeners 从多种途径中找到事件监听器对象，并添加到applicationEventMulticaster维护的集合当中

finishBeanFactoryInitialization 这一步会将beanFactory的成员补充完毕。并且初始化**剩下的所有的非懒加载单例bean**

finishRefresh 为ApplicationContext添加lifecycleProcessor成员。用来控制容器内需要生命周期管理的bean

prepareRefresh：准备好环境变量，配置变量
obtainFreshBeanFactory：创建或获取bean工厂
prepareBeanFactory：准备bean工厂
postProcessBeanFactory：子类去扩展bean工厂
invokeBeanFactoryPostProcessors：自定义beanFactory后置处理器去扩展bean工厂
registerBeanPostProcessors：准备bean后置处理器
initMessageSource：为spring容器提供国际化功能
initApplicationEventMulticaster：为spring容器提供事件发布器
onRefresh：留给子类对spring容器进行扩展
registerListeners：为spring容器注册监听器
finishBeanFactoryInitialization：初始化剩余的非懒加载单例bean，执行bean后置处理器扩展
finishRefresh：准备spring容器生命周期管理器，发布contextRefreshed事件


