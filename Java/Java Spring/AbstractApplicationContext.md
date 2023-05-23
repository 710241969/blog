
___
# 源码分析

## refresh

org.springframework.context.support.AbstractApplicationContext#invokeBeanFactoryPostProcessors


registerBeanPostProcessors(beanFactory);
org.springframework.context.support.AbstractApplicationContext#registerBeanPostProcessors
```java
	protected void registerBeanPostProcessors(ConfigurableListableBeanFactory beanFactory) {
		PostProcessorRegistrationDelegate.registerBeanPostProcessors(beanFactory, this);
	}
```

