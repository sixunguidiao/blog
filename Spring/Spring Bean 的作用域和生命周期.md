# Spring Bean 的作用域和生命周期

## Spring Bean 的作用域

### singleton——唯一 bean 实例

当一个 bean 的作用域为 singleton，那么容器中只会存在一个共享的 bean 实例，并且所有对 bean 的请求，只要 id 与该 bean 定义的相匹配，返回的都是该bean的同一个实例。

singleton 是单例类型 (对应于单例模式)，就是在创建容器时就同时自动创建了一个 bean 的对象，不管你是否使用，但我们可以指定 bean 节点的 `lazy-init=”true”` 来延迟初始化 bean，这时候，只有在第一次获取 bean 时才会初始化 bean，即第一次请求该 bean 时才初始化。 每次获取到的对象都是同一个对象。注意，singleton 作用域是 Spring 中的缺省作用域。要在 XML 中将 bean 定义成 singleton ，可以这样配置：

```xml
<bean id="ServiceImpl" class="cn.csdn.service.ServiceImpl" scope="singleton">
```

也可以通过 `@Scope` 注解的方式。(它可以显示指定 bean 的作用范围)

```java
@Service
@Scope("singleton")
public class ServiceImpl{

}
```

### prototype——每次请求都会创建一个新的 bean 实例

当一个 bean 的作用域为 prototype 时，表示一个 bean 定义对应多个对象实例。prototype 作用域的 bean 会导致在每次对该 bean 请求 (将其注入到另一个 bean 中，或者以程序的方式调用容器的 getBean() 方法) 时都会创建一个新的 bean 实例。prototype 是原型类型，它在我们创建容器的时候并没有实例化，而是当我们获取 bean 的时候才会去创建一个对象，而且我们每次获取到的对象都不是同一个对象。**根据经验，对有状态的 bean 应该使用 prototype 作用域，而对无状态的 bean 则应该使用 singleton 作用域**。在 XML 中将 bean 定义成 prototype ，可以这样配置：

```xml
<bean id="account" class="com.foo.DefaultAccount" scope="prototype"/>  
 或者
<bean id="account" class="com.foo.DefaultAccount" singleton="false"/> 
```

通过 `@Scope` 注解的方式实现就不做演示了。

### request——每一次 HTTP 请求都会产生一个新的 bean，该 bean 仅在当前 HTTP request 内有效

request 只适用于 Web 程序，每一次 HTTP 请求都会产生一个新的 bean，同时该 bean 仅在当前 HTTP request 内有效，当请求结束后，该对象的生命周期即告结束。在 XML 中将 bean 定义成 request ，可以这样配置：

```xml
<bean id="loginAction" class=cn.csdn.LoginAction" scope="request"/>
```

### session——每一次 HTTP 请求都会产生一个新的 bean，该 bean 仅在当前 HTTP session 内有效

session 只适用于 Web 程序，session 作用域表示针对每一次 HTTP 请求都会产生一个新的 bean，该 bean 仅在当前 HTTP session 内有效。与 request 作用域一样，可以根据需要放心的更改所创建实例的内部状态，而别的 HTTP session 中根据 userPreferences 创建的实例，将不会看到这些特定于某个 HTTP session 的状态变化。当 HTTP session 最终被废弃的时候，在该 HTTP session 作用域内的 bean 也会被废弃掉。

```xml
<bean id="userPreferences" class="com.foo.UserPreferences" scope="session"/>
```

### globalSession (*)

global session 作用域类似于标准的 HTTP session 作用域，不过仅仅在基于 portlet 的 web 应用中才有意义。Portlet 规范定义了全局 Session 的概念，它被所有构成某个 portlet web 应用的各种不同的 portlet 所共享。在global session 作用域中定义的 bean 被限定于全局 portlet Session 的生命周期范围内。

```xml
<bean id="user" class="com.foo.Preferences "scope="globalSession"/>
```

## bean 的生命周期

### initialization 和 destroy

有时我们需要在 Bean 属性值 set 好之后和 Bean 销毁之前做一些事情，比如检查 Bean 中某个属性是否被正常的设置好值了。Spring 框架提供了多种方法让我们可以在 Spring Bean 的生命周期中执行 initialization 和 pre-destroy 方法。

#### 1. 实现 InitializingBean 和 DisposableBean 接口

这两个接口都只包含一个方法。通过实现 InitializingBean 接口的 afterPropertiesSet() 方法可以在Bean属性值设置好之后做一些操作，实现 DisposableBean 接口的 destroy() 方法可以在销毁 Bean 之前做一些操作。

例子如下：

```java
public class GiraffeService implements InitializingBean,DisposableBean {
    @Override
    public void afterPropertiesSet() throws Exception {
        System.out.println("执行InitializingBean接口的afterPropertiesSet方法");
    }
    @Override
    public void destroy() throws Exception {
        System.out.println("执行DisposableBean接口的destroy方法");
    }
}
```

这种方法比较简单，但是不建议使用。因为这样会将 Bean 的实现和 Spring 框架耦合在一起。

#### 2. 在bean的配置文件中指定init-method和destroy-method方法

Spring 允许我们创建自己的 init 方法和 destroy 方法，只要在 Bean 的配置文件中指定 init-method 和 destroy-method 的值就可以在 Bean 初始化时和销毁之前执行一些操作。

例子如下：

```java
public class GiraffeService {
    //通过<bean>的destroy-method属性指定的销毁方法
    public void destroyMethod() throws Exception {
        System.out.println("执行配置的destroy-method");
    }
    //通过<bean>的init-method属性指定的初始化方法
    public void initMethod() throws Exception {
        System.out.println("执行配置的init-method");
    }
}
```

配置文件中的配置：

```xml
<bean name="giraffeService" class="com.giraffe.spring.service.GiraffeService" init-method="initMethod" destroy-method="destroyMethod">
</bean>
```

需要注意的是自定义的 init-method 和 destroy-method 方法可以抛异常但是不能有参数。

这种方式比较推荐，因为可以自己创建方法，无需将 Bean 的实现直接依赖于 spring 的框架。

#### 3. 使用 @PostConstruct 和 @PreDestroy 注解

除了 xml 配置的方式，Spring 也支持用 `@PostConstruct`和 `@PreDestroy`注解来指定 `init` 和 `destroy` 方法。这两个注解均在`javax.annotation` 包中。为了注解可以生效，需要在配置文件中定义 org.springframework.context.annotation.CommonAnnotationBeanPostProcessor 或 context:annotation-config。

例子如下：

```java
public class GiraffeService {
    @PostConstruct
    public void initPostConstruct(){
        System.out.println("执行PostConstruct注解标注的方法");
    }
    @PreDestroy
    public void preDestroy(){
        System.out.println("执行preDestroy注解标注的方法");
    }
}
```

配置文件：

```xml
<bean class="org.springframework.context.annotation.CommonAnnotationBeanPostProcessor" />
```

### 实现 *Aware 接口在 Bean 中使用 Spring 框架的一些对象

有些时候我们需要在 Bean 的初始化中使用 Spring 框架自身的一些对象来执行一些操作，比如获取 ServletContext 的一些参数，获取 ApplicaitionContext 中的 BeanDefinition 的名字，获取 Bean 在容器中的名字等等。为了让 Bean 可以获取到框架自身的一些对象，Spring 提供了一组名为 *Aware 的接口。

这些接口均继承于`org.springframework.beans.factory.Aware`标记接口，并提供一个将由 Bean 实现的 set*方法，Spring 通过基于 setter 的依赖注入方式使相应的对象可以被 Bean 使用。这些接口是利用观察者模式实现的。 介绍一些重要的 Aware 接口：

- **ApplicationContextAware**：获得 ApplicationContext 对象,可以用来获取所有 Bean definition 的名字。
- **BeanFactoryAware**：获得 BeanFactory 对象，可以用来检测 Bean 的作用域。
- **BeanNameAware**：获得 Bean 在配置文件中定义的名字。
- **ResourceLoaderAware**：获得 ResourceLoader 对象，可以获得 classpath 中某个文件。
- **ServletContextAware**：在一个 MVC 应用中可以获取 ServletContext 对象，可以读取 context 中的参数。
- **ServletConfigAware**：在一个 MVC 应用中可以获取 ServletConfig 对象，可以读取 config 中的参数。

### BeanPostProcessor

上面的 *Aware 接口是针对某个实现这些接口的 Bean 定制初始化的过程， Spring 同样可以针对容器中的所有 Bean，或者某些 Bean 定制初始化过程，只需提供一个实现 BeanPostProcessor 接口的类即可。 该接口中包含两个方法，postProcessBeforeInitialization 和 postProcessAfterInitialization。 postProcessBeforeInitialization 方法会在容器中的 Bean 初始化之前执行， postProcessAfterInitialization 方法在容器中的 Bean 初始化之后执行。

### 总结

Spring Bean的生命周期：

- Bean 容器找到配置文件中 Spring Bean 的定义。
- Bean 容器利用 Java Reflection API 创建一个 Bean 的实例。
- 如果涉及到一些属性值，利用 set 方法设置一些属性值。
- 如果 Bean 实现了 BeanNameAware 接口，调用 setBeanName() 方法，传入 Bean 的名字。
- 如果 Bean 实现了 BeanClassLoaderAware 接口，调用 setBeanClassLoader() 方法，传入 ClassLoader 对象的实例。
- 如果 Bean 实现了 BeanFactoryAware 接口，调用 setBeanFactory() 方法，传入 BeanFactory 对象的实例。
- 与上面的类似，如果实现了其他 *Aware 接口，就调用相应的方法。
- 如果有和加载这个 Bean 的 Spring 容器相关的 BeanPostProcessor 对象，执行 postProcessBeforeInitialization() 方法
- 如果 Bean 实现了 InitializingBean 接口，执行 afterPropertiesSet() 方法。
- 如果 Bean 在配置文件中的定义包含 init-method 属性 (或是 Bean 中有带 @PostConstruct 注解的方法)，执行指定的方法。
- 如果有和加载这个 Bean 的 Spring 容器相关的 BeanPostProcessor 对象，执行 postProcessAfterInitialization() 方法
- 当要销毁 Bean 的时候，如果 Bean 实现了 DisposableBean 接口，执行 destroy() 方法。
- 当要销毁 Bean 的时候，如果 Bean 在配置文件中的定义包含 destroy-method 属性 (或是 Bean 中有带 @PreDestroy 注解的方法)，执行指定的方法。

![img](https://camo.githubusercontent.com/a3d4415162d30d4659779f6db3717f9a68fd3c97/687474703a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f31382d392d31372f353439363430372e6a7067)

### 备注

#### 单例管理的对象

当 scope=”singleton”，即默认情况下，会在启动容器时（即实例化容器时）时实例化。但我们可以指定 Bean 节点的 lazy-init=”true” 来延迟初始化 bean，这时候，只有在第一次获取 bean 时才会初始化 bean，即第一次请求该 bean 时才初始化。

默认情况下，Spring 在读取 xml 文件的时候，就会创建对象。在创建对象的时候先调用构造器，然后调用 init-method 属性值中所指定的方法。对象在被销毁的时候，会调用 destroy-method 属性值中所指定的方法（例如调用Container.destroy()方法的时候）。

#### 非单例管理的对象

当 scope=”prototype” 时，容器也会延迟初始化 bean，Spring 读取xml 文件的时候，并不会立刻创建对象，而是在第一次请求该 bean 时才初始化（如调用 getBean 方法时）。在第一次请求每一个 prototype 的 bean 时，Spring 容器都会调用其构造器创建这个对象，然后调用 init-method 属性值中所指定的方法。**对象销毁的时候，Spring 容器不会帮我们调用任何方法，因为是非单例，这个类型的对象有很多个，Spring 容器一旦把这个对象交给你之后，就不再管理这个对象了**。

不管何种作用域，容器都会调用所有对象的初始化生命周期回调方法。但对 prototype 而言，任何配置好的析构生命周期回调方法都将不会被调用。清除 prototype 作用域的对象并释放任何 prototype bean 所持有的资源，都是客户端代码的职责 (**让 Spring 容器释放 prototype 作用域的 bean 占用资源的一种可行方式是，通过使用 bean 的后置处理器，该处理器持有要被清除的 bean 的引用**)。谈及 prototype 作用域的 bean 时，在某些方面你可以将Spring容器的角色看作是 Java new 操作的替代者，任何迟于该时间点的生命周期事宜都得交由客户端来处理。

## 参考资料

[Spring Bean](https://github.com/Snailclimb/JavaGuide/blob/master/%E4%B8%BB%E6%B5%81%E6%A1%86%E6%9E%B6/SpringBean.md#%E5%89%8D%E8%A8%80)

