
___
# 声明周期总结
1. org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#doCreateBean
org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#createBeanInstance
反射调用构造函数生成实例化对象

2. org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#populateBean
注入属性

3. org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#initializeBean(java.lang.String, java.lang.Object, org.springframework.beans.factory.support.RootBeanDefinition)
org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#invokeAwareMethods

实现了 BeanNameAware 接口则调用 setBeanName() 方法，主要是为了通过 Bean 的引用来获得 Bean 的 ID ，一般业务中是很少有用到 Bean 的 ID 的

实现了 BeanClassLoaderAware 接口则调用 setBeanClassLoader() 方法

实现了 BeanFactoryAware 接口则调用 setBeanFactory() 方法
实现 BeanFactoryAware 主要目的是为了获取 Spring 容器，如 Bean 通过 Spring 容器发布事件等

4. org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#applyBeanPostProcessorsBeforeInitialization
org.springframework.context.support.ApplicationContextAwareProcessor#invokeAwareInterfaces
Spring 内置的一堆 BeanPostProcessor 接口，包括自定义实现的 BeanPostProcessor 接口，执行扫描注册得到的全部 BeanPostProcessor 的 postProcessBeforeInitialization 方法都是会进入并执行的
其中的`ApplicationContextAwareProcessor`中，如果 Bean 有实现以下的 aware 的接口
`EnvironmentAware``EmbeddedValueResolverAware``ResourceLoaderAware``ApplicationEventPublisherAware``MessageSourceAware``ApplicationStartupAware``ApplicationContextAware`，则会调用对应的 Set 方法
比如 ApplicationContext 就是在这是 set 进去的

其中的 InitDestroyAnnotationBeanPostProcessor 中，如果 Bean 有方法添加@PostConstruct注解，也会执行这个方法


5. org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#invokeInitMethods
实现了 InitializingBean 接口则执行 afterPropertiesSet() 方法

6. org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#applyBeanPostProcessorsAfterInitialization
Spring 内置的一堆 BeanPostProcessor 接口，包括自定义实现的 BeanPostProcessor 接口，执行扫描注册得到的全部 BeanPostProcessor 的 postProcessAfterInitialization 方法都是会进入并执行的

7. org.springframework.beans.factory.support.AbstractBeanFactory#registerDisposableBeanIfNecessary
实现了 DisposableBean 接口则执行 destroy() 方法

___
# 循环依赖总结
Spring 在 2.6.0 之前，spring会自动处理循环依赖的问题，2.6.0 以后的版本默认禁止 Bean 之间的循环引用，如果存在循环引用就会启动失败报错
需要手动启动允许循环依赖
```java
public static void main(String[] args) {
    SpringApplication sa = new SpringApplication(xx.class);
    sa.setAllowCircularReferences(Boolean.TRUE);//加入的参数
    sa.run(args);
}
```

我们先来讨论面试常问的 2.6.0 前的版本
## 成功情况
失败的情况太多了，不如关注一下成功的情况
当首个创建的 bean 满足一下两个条件
1. 是 singleton
2. 是通过属性注入或者 setter 注入，即
```java
    @Autowired
    private BeanB b;
```
后者
```java
    @Autowired
    public void setB(BeanB b) {
        this.b = b;
    }
```
则一定成功
否则都会抛出`BeanCurrentlyInCreationException`异常

## 循环依赖的解决
通过三级缓存解决循环依赖

### 关键成员变量
先看看具体的成员变量，看看是哪些个缓存
```java
	/**
	 * 第一级缓存
     * K：beanName
     * V：bean的实例对象（有代理对象则指的是代理对象，已经创建完毕）
	 * 存放已经创建完全的单例模式的 Bean 实例，如果有代理则是代理对象
	 * 这是面向开发人员使用的，我们在调用 ApplicationContext 的 getBean 方法的时候，就是在这里取实例
	 */
	private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);

	/**
     * 第三级缓存
     * K：beanName
     * V：ObjectFactory，该对象持有提前暴露的bean的引用
     * 通过ObjectFactory对象来存储单例模式下提前暴露的Bean实例的引用（正在创建中）
     * 该缓存是对内使用的，指的就是Spring框架内部逻辑使用该缓存
	 *
 	 */
	private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);

    /**
	 * 第二级缓存
     * K：beanName
     * V：bean的实例对象（有代理对象则指的是代理对象，该Bean还在创建中）
	 * 用于存储单例模式下创建的Bean实例（该Bean被提前暴露的引用,该Bean还在创建中）
	 * 该缓存是对内使用的，指的就是Spring框架内部逻辑使用该缓存
	 * 为了解决第一个classA引用最终如何替换为代理对象的问题（如果有代理对象）
     */
	private final Map<String, Object> earlySingletonObjects = new HashMap<>(16);

    /**
     * 记录当前正在创建的单例 beanName 的集合
     */
	private final Set<String> singletonsCurrentlyInCreation =
			Collections.newSetFromMap(new ConcurrentHashMap<>(16));

    /**
     * 被排除掉的正在创建检查的 beanName 的集合，也就是这个 beanName 不会再被创建了
     */
	private final Set<String> inCreationCheckExclusions =
			Collections.newSetFromMap(new ConcurrentHashMap<>(16));
```

### 创建 Bean 循环依赖流程
看看创建 Bean 的时候出现循环依赖的流程，方便我们知道三级缓存在哪些环节起作用
1. org.springframework.beans.factory.support.AbstractBeanFactory#getBean(java.lang.String)
2. org.springframework.beans.factory.support.AbstractBeanFactory#doGetBean
3. org.springframework.beans.factory.support.DefaultSingletonBeanRegistry#getSingleton(java.lang.String)
这里尝试从三个缓存中获取实例对象，但是首次创建的对象，在这里一定返回 null，从而进入下面的真正实例化流程
但如果是循环依赖，则会在第二次进入这个方法的时候，从第三级缓存取出早期引用，从而帮助循环依赖的对象成功注入属性
```java
	@Nullable
	protected Object getSingleton(String beanName, boolean allowEarlyReference) {
		// 从一级缓存获取当前创建的对象
		Object singletonObject = this.singletonObjects.get(beanName);
		/*
		首次创建的对象，从一级缓存取出一定为 null
		isSingletonCurrentlyInCreation 肯定为 true，因为在 beforeSingletonCreation 中已经添加过了
		 */
		if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
			synchronized (this.singletonObjects) {
				// 从二级缓存获取当前创建的对象
				singletonObject = this.earlySingletonObjects.get(beanName);
				// 首次创建的对象，从二级缓存取出一定为 null
				if (singletonObject == null && allowEarlyReference) {
					// 从三级缓存获取当前创建的对象
					ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
					/*
					首次创建的对象，从三级缓存取出也一定为 null
					此时才会进入 AbstractAutowireCapableBeanFactory 的 createBean 方法去进行实例化
					 */
					/*
					如果是出现循环依赖，创建的对象会在循环依赖的对象中，第二次来到这里
					此时对象已经经过了 addSingletonFactory 将早期引用放入第三级缓存，所以可以取出
					*/
					if (singletonFactory != null) {
						// 三级缓存中取出对象的早期引用
						singletonObject = singletonFactory.getObject();
						// 二级缓存添加对象，从此对象进入二级缓存，不再存在在三级缓存中
						this.earlySingletonObjects.put(beanName, singletonObject);
						// 三级缓存取在上面取出后就删除对象
						this.singletonFactories.remove(beanName);
					}
				}
			}
		}
		return singletonObject;
	}
```
4. org.springframework.beans.factory.support.DefaultSingletonBeanRegistry#getSingleton(java.lang.String, org.springframework.beans.factory.ObjectFactory<?>)
参数 ObjectFactory<?> singletonFactory 是个方法，就是第6步的 org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#createBean(java.lang.String, org.springframework.beans.factory.support.RootBeanDefinition, java.lang.Object[])
5. org.springframework.beans.factory.support.DefaultSingletonBeanRegistry#beforeSingletonCreation
在开始创建实例之前，现在这一步记录下当前创建的 beanName，记录在`singletonsCurrentlyInCreation`和`inCreationCheckExclusions`中
```java
	protected void beforeSingletonCreation(String beanName) {
		if (!this.inCreationCheckExclusions.contains(beanName) && !this.singletonsCurrentlyInCreation.add(beanName)) {
			throw new BeanCurrentlyInCreationException(beanName);
		}
	}
```
6. org.springframework.beans.factory.support.AbstractBeanFactory#createBean
是模版方法模式，实现在
org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#createBean(java.lang.String, org.springframework.beans.factory.support.RootBeanDefinition, java.lang.Object[])
中
7. org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#doCreateBean
8. org.springframework.beans.factory.support.DefaultSingletonBeanRegistry#addSingletonFactory
在这个位置前面已经创建了对象的还未完成依赖注入的早期实例了，在这个位置首次将这个早期实例的引用放入三级缓存
```java
	protected void addSingletonFactory(String beanName, ObjectFactory<?> singletonFactory) {
		Assert.notNull(singletonFactory, "Singleton factory must not be null");
		synchronized (this.singletonObjects) {
            /*
              判断当前创建的单例 beanName 是否在一级缓存中
              首次创建的话肯定是不存在的
             */
			if (!this.singletonObjects.containsKey(beanName)) {
                // 三级缓存放入 beanName 对应的早期引用
				this.singletonFactories.put(beanName, singletonFactory);
                // 二级缓存做一次清除操作，目的是保证三个缓存都只有同一个实例引用
				this.earlySingletonObjects.remove(beanName);
                // 记录已经创建的 beanName
				this.registeredSingletons.add(beanName);
			}
		}
	}
```
9. org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#populateBean
进行属性的依赖注入
```java
		if (hasInstAwareBpps) {
			if (pvs == null) {
				pvs = mbd.getPropertyValues();
			}
			for (BeanPostProcessor bp : getBeanPostProcessors()) {
				if (bp instanceof InstantiationAwareBeanPostProcessor) {
					InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
					// 通过注解 @Autowired 注入的属性在这里注入
					PropertyValues pvsToUse = ibp.postProcessProperties(pvs, bw.getWrappedInstance(), beanName);
					if (pvsToUse == null) {
						if (filteredPds == null) {
							filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
						}
						pvsToUse = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
						if (pvsToUse == null) {
							return;
						}
					}
					pvs = pvsToUse;
				}
			}
		}
```
    9.1 org.springframework.beans.factory.annotation.AutowiredAnnotationBeanPostProcessor#postProcessProperties
    进行 @Autowired 注解注入
    9.2 org.springframework.beans.factory.support.DefaultListableBeanFactory#doResolveDependency
    9.3 org.springframework.beans.factory.config.DependencyDescriptor#resolveCandidate
    存在依赖注入的话，会在这里又回到第一步
10. org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#doCreateBean
populateBean 中，循环依赖注入完成，Bean 将进入到最后的自我完善流程，此时会从二级缓存再次取出早期引用
```java
	protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
			throws BeanCreationException {
                ...
                // 再次获取 beanName 的对象实例，这一次会从二级缓存取出来
                Object earlySingletonReference = getSingleton(beanName, false);
                ...
	}
```
11. org.springframework.beans.factory.support.DefaultSingletonBeanRegistry#addSingleton
最后创建结束后，回到 addSingleton 方法中
```java
	public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
        ...
		if (newSingleton) {
			addSingleton(beanName, singletonObject);
		}
		...
	}
```
org.springframework.beans.factory.support.DefaultSingletonBeanRegistry#getSingleton(java.lang.String, org.springframework.beans.factory.ObjectFactory<?>)
这里面进入 addSingleton 将对象实例加入到一级缓存
```java
	protected void addSingleton(String beanName, Object singletonObject) {
		synchronized (this.singletonObjects) {
			// 一级缓存加入对象实例
			this.singletonObjects.put(beanName, singletonObject);
			// 二、三级缓存清除对象实例，保证三级缓存只有同一个对象实例
			this.singletonFactories.remove(beanName);
			this.earlySingletonObjects.remove(beanName);
			this.registeredSingletons.add(beanName);
		}
	}
```

### 总结
三级缓存
一级缓存是完整实例对象
二级缓存是代理对象的早期引用
三级缓存是原始对象的早期引用
