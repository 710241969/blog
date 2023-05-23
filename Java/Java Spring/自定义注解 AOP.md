<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>

___
# 1. 自定义注解
```java
@Documented
@Target(value = ElementType.METHOD)
@Inherited
@Retention(value = RetentionPolicy.RUNTIME)
public @interface MyAnnotation {

}
```

___
# 2. 定义 AOP 实现
org.aspectj.lang.annotation.Aspect

@Pointcut ("execution()")
org.aspectj.lang.annotation.Pointcut
切入点表达式常用举例
execution(* com.wmx.aspect.EmpServiceImpl.findEmpById(Integer))	匹配 com.wmx.aspect.EmpService 类中的 findEmpById 方法，且带有一个 Integer 类型参数。
execution(* com.wmx.aspect.EmpServiceImpl.findEmpById(*))	匹配 com.wmx.aspect.EmpService 类中的 findEmpById 方法，且带有一个任意类型参数。
execution(* com.wmx.aspect.EmpServiceImpl.findEmpById(..))	匹配 com.wmx.aspect.EmpService 类中的 findEmpById 方法，参数不限。
execution(* grp.basic3.se.service.SEBasAgencyService3.editAgencyInfo(..)) || execution(* grp.basic3.se.service.SEBasAgencyService3.adjustAgencyInfo(..))	匹配 editAgencyInfo 方法或者 adjustAgencyInfo 方法
execution(* com.wmx.aspect.EmpService.*(..))	匹配 com.wmx.aspect.EmpService 类中的任意方法
execution(* com.wmx.aspect.*.*(..))	匹配 com.wmx.aspect 包(不含子包)下任意类中的任意方法
execution(* com.wmx.aspect..*.*(..))	匹配 com.wmx.aspect 包及其子包下任意类中的任意方法
execution(* grp.pm..*Controller.*(..))	匹配 grp.pm 包下任意子孙包中以 "Controller" 结尾的类中的所有方法
* com.wmx..*Controller*.*(..)) 	com.wmx 包及其子包下面类名包含'Controller'的任意类中的任意方法
* com.wmx.*.controller.*.*(..))	第一二层包名为 com.wmx ，第三层包名任意，第4层包名为 controller 下面的任意类中的任意方法

@Pointcut("@annotation(xx)")  拦截拥有指定注解的方法



org.aspectj.lang.annotation.Around
org.aspectj.lang.annotation.Before
org.aspectj.lang.annotation.After
org.aspectj.lang.annotation.AfterReturning
org.aspectj.lang.annotation.AfterThrowing

## 执行顺序
org.aspectj.lang.annotation.Around
org.aspectj.lang.annotation.Before
org.aspectj.lang.annotation.Around
org.aspectj.lang.annotation.After
org.aspectj.lang.annotation.AfterReturning
org.aspectj.lang.annotation.AfterThrowing

```java
@Component
@Aspect
public class MyAop {
    @Pointcut(value = "@annotation(com.example.demo.MyAnnotation)")
    @Order(value = 1)
    public void aspectPointcut1() {
    }

    // 第一个*代表返回类型不限
    // 第二个*代表所有类
    // 第三个*代表所有方法
    // (..) 代表参数不限
    @Pointcut("execution(* com.example.demo.controller.*.*(..))")
    @Order(value = 2)
    public void aspectPointcut2() {
    }

    @Before(value = "aspectPointcut1()")
    public void doBefore1() {
        System.out.println("doBefore1");
    }

    @Before(value = "aspectPointcut2()")
    public void doBefore2() {
        System.out.println("doBefore2");
    }
}
```

___
# 3.业务代码
```java
    @MyAnnotation
    @RequestMapping(path = "/test")
    public String test() {
        return "hello";
    }
```

