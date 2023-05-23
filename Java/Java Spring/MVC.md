# Spring MVC 工作流程
![[Pasted image 20230516173547.png]]

流程说明（重要）：
1. 客户端发送一个http请求给前端控制器（DispatcherServlet）；请求被Spring 前端控制Servelt DispatcherServlet捕获；
2. 前端控制器（DispacherServlet）根据请求信息调用处理器映射器（HandlerMapping）；DispatcherServlet对请求URL进行解析，得到请求资源标识符（URI）。然后根据该URI，调用HandlerMapping获得该Handler配置的所有相关的对象（包括Handler对象以及Handler对象对应的拦截器）
3. 处理器映射器（HandlerMapping）根据url找到具体的处理器（Handler），生成处理器对象以及对应的处理器拦截器（HandlerInterceptor有则生成）最后以HandlerExecutionChain对象的形式返回给前端控制器（DispacherServlet）；
4. 前端控制器（DispacherServlet）根据返回信息找到对应的处理器适配器（HandlerAdapter）；（附注：如果成功获得HandlerAdapter后，此时将开始执行拦截器的preHandler(...)方法）
5. 处理器适配器（HandlerAdapter）会调用并执行（处理器）Handler，这里的处理器指的是程序中编写的Controller类，也称后端控制器，通过反射调用；
6. Handler 返回业务数据
7. HandlerAdapter 拿到 Handler 返回的数据后，生成 ModleAndView 对象，返回ModelAndView对象（Model 是返回的数据对象，View 是个逻辑上的视图）给前端控制器（DispacherServlet）；
8. 前端控制器（DispacherServlet）根据返回信息，选择一个适合的ViewResolver（必须是已经注册到Spring容器中的ViewResolver)，找到 ViewReslover 将逻辑视图解析为具体的视图（view）
9. 返回 View 接口的具体实现
10. View 通过最终调用自己 render 方法的具体实现，进行渲染成完整的视图（view）并复制字节流到 http response 的输出流，返回给客户端。
11. 返回响应到客户端

# SpringMVC 重要组件说明

1、前端控制器DispatcherServlet（不需要工程师开发）,由框架提供（重要）

作用：Spring MVC 的入口函数。接收请求，响应结果，相当于转发器，中央处理器。有了 DispatcherServlet 减少了其它组件之间的耦合度。用户请求到达前端控制器，它就相当于mvc模式中的c，DispatcherServlet是整个流程控制的中心，由它调用其它组件处理用户的请求，DispatcherServlet的存在降低了组件之间的耦合性。

2、处理器映射器HandlerMapping(不需要工程师开发),由框架提供

作用：根据请求的url查找Handler。HandlerMapping负责根据用户请求找到Handler即处理器（Controller），SpringMVC提供了不同的映射器实现不同的映射方式，例如：配置文件方式，实现接口方式，注解方式等。

3、处理器适配器HandlerAdapter

作用：按照特定规则（HandlerAdapter要求的规则）去执行Handler 通过HandlerAdapter对处理器进行执行，这是适配器模式的应用，通过扩展适配器可以对更多类型的处理器进行执行。

4、处理器Handler(需要工程师开发)

注意：编写Handler时按照HandlerAdapter的要求去做，这样适配器才可以去正确执行Handler Handler 是继DispatcherServlet前端控制器的后端控制器，在DispatcherServlet的控制下Handler对具体的用户请求进行处理。 由于Handler涉及到具体的用户业务请求，所以一般情况需要工程师根据业务需求开发Handler。

5、视图解析器View resolver(不需要工程师开发),由框架提供

作用：进行视图解析，根据逻辑视图名解析成真正的视图（view） View Resolver负责将处理结果生成View视图，View Resolver首先根据逻辑视图名解析成物理视图名即具体的页面地址，再生成View视图对象，最后对View进行渲染将处理结果通过页面展示给用户。 springmvc框架提供了很多的View视图类型，包括：jstlView、freemarkerView、pdfView等。 一般情况下需要通过页面标签或页面模版技术将模型数据通过页面展示给用户，需要由工程师根据业务需求开发具体的页面。

6、视图View(需要工程师开发)

View是一个接口，实现类支持不同的View类型（jsp、freemarker、pdf...）

注意：处理器Handler（也就是我们平常说的Controller控制器）以及视图层view都是需要我们自己手动开发的。其他的一些组件比如：前端控制器DispatcherServlet、处理器映射器HandlerMapping、处理器适配器HandlerAdapter等等都是框架提供给我们的，不需要自己手动开发。

# 重定向
redirect 又叫重定向，表示转发，当请求发给A服务时，服务A返回重定向给客户端，客户端再去请求B服务
1. redirect不支持post请求
2. redirect 如果要携带请求参数，需要在url地址中进行编码防止中文乱码。
## 1. 使用 response 的重定向功能，此方法可以跳转外网url
```java
public void toRedirect(HttpServletResponse response) throws Exception{
	response.sendRedirect("https://www.baidu.com");
}
```
## 2. 在返回的字符串前面添加`redirect:`方式来告诉Spring框架，需要做302重定向处理
```java
public  String testredirect(HttpServletResponse response){  
    return "redirect:/index";  
} 
```
带参数
```java
public String testredirect(Model model, RedirectAttributes attr) {
	attr.addAttribute("param", "123"); // 跳转地址带上test参数
	return "redirect:/user/users";
}
```
或者用 url 上的参数
```java
public  String testredirect(HttpServletResponse response){  
    return "redirect:/index?param=123";  
} 
```
## 3. 通过返回 ModelAndView 重定向
```java
public  ModelAndView restredirect(String userName){  
    ModelAndView  model = new ModelAndView("redirect:/main/index");    
    return model;  
}
```
带参数
```java
public  ModelAndView toredirect(String userName){  
    ModelAndView  model = new ModelAndView("/main/index");   
    model.addObject("param", 123);  // 把 param 参数带入到controller 的 RedirectAttributes
    return model;  
}
```

# 转发
forward又叫转发，表示转发，当请求来到时，可以将请求转发到其他的指定服务，用户端不知晓。
使用forward注意事项
1. 转发和被转发的请求类型必须一致，即全是GET或者POST
2. 转发者方法不能被标识位@RestController或者@ResponseBody
## 1. 在返回的字符串前面添加`forward:`方式来告诉 Spring 框架，需要做转发
```java
public  String testredirect(HttpServletResponse response){  
    return "forward:/index";  
}
```
带参数
```java
public  String testredirect(HttpServletRequest request){ 
    request.setAttribute("param", "123");   //把 param 参数传递到request中
    return "forward:/user/index";  
}
```
## 2. ModelAndView请求转发
```java
public  ModelAndView restredirect(String userName){  
    ModelAndView  model = new ModelAndView("forward:/main/index"); // 默认forward，可以不用写
    return model;  
}
```
带参数
```java
public  ModelAndView toredirect(String userName){  
    ModelAndView  model = new ModelAndView("/user/userinfo");   
    model.addObject("param", 123); // 把 param 参数带入到controller的RedirectAttributes
    return model;  
}
```

# 重定向和转发的区别
1. 从地址栏显示来说：
​forword是服务器内部的重定向，服务器直接访问目标地址的 url网址，把里面的东西读取出来，但是客户端 并不知道，因此用forward的话，客户端浏览器的网址是不会发生变化的。
​redirect是服务器根据逻辑，发送一个状态码，告诉浏览器重新去请求那个地址，所以地址栏显示的是新的地址。

2. 从数据共享来说：
​由于在整个定向的过程中用的是同一个request，因此forward会将request的信息带到被重定向的jsp或者 servlet中使用。即可以共享数据
​redirect不能共享

3. 从运用的地方来说
​forword 一般用于用户登录的时候，根据角色转发到相应的模块
​redirect一般用于用户注销登录时返回主页面或者跳转到 其他网站

4. 从效率来说：
​forword效率高，而redirect效率低

5. 从本质来说：
​forword转发是服务器上的行为，而redirect重定向是客户端的行为

6. 从请求的次数来说：
​forword只有一次请求；而redirect有两次请求