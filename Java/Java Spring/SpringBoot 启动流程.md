___
# 应用类型识别

## 源码分析

### 枚举类型
```java
public enum WebApplicationType {
	/**
	 * 应用程序不作为web应用启动，不启动内嵌的服务
	 */
	NONE,

	/**
	 * 应用程序以基于servlet的web应用启动，需启动内嵌servlet web服务
	 */
	SERVLET,

	/**
	 * 应用程序以响应式web应用启动，需启动内嵌的响应式web服务
	 */
	REACTIVE;
}
```

### 成员变量
```java
	private static final String[] SERVLET_INDICATOR_CLASSES = { "javax.servlet.Servlet",
			"org.springframework.web.context.ConfigurableWebApplicationContext" };

	private static final String WEBMVC_INDICATOR_CLASS = "org.springframework."
			+ "web.servlet.DispatcherServlet";

	private static final String WEBFLUX_INDICATOR_CLASS = "org."
			+ "springframework.web.reactive.DispatcherHandler";

	private static final String JERSEY_INDICATOR_CLASS = "org.glassfish.jersey.servlet.ServletContainer";
```

### 判断项目类型过程
```java
	static WebApplicationType deduceFromClasspath() {
		/*
		如果尝试加载 org.springframework.web.reactive.DispatcherHandler 成功
		并且加载 org.springframework.web.servlet.DispatcherServlet 和 org.glassfish.jersey.servlet.ServletContainer 失败
		则这个 application 是 WebApplicationType.REACTIVE 类型。
		 */
		if (ClassUtils.isPresent(WEBFLUX_INDICATOR_CLASS, null)
				&& !ClassUtils.isPresent(WEBMVC_INDICATOR_CLASS, null)
				&& !ClassUtils.isPresent(JERSEY_INDICATOR_CLASS, null)) {
			return WebApplicationType.REACTIVE;
		}
		/*
		如果尝试加载 javax.servlet.Servlet 
		和 org.springframework.web.context.ConfigurableWebApplicationContext 其中一个失败
		则 application 是 WebApplicationType.NONE 类型
		 */
		for (String className : SERVLET_INDICATOR_CLASSES) {
			if (!ClassUtils.isPresent(className, null)) {
				return WebApplicationType.NONE;
			}
		}
		/*
		根据上面的判断，来到这一步也就是加载 javax.servlet.Servlet
		和 org.springframework.web.context.ConfigurableWebApplicationContext 都成功了
		则 application 是 WebApplicationType.SERVLET 类型
		 */
		return WebApplicationType.SERVLET;
	}
```

___
# ApplicationContext 实现类确定
org.springframework.boot.SpringApplication#run(java.lang.String...)
org.springframework.boot.SpringApplication#createApplicationContext

## SpringApplication 源码分析

### 成员变量
```java
	public static final String DEFAULT_CONTEXT_CLASS = "org.springframework.context."
			+ "annotation.AnnotationConfigApplicationContext";

	/**
	 * The class name of application context that will be used by default for web
	 * environments.
	 */
	public static final String DEFAULT_SERVLET_WEB_CONTEXT_CLASS = "org.springframework.boot."
			+ "web.servlet.context.AnnotationConfigServletWebServerApplicationContext";

	/**
	 * The class name of application context that will be used by default for reactive web
	 * environments.
	 */
	public static final String DEFAULT_REACTIVE_WEB_CONTEXT_CLASS = "org.springframework."
			+ "boot.web.reactive.context.AnnotationConfigReactiveWebServerApplicationContext";
```

### 启动流程
```java
public ConfigurableApplicationContext run(String... args) {
		StopWatch stopWatch = new StopWatch();
		stopWatch.start();
		ConfigurableApplicationContext context = null;
		Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
		configureHeadlessProperty();
		SpringApplicationRunListeners listeners = getRunListeners(args);
		listeners.starting();
		try {
			ApplicationArguments applicationArguments = new DefaultApplicationArguments(
					args);
			ConfigurableEnvironment environment = prepareEnvironment(listeners,
					applicationArguments);
			configureIgnoreBeanInfo(environment);
			Banner printedBanner = printBanner(environment);
			// 根据项目类型获取 ApplicationContext 实现类
			context = createApplicationContext();
			exceptionReporters = getSpringFactoriesInstances(
					SpringBootExceptionReporter.class,
					new Class[] { ConfigurableApplicationContext.class }, context);
			prepareContext(context, environment, listeners, applicationArguments,
					printedBanner);
			// 完成 IOC 容器初始化
			refreshContext(context);
			afterRefresh(context, applicationArguments);
			stopWatch.stop();
			if (this.logStartupInfo) {
				new StartupInfoLogger(this.mainApplicationClass)
						.logStarted(getApplicationLog(), stopWatch);
			}
			listeners.started(context);
			callRunners(context, applicationArguments);
		}
		catch (Throwable ex) {
			handleRunFailure(context, ex, exceptionReporters, listeners);
			throw new IllegalStateException(ex);
		}

		try {
			listeners.running(context);
		}
		catch (Throwable ex) {
			handleRunFailure(context, ex, exceptionReporters, null);
			throw new IllegalStateException(ex);
		}
		return context;
	}
```

### createApplicationContext
创建 ApplicationContext 实现类
```java
	protected ConfigurableApplicationContext createApplicationContext() {
		Class<?> contextClass = this.applicationContextClass;
		if (contextClass == null) {
			try {
				/*
				根据 org.springframework.boot.WebApplicationType#deduceFromClasspath 的结果选择 ApplicationContext
				 */
				switch (this.webApplicationType) {
				case SERVLET:
					contextClass = Class.forName(DEFAULT_SERVLET_WEB_CONTEXT_CLASS);
					break;
				case REACTIVE:
					contextClass = Class.forName(DEFAULT_REACTIVE_WEB_CONTEXT_CLASS);
					break;
				default:
					contextClass = Class.forName(DEFAULT_CONTEXT_CLASS);
				}
			}
			catch (ClassNotFoundException ex) {
				throw new IllegalStateException(
						"Unable create a default ApplicationContext, "
								+ "please specify an ApplicationContextClass",
						ex);
			}
		}
		return (ConfigurableApplicationContext) BeanUtils.instantiateClass(contextClass);
	}
```

### refreshContext
IOC 容器初始化流程，最终其实调用的是 org.springframework.context.support.AbstractApplicationContext#refresh
[[IOC 容器#源码分析#refresh]]

## Tomcat 启动
Spring Boot是在容器的onRefresh方法中调用Tomcat的。
org.springframework.boot.web.servlet.context.ServletWebServerApplicationContext#onRefresh
```java
	@Override
	protected void onRefresh() {
		super.onRefresh();
		try {
			createWebServer();
		}
		catch (Throwable ex) {
			throw new ApplicationContextException("Unable to start web server", ex);
		}
	}
```
org.springframework.boot.web.servlet.context.ServletWebServerApplicationContext#createWebServer
```java
    private void createWebServer() {
        WebServer webServer = this.webServer;
        ServletContext servletContext = this.getServletContext();
        //webServer和servletContext都是null，表示还没创建容器，进入创建容器的逻辑
        if (webServer == null && servletContext == null) {
            //获取创建容器的工厂，可以通过WebServerFactoryCustomizer接口对这个工厂进行自定义设置
            ServletWebServerFactory factory = this.getWebServerFactory();
            //具体的创建容器的方法，我们进去具体看下
            this.webServer = factory.getWebServer(new ServletContextInitializer[]{this.getSelfInitializer()});
        } else if (servletContext != null) {
            try {
                this.getSelfInitializer().onStartup(servletContext);
            } catch (ServletException var4) {
                throw new ApplicationContextException("Cannot initialize servlet context", var4);
            }
        }
        this.initPropertySources();
    }
```
Servlet容器初始化：在Tomcat启动过程中，Servlet容器会被初始化。它会读取应用程序的类路径下的Servlet、Filter和Listener等相关组件，并进行初始化和注册。

请求处理：一旦Tomcat成功启动，它会监听指定的端口号，并等待来自客户端的HTTP请求。当收到请求时，Tomcat会根据配置的路由规则和请求的URL将请求转发给相应的Servlet进行处理。

