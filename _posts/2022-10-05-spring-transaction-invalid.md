---
layout: post
title: Spring事务失效，我总结了这6个主要原因
category: spring
tags: [spring,面试]
description: 关于 Spring 事务失效原因，我总结了这6个主要原因
keywords: Spring 事务,事务失效
---

你好，我是Weiki，一个践行终身学习的程序员。

提起事务，很多人自然而然想到 MySQL数据库的 ACID，而 Spring Transaction 作为业务层的事务，也是Java日常开发中必不可少的。然而在工作中，很多开发认为只要在类或者方法上加上 @Transactional注解就可以万事大吉了，
殊不知事务失效的场景有多种，一个疏忽就可能造成了线上事故，这篇文章我来和你聊一聊导致Spring 事务失效的6个主要原因，帮助你的日常工作避雷。

> 申明：本文是基于 Spring 5.3.21 版本，如果没有特殊说明，本文的数据库泛指 MySQL数据库

内容大纲：
![img.png](https://yuanjava.cn/assets/md/spring/spring-transaction-outline.png)


## 1.数据库不支持事务

Spring 事务是业务层的事务，其底层还是依赖于数据库本身的事务支持。比如 MySQL 数据库，MyISAM 引擎不支持事务而 InnoDB 引擎支持事务。所以，开发中如果需要使用事务，一定要确保你选择的数据库支持事务。

MySQL数据库查看表信息指令如下，信息中包含表的使用的引擎类型：

```mysql
show create table tablename\G
```
![img.png](https://yuanjava.cn/assets/md/mysql/sql-transaction.png)

从 MySQL 5.5.5版本开始，默认的数据库引擎已经从 MyISAM 更改成 InnoDB。所以，如果你的数据库低于这个版本，就得注意数据库或者表在创建的时候设置的引擎是 MyISAM 还是 InnoDB。


## 2.类不受 Spring管理

我们先看一段 Spring事务失效的代码：

```java
// @Service
public class UserServiceImpl implements UserService {

    private final UserRepository userRepository;

    public UserServiceImpl(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
    /* user{ "name": "name1", "age": 18 } */
    @Transactional
    @Override
    public void addUser(User user) {
        userRepository.save(user);
        // 模拟异常，数据库的记录应该回滚
        throw new RuntimeException();
    }
}
```
如上代码，将 UserServiceImpl类中的 @Service注解注释后，UserServiceImpl类就不受 Spring容器管理，导致事务失效！

事务为什么会失效？

这是因为 Spring事务是由 AOP机制实现的，AOP机制的本质就是动态代理，更直白的说：从 Spring IOC容器获取 bean时，Spring会为目标类创建代理，从而支持事务的，核心源码如下：

```java
public abstract class AbstractAutoProxyCreator extends ProxyProcessorSupport
  implements SmartInstantiationAwareBeanPostProcessor, BeanFactoryAware {

    protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
        if (StringUtils.hasLength(beanName) && this.targetSourcedBeans.contains(beanName)) {
            return bean;
        }
        if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
            return bean;
        }
        if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {
            this.advisedBeans.put(cacheKey, Boolean.FALSE);
            return bean;
        }

        // Create proxy if we have advice.
        Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
        if (specificInterceptors != DO_NOT_PROXY) {
            this.advisedBeans.put(cacheKey, Boolean.TRUE);
            // 创建 AOP代理
            Object proxy = createProxy(
                    bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
            this.proxyTypes.put(cacheKey, proxy.getClass());
            return proxy;
        }

        this.advisedBeans.put(cacheKey, Boolean.FALSE);
        return bean;
    }

  protected Object createProxy(Class<?> beanClass, @Nullable String beanName,
                               @Nullable Object[] specificInterceptors, TargetSource targetSource) {

    if (this.beanFactory instanceof ConfigurableListableBeanFactory) {
      AutoProxyUtils.exposeTargetClass((ConfigurableListableBeanFactory) this.beanFactory, beanName, beanClass);
    }

    ProxyFactory proxyFactory = new ProxyFactory();
    proxyFactory.copyFrom(this);
    // 部分代码省略
    // 创建 AOP 切面，将信息存放在代理中
    Advisor[] advisors = buildAdvisors(beanName, specificInterceptors);
    proxyFactory.addAdvisors(advisors);
    proxyFactory.setTargetSource(targetSource);
    customizeProxyFactory(proxyFactory);

    proxyFactory.setFrozen(this.freezeProxy);
    if (advisorsPreFiltered()) {
      proxyFactory.setPreFiltered(true);
    }
    // Use original ClassLoader if bean class not locally loaded in overriding class loader
    ClassLoader classLoader = getProxyClassLoader();
    if (classLoader instanceof SmartClassLoader && classLoader != beanClass.getClassLoader()) {
      classLoader = ((SmartClassLoader) classLoader).getOriginalClassLoader();
    }
    return proxyFactory.getProxy(classLoader);
  }
}

```
通过源码可以看出：事务需要的 Advisor等资源是在 Spring创建代理类时去创建的，因此，注释 UserServiceImpl类的 @Service注解，该类就不受 Spring容器管理，那么事务需要的 Advisor资源就无法生成，事务自然就失效了。

不过，需要说明的是，目前 Java业务开发几乎都采用了 Spring生态圈的框架，如果将 UserServiceImpl类的 @Service注解去掉：

假如 UserServiceImpl类是通过构造器注入到Controller层，则 IDEA会报错，编译通过不了，如下图：

![img.png](https://yuanjava.cn/assets/md/spring/spring-transaction-construct.png)

假如 UserServiceImpl类是通过 @Autowired字段依赖注入，运行时会出现空指针异常，导致整个业务逻辑异常中断。

所以，"类不受 Spring管理"这个原因在现实开发中发生的概率比较低，但是该 case可以帮助我们更好的了解 Spring事务机制，所以被小编作为一个主要原因。

## 3.事务方法不是 public

假如对 UserServiceImpl类做部分修改，将事务方法 addUser()修改成 protected，代码如下：

```java

@Service
public class UserServiceImpl /** implements UserService */{

    private final UserRepository userRepository;

    public UserServiceImpl(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    /* user{ "name": "name1", "age": 18 } */
    @Transactional
    //@Override
    protected void addUser(User user) {
        userRepository.save(user);
        // 模拟异常，数据库的记录应该回滚
        throw new RuntimeException();
    }
}
```
单测运行上面的代码，查询数据库中记录，如下图:

![img.png](https://yuanjava.cn/assets/md/spring/spring-transaction-public.png)

通过 client客户端查询user表数据发现：{ "name": "name1", "age": 18 } 记录存在数据库中，事务并未回滚，也就是事务失效了，不符合预期。

为什么事务会失效？

我们还是通过 Spring的官方核心源码来分析：

```java
public abstract class AbstractFallbackTransactionAttributeSource
  implements TransactionAttributeSource, EmbeddedValueResolverAware {

  // 获取事务属性
  @Override
  @Nullable
  public TransactionAttribute getTransactionAttribute(Method method, @Nullable Class<?> targetClass) {
      // 部分代码省略

      if (cached != null) {
        // 部分代码省略
      }
      else {
        // 调用 计算事务属性 方法
        // We need to work it out.
        TransactionAttribute txAttr = computeTransactionAttribute(method, targetClass);
        // 部分代码省略
        return txAttr;
      }
  }

     // 计算事务属性
    @Nullable
    protected TransactionAttribute computeTransactionAttribute(Method method, @Nullable Class<?> targetClass) {
        // 按照配置，不允许使用非公共方法。
        // Don't allow non-public methods, as configured.
        if (allowPublicMethodsOnly() && !Modifier.isPublic(method.getModifiers())) {
            return null;
        }

        // 部分代码省略
        return null;
    }
}
```
通过源码，我们可以很清晰的看出，getTransactionAttribute()获取事务属性方法会调用 computeTransactionAttribute()方法，而computeTransactionAttribute()方法的逻辑，判断当方法非 public，则直接返回null，因此事务属性都不存在，事务自然也就不生效了。


## 4.异常被业务代码 catch

我们再对 UserServiceImpl类做代码调整，将 addUser()方法里的业务逻辑加上 try-catch，代码如下：

```java

@Service
public class UserServiceImpl implements UserService {

    private final UserRepository userRepository;

    public UserServiceImpl(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    /* user{ "name": "name2", "age": 20 } */
    @Transactional
    @Override
    public void addUser(User user) {
       try{
         userRepository.save(user);
         // 模拟异常，数据库的记录应该回滚
         throw new RuntimeException();
       } catch (RuntimeException e) {
         log.error("addUser#error,user:{},e", user, e);
       }
    }
}
```
单测运行上面的代码，查询数据库中记录，如下图:

![img.png](https://yuanjava.cn/assets/md/spring/spring-transaction-try.png)

通过 client客户端查询user表数据发现：{ "name": "name2", "age": 20 } 记录存在数据库中，并未回滚，也就是事务失效了，不符合预期。

为什么事务没有回滚？

我们依旧来看看 Spring的官方核心源码：

```java
public abstract class TransactionAspectSupport implements BeanFactoryAware, InitializingBean {
  @Nullable
  protected Object invokeWithinTransaction(Method method, @Nullable Class<?> targetClass,
                                           final InvocationCallback invocation) throws Throwable {
      // 省略部分代码

      if (txAttr == null || !(ptm instanceof CallbackPreferringPlatformTransactionManager)) {
          Object retVal;
          try {
              // This is an around advice: Invoke the next interceptor in the chain.
              // This will normally result in a target object being invoked.
              retVal = invocation.proceedWithInvocation();
          } catch (Throwable ex) {
              // target invocation exception
              // 回滚事务 是在 spring 的catch中处理，也就是说，如果Spring catch不到对应的异常，就不会进入回滚事务的逻辑
              completeTransactionAfterThrowing(txInfo, ex);
              throw ex;
          } finally {
              cleanupTransactionInfo(txInfo);
          }
          // 省略部分代码

          // 提交事务
          commitTransactionAfterReturning(txInfo);
          return retVal;
      }
  }

  /**
   * Handle a throwable, completing the transaction.
   * We may commit or roll back, depending on the configuration.
   * @param txInfo information about the current transaction
   * @param ex throwable encountered
   */
  protected void completeTransactionAfterThrowing(@Nullable TransactionInfo txInfo, Throwable ex) {
    if (txInfo != null && txInfo.getTransactionStatus() != null) {
       // 省略部分代码
      if (txInfo.transactionAttribute != null && txInfo.transactionAttribute.rollbackOn(ex)) {
        try {
            // 回滚事务
          txInfo.getTransactionManager().rollback(txInfo.getTransactionStatus());
        }
        catch (TransactionSystemException ex2) {
          logger.error("Application exception overridden by rollback exception", ex);
          ex2.initApplicationException(ex);
          throw ex2;
        }
        catch (RuntimeException | Error ex2) {
          logger.error("Application exception overridden by rollback exception", ex);
          throw ex2;
        }
      }
        // 省略部分代码
    }
  }
}
```
通过源码，我们可以很清晰的看出，在 invokeWithinTransaction() 方法中，当 Spring catch到 Throwable异常，就会调用 completeTransactionAfterThrowing()方法进行事务回滚的逻辑。但是，在 UserServiceImpl类的业务代码中直接把异常catch住了，Spring自然就 catch不到异常，因此事务回滚的逻辑就不会执行，事务就失效了。

所以，在日常开发中，当我们在 try-catch做完一些常规操作（比如异常日志记录和监控打点）后，一定要记得把异常抛出来（特别注意抛出的异常一定要是 Throwable或其子类，原因在下文会分析），这样 Spring才能感知异常，回滚事务。


## 5.rollbackFor属性配置错误

我们接着对 UserServiceImpl类做修改，在 @Transactional注解里指定 rollbackFor属性，比如：@Transactional(rollbackFor = Error.class)，代码如下：

```java
@Service
public class UserServiceImpl implements UserService {

    private final UserRepository userRepository;

    public UserServiceImpl(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    /* user{ "name": "name3", "age": 30 } */
    @Transactional(rollbackFor = Error.class)
    @Override
    public void addUser(User user) {
      userRepository.save(user);
      // 模拟异常，数据库的记录应该回滚
      throw new Exception();
    }
}
```
单测运行上面的代码后，查询数据库中记录，如下图:

![img.png](https://yuanjava.cn/assets/md/spring/spring-transaction-rollback.png)

通过 client客户端查询user表数据发现：{ "name": "name3", "age": 30 } 记录存在数据库中，并未回滚，也就是事务失效了，不符合预期。

为什么事务没有回滚？

我们依旧来看看 Spring的官方核心源码：

```java

@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
public @interface Transactional {
  /**
   * Defines zero (0) or more exception classes, which must be subclasses of Throwable, indicating which exception types must cause a transaction rollback.
   * By default, a transaction will be rolled back on RuntimeException and Error but not on checked exceptions (business exceptions). See org.springframework.transaction.interceptor.DefaultTransactionAttribute.rollbackOn(Throwable) for a detailed explanation.
   * This is the preferred way to construct a rollback rule (in contrast to rollbackForClassName), matching the exception type, its subclasses, and its nested classes. See the class-level javadocs for further details on rollback rule semantics and warnings regarding possible unintentional matches.
   * @return
   */
  //  rollbackFor的异常必须是 Throwable的子类
  Class<? extends Throwable>[] rollbackFor() default {};
}
```
通过 Transactional注解源码，我们可以发现 rollbackFor属性指定的异常必须是 Throwable及其子类，并且在默认情况下，Spring对 RuntimeException 和 Error 两种异常会自动回滚事务，也就是说，如果业务抛出来的异常是 RuntimeException 和 Error类型，可以不需要通过 rollbackFor属性指定，Spring 默认会识别处理。

可是，UserServiceImpl类中 rollbackFor = Error.class，Error 明明是 Throwable的子类，为什么事务还是会失效？

这是因为，尽管业务代码指定了 rollbackFor = Error.class，但是代码中抛出的异常是 Exception(throw new Exception())，而 Exception 和 Error没有任何关系，也就是说，事务需要捕获到 Error才会回滚，可是你偏偏抛出一个和 Error不相关的 Exception异常，因此事务自然无效，不能回滚。


## 6.方法内部调用事务方法

我们接着对 UserServiceImpl类做修改，addUser()方法不加 @Transactional注解，而是在 addUser()内部调用一个加了 @Transactional注解的 doAddUser()方法，代码如下：

```java
@Service
public class UserServiceImpl implements UserService {

    private final UserRepository userRepository;

    public UserServiceImpl(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

  /* user{ "name": "name4", "age": 40 } */
  @Override
  public void addUser(User user){
    // 调用内部方法
    this.doAddUser(user);
  }

  @Transactional(rollbackFor = Error.class)
  public void doAddUser (User user){
    userRepository.save(user);
    // 模拟异常，数据库的记录应该回滚
    throw new RuntimeException();
  }
}
```
单测运行上面的代码，查询数据库中记录，如下图:

![img.png](https://yuanjava.cn/assets/md/spring/spring-transaction-inter.png)

通过 client客户端查询user表数据发现：{ "name": "name4", "age": 40 } 记录存在数据库中，并未回滚，事务失效，不符合预期。

为什么事务没有回滚？

从上文"事务方法不是 public" 的分析可以知道，事务是通过Spring AOP代理来实现的，而 doAddUser()内部事务方法其实是this对象调用的，而不是通过AOP代理来调用的，因此事务失效。


## 总结

本文列举了日常开发中 Spring事务失效常见的6个主要原因，或许在开发中大家还是会遇到其他形形色色的小众原因，但不管怎样，只要我们能去认真去分析 Spring事务相关的源码，很多问题就迎刃而解。


## 鸣谢
如果你觉得本文章对你有帮助，感谢转发给更多的好友，关注我：猿java，为你呈现更多的硬核文章。

<img src="https://yuanjava.cn/assets/img/pub.jpg" alt="drawing" style="width:300px;"/>

