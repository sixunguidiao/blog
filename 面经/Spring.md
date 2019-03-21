# Spring

## 1、说说 Spring AOP 的实现原理

### 概览

AOP 广泛应用于处理一些具有横切性质的系统级服务，AOP 的出现是对 OOP 的良好补充，它使得开发者能用更优雅的方式处理具有横切性质的服务。不管是那种 AOP 实现，不论是 AspectJ、还是 Spring AOP，它们都需要动态地生成一个 AOP 代理类，区别只是生成 AOP 代理类的时机不同：**AspectJ 采用编译时生成 AOP 代理类，因此具有更好的性能，但需要使用特定的编译器进行处理；而 Spring AOP 则采用运行时生成 AOP 代理类，因此无需使用特定编译器进行处理。由于 Spring AOP 需要在每次运行时生成 AOP 代理，因此性能略差一些。**

### Spring AOP 原理剖析

通过前面介绍可以知道：AOP 代理其实是由 AOP 框架动态生成的一个对象，该对象可作为目标对象使用。AOP 代理包含了目标对象的全部方法，但 AOP 代理中的方法与目标对象的方法存在差异：AOP 方法在特定切入点添加了增强处理，并回调了目标对象的方法。

AOP 代理所包含的方法与目标对象的方法示意图如下图所示。

![å¾ 3.AOP ä"£ççæ¹æ³ä¸ç®æ å¯¹è±¡çæ¹æ³](https://www.ibm.com/developerworks/cn/java/j-lo-springaopcglib/image007.gif)

Spring 的 AOP 代理由 Spring 的 IoC 容器负责生成、管理，其依赖关系也由 IoC 容器负责管理。因此，AOP 代理可以直接使用容器中的其他 Bean 实例作为目标，这种关系可由 IoC 容器的依赖注入提供。

纵观 AOP 编程，其中需要程序员参与的只有 3 个部分：

- 定义普通业务组件。
- 定义切入点，一个切入点可能横切多个业务组件。
- 定义增强处理，增强处理就是在 AOP 框架为普通业务组件织入的处理动作。

上面 3 个部分的第一个部分是最平常不过的事情，无须额外说明。那么进行 AOP 编程的关键就是定义切入点和定义增强处理。一旦定义了合适的切入点和增强处理，AOP 框架将会自动生成 AOP 代理，而 AOP 代理的方法大致有如下公式：

**代理对象的方法 = 增强处理 + 被代理对象的方法**

在上面这个业务定义中，不难发现 Spring AOP 的实现原理其实很简单：AOP 框架负责动态地生成 AOP 代理类，这个代理类的方法则由 Advice 和回调目标对象的方法所组成。

对于前面提到的图 2 所示的软件调用结构：当方法 1、方法 2、方法 3 ……都需要去调用某个具有“横切”性质的方法时，传统的做法是程序员去手动修改方法 1、方法 2、方法 3 ……、通过代码来调用这个具有“横切”性质的方法，但这种做法的可扩展性不好，因为每次都要改代码。

于是 AOP 框架出现了，AOP 框架则可以“动态的”生成一个新的代理类，而这个代理类所包含的方法 1、方法 2、方法 3 ……也增加了调用这个具有“横切”性质的方法——但这种调用由 AOP 框架自动生成的代理类来负责，因此具有了极好的扩展性。程序员无需手动修改方法 1、方法 2、方法 3 的代码，程序员只要定义切入点即可—— AOP 框架所生成的 AOP 代理类中包含了新的方法 1、访法 2、方法 3，而 AOP 框架会根据切入点来决定是否要在方法 1、方法 2、方法 3 中回调具有“横切”性质的方法。

简而言之：AOP 原理的奥妙就在于动态地生成了代理类，这个代理类实现了图 2 的调用——这种调用无需程序员修改代码。接下来介绍的 CGLIB 就是一个代理生成库，下面介绍如何使用 CGLIB 来生成代理类。

### Spring AOP 框架对 AOP 代理类的处理原则

Spring AOP 框架对 AOP 代理类的处理原则是：**如果目标对象的实现类实现了接口，Spring AOP 将会采用 JDK 动态代理来生成 AOP 代理类；如果目标对象的实现类没有实现接口，Spring AOP 将会采用 CGLIB 来生成 AOP 代理类**——不过这个选择过程对开发者完全透明、开发者也无需关心。

## 2、Spring bean 的作用域

- ### singleton——唯一 bean 实例

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

- ###  prototype——每次请求都会创建一个新的 bean 实例

当一个 bean 的作用域为 prototype 时，表示一个 bean 定义对应多个对象实例。prototype 作用域的 bean 会导致在每次对该 bean 请求 (将其注入到另一个 bean 中，或者以程序的方式调用容器的 getBean() 方法) 时都会创建一个新的 bean 实例。prototype 是原型类型，它在我们创建容器的时候并没有实例化，而是当我们获取 bean 的时候才会去创建一个对象，而且我们每次获取到的对象都不是同一个对象。**根据经验，对有状态的 bean 应该使用 prototype 作用域，而对无状态的 bean 则应该使用 singleton 作用域**。在 XML 中将 bean 定义成 prototype ，可以这样配置：

```xml
<bean id="account" class="com.foo.DefaultAccount" scope="prototype"/>  
 或者
<bean id="account" class="com.foo.DefaultAccount" singleton="false"/> 
```

通过 `@Scope` 注解的方式实现就不做演示了。

- ### request——每一次 HTTP 请求都会产生一个新的 bean，该 bean 仅在当前 HTTP request 内有效

request 只适用于 Web 程序，每一次 HTTP 请求都会产生一个新的 bean，同时该 bean 仅在当前 HTTP request 内有效，当请求结束后，该对象的生命周期即告结束。在 XML 中将 bean 定义成 request ，可以这样配置：

```xml
<bean id="loginAction" class=cn.csdn.LoginAction" scope="request"/>
```

- ### session——每一次 HTTP 请求都会产生一个新的 bean，该 bean 仅在当前 HTTP session 内有效

session 只适用于 Web 程序，session 作用域表示针对每一次 HTTP 请求都会产生一个新的 bean，该 bean 仅在当前 HTTP session 内有效。与 request 作用域一样，可以根据需要放心的更改所创建实例的内部状态，而别的 HTTP session 中根据 userPreferences 创建的实例，将不会看到这些特定于某个 HTTP session 的状态变化。当HTTP session 最终被废弃的时候，在该 HTTP session 作用域内的 bean 也会被废弃掉。

```xml
<bean id="userPreferences" class="com.foo.UserPreferences" scope="session"/>
```

- ### globalSession (*)

global session 作用域类似于标准的 HTTP session 作用域，不过仅仅在基于 portlet 的 web 应用中才有意义。Portlet 规范定义了全局 Session 的概念，它被所有构成某个 portlet web 应用的各种不同的 portlet 所共享。在global session 作用域中定义的 bean 被限定于全局 portlet Session 的生命周期范围内。

```xml
<bean id="user" class="com.foo.Preferences "scope="globalSession"/>
```

## 3、bean 的生命周期

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

需要注意的是自定义的 init-method 和 post-method 方法可以抛异常但是不能有参数。

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

[SpringBean](https://github.com/Snailclimb/JavaGuide/blob/master/%E4%B8%BB%E6%B5%81%E6%A1%86%E6%9E%B6/SpringBean.md#%E5%89%8D%E8%A8%80)

## 3、bean 的加载过程 (*)

1. 转换 beanName

要知道平时开发中传入的参数 name 可能只是别名，也可能是 FactoryBean，所以需要进行解析转换，一般会进行以下解析：
 （1）消除修饰符，比如 name="&test"，会去除 & 使 name="test"；
 （2）取 alias 表示的最后的 beanName，比如别名 test01 指向名称为 test02 的 bean 则返回 test02。

2. 从缓存中加载实例

实例在 Spring 的同一个容器中只会被创建一次，后面再想获取该 bean 时，就会尝试从缓存中获取；如果获取不到的话再从 singletonFactories 中加载。

3. 实例化 bean

缓存中记录的 bean 一般只是最原始的 bean 状态，这时就需要对 bean 进行实例化。如果得到的是 bean 的原始状态，但又要对 bean 进行处理，这时真正需要的是工厂 bean 中定义的 factory-method 方法中返回的 bean。

4. 检测 parentBeanFacotory

从源码可以看出如果缓存中没有数据会转到父类工厂 parentBeanFacotory 去加载。

5. 存储 XML 配置文件的 GernericBeanDefinition 转换成 RootBeanDefinition

XML 配置文件中读取到的 bean 信息是存储在 GernericBeanDefinition 中的，但 Bean 的后续处理是针对于 RootBeanDefinition 的，所以需要转换后才能进行后续操作。

6. 初始化依赖的 bean

这里应该比较好理解，就是 bean 中可能依赖了其他 bean 属性，在初始化 bean 之前会先初始化这个 bean 所依赖的 bean 属性。

7. 创建 bean

Spring 容器根据不同 scope 创建 bean 实例。

[【Spring】详解Spring中Bean的加载](https://www.jianshu.com/p/5fd1922ccab1)

## 4、Spring 循环依赖

### 1. 构造器参数循环依赖

Spring 容器会将每一个正在创建的 Bean 标识符放在一个“当前创建 Bean 池”中，Bean 标识符在创建过程中将一直保持在这个池中，因此如果在创建 Bean 过程中发现自己已经在“当前创建 Bean 池”里时将抛出 
BeanCurrentlyInCreationException 异常表示循环依赖；而对于创建完毕的 Bean 将从“当前创建 Bean 池”中清除掉。

首先我们先初始化三个 Bean。

```java
public class StudentA {
 
    private StudentB studentB ;
 
    public void setStudentB(StudentB studentB) {
        this.studentB = studentB;
    }
 
    public StudentA() {
    }
    
    public StudentA(StudentB studentB) {
        this.studentB = studentB;
    }
}
```

```java
public class StudentB {
 
    private StudentC studentC ;
 
    public void setStudentC(StudentC studentC) {
        this.studentC = studentC;
    }
    
    public StudentB() {
    }
 
    public StudentB(StudentC studentC) {
        this.studentC = studentC;
    }
}
```

```java
public class StudentC {
 
    private StudentA studentA ;
 
    public void setStudentA(StudentA studentA) {
        this.studentA = studentA;
    }
 
    public StudentC() {
    }
 
    public StudentC(StudentA studentA) {
        this.studentA = studentA;
    }
}
```

OK，上面是很基本的3个类，，StudentA 的有参构造是 StudentB，StudentB 的有参构造是 StudentC，StudentC 的有参构造是 StudentA ，这样就产生了一个循环依赖的情况，

我们都把这三个 Bean 交给 Spring 管理，并用有参构造实例化。

```xml
<bean id="a" class="com.zfx.student.StudentA">
    <constructor-arg index="0" ref="b"></constructor-arg>
</bean>
<bean id="b" class="com.zfx.student.StudentB">
    <constructor-arg index="0" ref="c"></constructor-arg>
</bean>
<bean id="c" class="com.zfx.student.StudentC">
    <constructor-arg index="0" ref="a"></constructor-arg>
</bean> 
```

执行结果报错信息为：

```
Caused by: org.springframework.beans.factory.BeanCurrentlyInCreationException: 
	Error creating bean with name 'a': Requested bean is currently in creation: Is there an unresolvable circular reference?
```

如果大家理解开头那句话的话，这个报错应该不惊讶，Spring 容器先创建单例 StudentA，StudentA 依赖 StudentB，然后将 A 放在“当前创建 Bean 池”中，此时创建 StudentB，StudentB 依赖 StudentC，然后将 B 放在“当前创建 Bean 池”中，此时创建 StudentC，StudentC 又依赖 StudentA，但是此时 StudentA 已经在池中，所以会报错，因为在池中的 Bean 都是未初始化完的，所以会依赖错误 (初始化完的 Bean 会从池中移除)

### 2. setter 方式单例，默认方式

如果要说 setter 方式注入的话，我们最好先看一张 Spring 中 Bean 实例化的图

![img](https://camo.githubusercontent.com/a3d4415162d30d4659779f6db3717f9a68fd3c97/687474703a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f31382d392d31372f353439363430372e6a7067)

如图中前两步骤得知：**Spring 是先将 Bean 对象实例化之后再设置对象属性的**

修改配置文件为 setter 方式注入。

```xml
<!--scope="singleton"(默认就是单例方式)  -->
<bean id="a" class="com.zfx.student.StudentA" scope="singleton">
    <property name="studentB" ref="b"></property>
</bean>
<bean id="b" class="com.zfx.student.StudentB" scope="singleton">
    <property name="studentC" ref="c"></property>
</bean>
<bean id="c" class="com.zfx.student.StudentC" scope="singleton">
    <property name="studentA" ref="a"></property>
</bean>
```

运行，发现没有报异常。

为什么用 setter 方式就不报错了呢 ？

我们结合上面那张图看，Spring 先实例化 Bean 对象 ，此时 Spring 会将这个实例化结束的对象放到一个 Map 中，并且 Spring 提供了获取这个未设置属性的实例化对象引用的方法。结合我们的实例来看，当 Spring 实例化了 StudentA，StudentB，StudentC 后，紧接着会去设置对象的属性，此时 StudentA 依赖 StudentB，就会去 Map 中取出存在里面的单例 StudentB 对象，以此类推，不会出来循环的问题。

### 3. setter 方式 + prototype (原型模式)

修改配置文件为：

```xml
<bean id="a" class="com.zfx.student.StudentA" scope="prototype">
    <property name="studentB" ref="b"></property>
</bean>
<bean id="b" class="com.zfx.student.StudentB" scope="prototype">
    <property name="studentC" ref="c"></property>
</bean>
<bean id="c" class="com.zfx.student.StudentC" scope="prototype">
    <property name="studentA" ref="a"></property>
</bean>
```

打印结果：

```
Caused by: org.springframework.beans.factory.BeanCurrentlyInCreationException: 
	Error creating bean with name 'a': Requested bean is currently in creation: Is there an unresolvable circular reference?
```

为什么原型模式就报错了呢 ？

对于 prototype 作用域的 Bean，Spring 容器无法完成依赖注入，因为 prototype 作用域的 Bean，**Spring容器不进行缓存**，因此无法提前暴露一个创建中的 Bean。

[Spring循环依赖的三种方式](https://blog.csdn.net/u010644448/article/details/59108799)

## 5、 Spring MVC 请求的处理流程

![1553007418195](C:\Users\guidiao\AppData\Roaming\Typora\typora-user-images\1553007418195.png)

在请求离开浏览器时，会带有用户所请求内容的信息，至少会包含请求的 URL。但是还可能带有其他的信息，例如用户提交的表单信息。

请求旅程的第一站是 Spring 的 DispatcherServlet。与大多数基于 Java 的 Web 框架一样，Spring MVC 所有的请求都会通过一个前端控制器 (front controller) Servlet。前端控制器是常用的 Web 应用程序模式，在这里一个单实例的 Servlet 将请求委托给应用程序的其他组件来执行实际的处理。在 Spring MVC 中，DispatcherServlet 就是前端控制器。

DispatcherServlet 的任务是将请求发送给 Spring MVC 控制器 controller。控制器是一个用于处理请求的 Spring 组件。在典型的应用程序中可能会有多个控制器，DispatcherServlet 需要知道应该将请求发送给哪个控制器。所以 DispatcherServlet 以会查询一个或多个处理器映射 (handler mapping) 来确定请求的下一站在哪里。处理器映射会根据请求所携带的 URL 信息来进行决策。

一旦选择了合适的控制器，DispatcherServlet 会将请求发送给选中的控制器。到了控制器，请求会卸下其负载 (用户提交的信息) 并耐心等待控制器处理这些信息。(实际上，设计良好的控制器本身只处理很少甚至不处理工作，而是将业务逻辑委托给一个或多个服务对象进行处理。)

控制器在完成逻辑处理后，通常会产生一些信息，这些信息需要返回给用户并在浏览器上显示。这些信息被称为模型 (model)。不过仅仅给用户返回原始的信息是不够的——这些信息需要以用户友好的方式进行格式化，一般会是 HTML。所以，信息需要发送给一个视图 (view)，通常会是 JSP。

控制器所做的最后一件事就是将模型数据打包，并且标示出用于渲染输出的视图名。它接下来会将请求连同模型和视图名发送回 DispatcherServlet。

这样，控制器就不会与特定的视图相耦合，传递给 DispatcherServlet 的视图名并不直接表示某个特定的 JSP。实际上，它甚至并不能确定视图就是 JSP。相反，它仅仅传递了一个逻辑名称，这个名字将会用来查找产生结果的真正视图。DispatcherServlet 将会使用视图解析器 (view resolver) 来将逻辑视图名匹配为一个特定的视图实现，它可能是也可能不是 JSP。

既然 DispatcherServlet 已经知道由哪个视图渲染结果，那请求的任务基本上也就完成了。它的最后一站是视图的实现 (可能是JSP) ，在这里它交付模型数据。请求的任务就完成了。视图将使用模型数据渲染输出，这个输出会通过响应对象传递给客户端 (不会像听上去那样硬编码) 。

[《Spring实战》]()

