# 概述
作用域限定了Spring Bean的作用范围，在Spring配置文件定义Bean时，通过声明scope配置项，可以灵活定义Bean的作用范围。例如，当你希望每次IOC容器返回的Bean是同一个实例时，可以设置scope为singleton；当你希望每次IOC容器返回的Bean实例是一个新的实例时，可以设置scope为prototype

# 使用
1. @Scope 注解
2. XML scope 配置参数

# singleton
默认的作用域,如果没有指定scope配置项，Bean的作用域被默认为singleton。
每个容器中只有一个bean的实例，单例的模式由BeanFactory自身来维护。
在整个系统上下文环境中，你通过Spring IOC获取的都是同一个实例。

# prototype
为每一个bean请求提供一个实例。意味着程序每次从IOC容器获取的Bean都是一个新的实例。因此，对有状态的bean应该使用prototype作用域。

# request
为 HTTP 请求创建一个实例，在请求完成以后，bean会失效并被垃圾回收器回收。
在一次 Http 请求中，容器会返回该 Bean 的同一实例。
而对不同的 Http 请求则会产生新的 Bean，而且该 bean 仅在当前Http Request内有效

# session
与request范围类似，确保每个session中有一个bean的实例，在session过期后，bean会随之失效。

# global-session
全局作用域，global-session和Portlet应用相关。当你的应用部署在Portlet容器中工作时，它包含很多portlet。如果你想要声明让所有的portlet共用全局的存储变量的话，那么这全局变量需要存储在global-session中。全局作用域与Servlet中的session作用域效果相同。