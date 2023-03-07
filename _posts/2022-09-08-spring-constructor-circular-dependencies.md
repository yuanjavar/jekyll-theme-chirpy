---
layout: post
title: 源码解读：Spring如何解决构造器注入的循环依赖？
category: spring
tags: [spring]
excerpt: spring如何解决构造器注入的循环依赖？
keywords: spring,spring注入,spring constructor注入,spring 循坏依赖
---

你好，我是猿java。

Spring 循环依赖一般包含 构造器注入循环依赖 和字段注入（setter方式）循环依赖， 字段注入循环依赖，Spring 官方通过三层缓存解决。而今天分享的重点是：Spring 是如何解决构造器注入产生的循环依赖问题？

> 申明：本文源码 基于 springboot-2.7.0 、spring-5.3.20 和 JDK11
>
> 注意：从 springboot-2.7.0 开始，不再推荐 使用 /META-INF/spring.factories 文件所指定的配置类去加载Bean。

## 背景

前段时间，因部门同事遇到一个 Spring 循环依赖的问题，IDEA 错误信息如下：
```text
  ***************************
  APPLICATION FAILED TO START
  ***************************

  Description:

  The dependencies of some of the beans in the application context form a cycle:

  ┌─────┐
  |  orderService defined in file [./target/classes/cn/xxx/spring/OrderService.class]
  ↑     ↓
  |  userService defined in file [./target/classes/cn/xxx/spring/UserService.class]
  └─────┘

  Action:

  Relying upon circular references is discouraged and they are prohibited by default. Update your application to remove the dependency cycle between beans. As a last resort, it may be possible to break the cycle automatically by setting spring.main.allow-circular-references to true.

```

错误信息大体意思是：不鼓励依赖循环引用，默认情况下是禁止的。可以通过修改代码，删除 bean 之间的依赖循环。或者通过将 spring.main.allow-circular-references 设置为 true 来自动中断循环。

鉴于自己曾经也遇到过这个问题，因此把曾经整理的云笔记结合源码输出此文，希望帮助到同样遇坑的小伙伴。



## 什么是循环依赖

循环依赖是指：对象实例之间依赖关系构成一个闭环，通常分为：单个对象的自我依赖、两个对象的相互依赖、多个对象依赖成环。循环依赖抽象如下图：

![img.png](https://yuanjava.cn//assets/md/spring/circular-type.png)

### 单个对象的自我依赖

```java
// 变量依赖注入循环依赖，服务器启动会报错
@Component
public class UserService {
  @Autowired
  private UserService userService;
}
```

运行结果
```java
***************************
APPLICATION FAILED TO START
***************************

Description:

The dependencies of some of the beans in the application context form a cycle:

┌──->──┐
|  userService2 (field cn.yuanjava.service.UserService2 cn.yuanjava.service.UserService2.userService2)
└──<-──┘

```

```java
// 字段依赖注入循环依赖
@Component
public class OrderService {
  @Autowired
  private OrderService orderService;
}

// 构造器注入循环依赖
@Component
public class OrderService {

  private final OrderService orderService;

  public OrderService(){
    this.OrderService = orderService;
  }
}
```


```java
Error starting ApplicationContext. To display the conditions report re-run your application with 'debug' enabled.
 ERROR 3846 --- [           main] o.s.b.d.LoggingFailureAnalysisReporter   :

  ***************************
  APPLICATION FAILED TO START
  ***************************

  Description:
  The dependencies of some of the beans in the application context form a cycle:

  ┌─────┐
  |  orderService (field private cn.yuanjava.service.UserService cn.yuanjava.service.OrderService.userService)
  ↑     ↓
  |  userService (field private cn.yuanjava.service.OrderService cn.yuanjava.service.UserService.orderService)
  └─────┘

  Action:
  Relying upon circular references is discouraged and they are prohibited by default. Update your application to remove the dependency cycle between beans. As a last resort, it may be possible to break the cycle automatically by setting spring.main.allow-circular-references to true.

```

产生了这种循环依赖，说明代码真的很low，要自我检讨。

### 两个对象的相互依赖
从上文 OrderService 和 UserService 两个类的代码可以看出，在初始化 OrderService 类时，需要依赖 UserService，而 UserService 类未实例化，因此需要实例化 UserService 类，但是在初始化 UserService 类时发现它又依赖 OrderService 类，因此就产生了循环依赖，依赖关系可以抽象成下图：

![img.png](https://yuanjava.cn//assets/md/spring/spring-circular.png)

```java
// 字段依赖注入循环依赖，服务器能正常run
@Component
public class OrderService {
  @Autowired
  private UserService userService;
}

@Component
public class UserService {
  @Autowired
  private OrderService orderService;
}


// 构造器注入循环依赖，服务器启动会报错
@Component
public class OrderService {
  private final UserService userService;
  public OrderService(UserService userService){
    this.userService = userService;
  }
}

@Component
public class UserService {

  private final OrderService orderService;
  public UserService(OrderService orderService){
    this.orderService = orderService;
  }
}
```

### 多个对象的依赖成环

```java
// 变量依赖注入循环依赖，服务器启动会报错
@Component
public class UserService {
  @Autowired
  private OrderService orderService;
}

@Component
public class OrderService {
  @Autowired
  private GoodsService goodsService;
}

@Component
public class GoodsService {
  @Autowired
  private UserService userService;
}

// 构造器注入循环依赖，服务器启动会报错
@Component
public class OrderService {
  private final UserService userService;
  public OrderService(UserService userService){
    this.userService = userService;
  }
}

@Component
public class UserService {

  private final GoodsService goodsService;
  public UserService(GoodsService goodsService){
    this.goodsService = goodsService;
  }
}

@Component
public class GoodsService {

  private final OrderService orderService;
  public GoodsService(OrderService orderService){
    this.orderService = orderService;
  }
}
```
这种循环依赖比较隐蔽，多个对象依赖，最终成环。

```java
Error starting ApplicationContext. To display the conditions report re-run your application with 'debug' enabled.
ERROR 33185 --- [           main] o.s.b.d.LoggingFailureAnalysisReporter   :

***************************
APPLICATION FAILED TO START
***************************

Description:

The dependencies of some of the beans in the application context form a cycle:

┌─────┐
|  goodsService (field private cn.yuanjava.service.UserService cn.yuanjava.service.GoodsService.userService)
↑     ↓
|  userService (field private cn.yuanjava.service.OrderService cn.yuanjava.service.UserService.orderService)
↑     ↓
|  orderService (field private cn.yuanjava.service.GoodsService cn.yuanjava.service.OrderService.goodsService)
└─────┘


Action:

Relying upon circular references is discouraged and they are prohibited by default. Update your application to remove the dependency cycle between beans. As a last resort, it may be possible to break the cycle automatically by setting spring.main.allow-circular-references to true.


```



```java

@Service
public class UserService {
    private OrderService orderService;
    public UserService(OrderService orderService) {
        this.orderService = orderService;
    }
}

@Service
public class OrderService {
    private UserService userService;

    public OrderService(UserService userService) {
        this.userService = userService;
    }
}


Error starting ApplicationContext. To display the conditions report re-run your application with 'debug' enabled.
2023-02-19 14:36:13.133 ERROR 15885 --- [           main] o.s.b.d.LoggingFailureAnalysisReporter   :

***************************
APPLICATION FAILED TO START
***************************

Description:

The dependencies of some of the beans in the application context form a cycle:

┌─────┐
|  orderService defined in file [/Users/weiki/Desktop/workspace/yuan/yuanjava/build/classes/java/main/cn/yuanjava/service/OrderService.class]
↑     ↓
|  userService defined in file [/Users/weiki/Desktop/workspace/yuan/yuanjava/build/classes/java/main/cn/yuanjava/service/UserService.class]
└─────┘


Action:

Relying upon circular references is discouraged and they are prohibited by default. Update your application to remove the dependency cycle between beans. As a last resort, it may be possible to break the cycle automatically by setting spring.main.allow-circular-references to true.



```

## 如何解决循环依赖

### 3.1 修改代码

既然循环依赖是代码编写带来的，最彻底的方案是把出现循环依赖的代码重构，但是，重构代码的范围可能不可控，因此，对于测试等存在一定的回归成本，这是一种代价稍微大点的方案。

不过，代码出现循环依赖，在一定意义上（不是绝对哦）预示了 code smell：为什么会存在循环依赖？代码抽象是否合理？代码设计是否违背了 SOLID 原则？

### 3.2 使用字段依赖注入

曾经很长一段时间（Spring 3.0 以前的版本），字段依赖是比较主流的一种编程方式，因为这种方式编写方便简洁，而且 Spring 也利用三层缓存解决了循环依赖问题，但后面因 Spring 不推荐字段依赖注入方式，并且在 github上也可以发现大部分的开源软件也不采用这种方式了，所以该方案也仅供参考不推荐，改造代码如下：

```java
@Component
public class OrderService {
    @Autowired
    private UserService userService;

    public User getUser(){
        return userService.getUser();
    }
}

@Component
public class UserService {
  @Autowired
  private OrderService orderService;

  public Order getOrder(){
    return orderService.getOrder();
  }
}
```

**3. 使用 @Lazy 注解**

@Lazy 是 spring 3.0 提供的一个注解，用来表示是否要延迟初始化 bean，首先看下 @Lazy注解的源码：

```java
/**
 * Indicates whether a bean is to be lazily initialized.
 *
 * <p>May be used on any class directly or indirectly annotated with {@link
 * org.springframework.stereotype.Component @Component} or on methods annotated with
 * {@link Bean @Bean}.
 *
 * <p>If this annotation is not present on a {@code @Component} or {@code @Bean} definition,
 * eager initialization will occur. If present and set to {@code true}, the {@code @Bean} or
 * {@code @Component} will not be initialized until referenced by another bean or explicitly
 * retrieved from the enclosing {@link org.springframework.beans.factory.BeanFactory
 * BeanFactory}. If present and set to {@code false}, the bean will be instantiated on
 * startup by bean factories that perform eager initialization of singletons.
 *
 * <p>If Lazy is present on a {@link Configuration @Configuration} class, this
 * indicates that all {@code @Bean} methods within that {@code @Configuration}
 * should be lazily initialized. If {@code @Lazy} is present and false on a {@code @Bean}
 * method within a {@code @Lazy}-annotated {@code @Configuration} class, this indicates
 * overriding the 'default lazy' behavior and that the bean should be eagerly initialized.
 *
 * <p>In addition to its role for component initialization, this annotation may also be placed
 * on injection points marked with {@link org.springframework.beans.factory.annotation.Autowired}
 * or {@link javax.inject.Inject}: In that context, it leads to the creation of a
 * lazy-resolution proxy for all affected dependencies, as an alternative to using
 * {@link org.springframework.beans.factory.ObjectFactory} or {@link javax.inject.Provider}.
 * Please note that such a lazy-resolution proxy will always be injected; if the target
 * dependency does not exist, you will only be able to find out through an exception on
 * invocation. As a consequence, such an injection point results in unintuitive behavior
 * for optional dependencies. For a programmatic equivalent, allowing for lazy references
 * with more sophistication, consider {@link org.springframework.beans.factory.ObjectProvider}.
 *
 * @author Chris Beams
 * @author Juergen Hoeller
 * @since 3.0
 * @see Primary
 * @see Bean
 * @see Configuration
 * @see org.springframework.stereotype.Component
 */
@Target({ElementType.TYPE, ElementType.METHOD, ElementType.CONSTRUCTOR, ElementType.PARAMETER, ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Lazy {

	/**
	 * Whether lazy initialization should occur.
	 */
	boolean value() default true;

}
```

从 @Lazy 注解的源码可以总结几点：

1. @Lazy 用来标识类是否需要延迟加载
2. @Lazy 可以作用在类上、方法上、构造器上、方法参数上、成员变量中
3. @Lazy 作用于类上时，通常与 @Component 及其衍生注解配合使用，@Lazy 注解作用于方法上时，通常与 @Bean 注解配合使用

因此，通过 @Lazy 解决构造器循环依赖的代码改造如下：

```java
@Component
public class UserService {

  private final OrderService orderService;

  @Lazy
  public UserService(OrderService orderService){
    this.orderService = orderService;
  }
  // 或者
  public UserService(@Lazy OrderService orderService){
    this.orderService = orderService;
  }

  public Order getOrder(){
    return orderService.getOrder();
  }
}
```

## @Lazy 原理剖析

本文使用的是 Springboot 2.7 启动的，因此整体思路是：Springboot是如何启动 Spring IOC容器？如何加载 Bean？如何 处理 @Lazy注解？

源码查看足迹可以参考下面的类：

> Springboot 启动类 main() 调用 org.springframework.boot.SpringApplication#run()
>
> org.springframework.boot.SpringApplication#refreshContext()
>
> org.springframework.context.support.AbstractApplicationContext#refresh()
>
> org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#populateBean()
>
> org.springframework.beans.factory.annotation.AutowiredAnnotationBeanPostProcessor#postProcessProperties()
>
> org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#createBean
>
> org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#doCreateBean
>
> org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#createBeanInstance
>
> org.springframework.beans.factory.support.ConstructorResolver#autowireConstructor
>
> org.springframework.beans.factory.support.ConstructorResolver#resolvePreparedArguments
>
> org.springframework.beans.factory.support.ConstructorResolver#resolveAutowiredArgument
>
> org.springframework.beans.factory.config.AutowireCapableBeanFactory#resolveDependency()
>

这里摘取了处理构造器依赖的几个核心方法来解释@Lazy 如何解决循环依赖，因为 UserService类 构造器注入 OrderService 是强依赖关系，因此会经过 AbstractAutowireCapableBeanFactory#createBeanInstance() 中关于构造器逻辑代码:
```java
// org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#createBeanInstance
class AbstractAutowireCapableBeanFactory{
    // Create a new instance for the specified bean, using an appropriate instantiation strategy: factory method, constructor autowiring, or simple instantiation.
    // 使用适当的实例化策略为指定的 bean 创建一个新实例：工厂方法、构造函数自动装配或简单实例化。
    protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) {
      // Candidate constructors for autowiring?
      Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);
      if (ctors != null || mbd.getResolvedAutowireMode() == AUTOWIRE_CONSTRUCTOR ||
          mbd.hasConstructorArgumentValues() || !ObjectUtils.isEmpty(args)) {
        return autowireConstructor(beanName, mbd, ctors, args);
      }
		}
}
```

在 autowireConstructor(beanName, mbd, ctors, args) 方法会调用 ConstructorResolver#resolvePreparedArguments()，再进入ConstructorResolver#resolveAutowiredArgument()，
再进入DefaultListableBeanFactory#resolveDependency()，
resolveDependency()方法的 getAutowireCandidateResolver().getLazyResolutionProxyIfNecessary 逻辑就是针对Lazy情况进行处理：
判断构造器参数是有@Lazy注解，有则通过buildLazyResolutionProxy 生成代理对象，无则直接返回beanName。而在buildLazyResolutionProxy()里会生成 一个TargetSource对象来和代理对象相关联。部分源码如下：

```java
// org.springframework.beans.factory.support.DefaultListableBeanFactory#resolveDependency
public class DefaultListableBeanFactory{
  public Object resolveDependency(DependencyDescriptor descriptor, @Nullable String requestingBeanName,
                                  @Nullable Set<String> autowiredBeanNames, @Nullable TypeConverter typeConverter) throws BeansException {

    // 此处省略部分代码
    if (Optional.class == descriptor.getDependencyType()) {
    } else {
       // 处理 Lazy 逻辑
      Object result = getAutowireCandidateResolver().getLazyResolutionProxyIfNecessary(
        descriptor, requestingBeanName);
      if (result == null) {
        result = doResolveDependency(descriptor, requestingBeanName, autowiredBeanNames, typeConverter);
      }
      return result;
    }
  }
}

// org.springframework.context.annotation.ContextAnnotationAutowireCandidateResolver#getLazyResolutionProxyIfNecessary
public class ContextAnnotationAutowireCandidateResolver extends QualifierAnnotationAutowireCandidateResolver {

    @Override
    @Nullable
    public Object getLazyResolutionProxyIfNecessary(DependencyDescriptor descriptor, @Nullable String beanName) {
        // 判断注解是否有@Lazy，有则通过buildLazyResolutionProxy 生成代理对象，没有则直接返回beanName
        return (isLazy(descriptor) ? buildLazyResolutionProxy(descriptor, beanName) : null);
    }

  protected boolean isLazy(DependencyDescriptor descriptor) {
    for (Annotation ann : descriptor.getAnnotations()) {
      Lazy lazy = AnnotationUtils.getAnnotation(ann, Lazy.class);
      if (lazy != null && lazy.value()) {
        return true;
      }
    }
    MethodParameter methodParam = descriptor.getMethodParameter();
    if (methodParam != null) {
      Method method = methodParam.getMethod();
      if (method == null || void.class == method.getReturnType()) {
        Lazy lazy = AnnotationUtils.getAnnotation(methodParam.getAnnotatedElement(), Lazy.class);
        if (lazy != null && lazy.value()) {
          return true;
        }
      }
    }
    return false;
  }
}
```

通过上面核心代码的解读，我们可以知道，构造器(参数)增加 @Lazy 注解后，Spring不会去初始化参数对应类的实例，而是返回它的一个代理对象，解决了循环依赖问题，逻辑可以抽象为下图：

![img.png](https://yuanjava.cn//assets/md/spring/spring-inject-proxy.png)

尽管循环依赖的问题解决了，但是，UserService类 依赖的只是OrderService的一个代理对象。因此，我们自然会好奇：当调用orderService.getOrder()时，spring是如何找到 OrderService 的真实对象呢？

从上文知道，注入给UserService类的是一个代理，说起代理就不得不说起Spring AOP机制，它就是通过动态代理实现的(JDK动态代理 和 CGLib动态代理)。
因为OrderService并非接口，因此不能使用 JDK动态代理，只能通过 CGLib进行代理，CGLib源码如下:
```java
// org.springframework.aop.framework.CglibAopProxy.DynamicAdvisedInterceptor#intercept
class CglibAopProxy implements AopProxy, Serializable {
    private static class DynamicAdvisedInterceptor implements MethodInterceptor, Serializable {
      public Object intercept(Object proxy, Method method, Object[] args, MethodProxy methodProxy) throws Throwable {
        Object oldProxy = null;
        boolean setProxyContext = false;
        Object target = null;
        TargetSource targetSource = this.advised.getTargetSource();
        try {
          if (this.advised.exposeProxy) {
            // Make invocation available if necessary.
            oldProxy = AopContext.setCurrentProxy(proxy);
            setProxyContext = true;
          }
          // Get as late as possible to minimize the time we "own" the target, in case it comes from a pool...
          // 获取被代理的对象
          target = targetSource.getTarget();
          Class<?> targetClass = (target != null ? target.getClass() : null);
          List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);
          Object retVal;
          // Check whether we only have one InvokerInterceptor: that is,
          // no real advice, but just reflective invocation of the target.
          if (chain.isEmpty() && CglibMethodInvocation.isMethodProxyCompatible(method)) {
            // We can skip creating a MethodInvocation: just invoke the target directly.
            // Note that the final invoker must be an InvokerInterceptor, so we know
            // it does nothing but a reflective operation on the target, and no hot
            // swapping or fancy proxying.
            Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
            // 通过反射调用被代理对象的方法
            retVal = invokeMethod(target, method, argsToUse, methodProxy);
          }
          else {
            // We need to create a method invocation...
            retVal = new CglibMethodInvocation(proxy, target, method, args, targetClass, chain, methodProxy).proceed();
          }
          retVal = processReturnType(proxy, target, method, retVal);
          return retVal;
        }catch (Exception e){}
      }
    }
}

```

这里抽取了CGLib动态代理核心的3步：

```text
// 此处的TargetSource 和 上文 buildLazyResolutionProxy() 构建的TargetSource 关联
1. TargetSource targetSource = this.advised.getTargetSource();

// 获取被代理的对象target
2. target = targetSource.getTarget();

// 反射调用被代理对象的方法
3. retVal = invokeMethod(target, method, argsToUse, methodProxy);
```

通过CGLib核心的3步解释了，Spring中代理类是如何与真实对象进行关联，因此，orderService关联到真实对象可以抽象成下图：

![img.png](https://yuanjava.cn/assets/md/spring/spring-circular-cglib.png)

另外，我们通过3张 IDEA debugger 截图来佐证下：

![img.png](https://yuanjava.cn/assets/md/spring/spring-circular-cglib-debug.png)

![img.png](https://yuanjava.cn/assets/md/spring/spring-circular-cglib-debug2.png)

![img.png](https://yuanjava.cn/assets/md/spring/spring-circular-cglib-debug3.png)


## 总结

- Spring构造器注入循环依赖有3种解决办法：重构代码、字段依赖注入、@Lazy注解。强烈推荐 @Lazy注解；
- @Lazy注解 解决思路是：初始化时注入代理对象，真实调用时使用Spring AOP动态代理去关联真实对象，然后通过反射完成调用；
- @Lazy注解 加在构造器上，作用域为构造器所有参数，加在构造器某个参数上，作用域为该参数；
- @Lazy注解 作用在接口上，使用 JDK动态代理，作用在类上，使用 CGLib动态代理；


## 鸣谢
如果你觉得本文章对你有帮助，感谢转发给更多的好友，关注我：猿java，为你呈现更多的硬核文章。

<img src="https://yuanjava.cn/assets/img/pub.jpg" alt="drawing" style="width:300px;"/>

