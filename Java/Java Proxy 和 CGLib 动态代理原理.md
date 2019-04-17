# # Java Proxy 和 CGLib 动态代理原理

动态代理在 Java 中有着广泛的应用，比如 Spring AOP，Hibernate 数据查询、测试框架的后端 mock，RPC，Java 注解对象获取等。静态代理的代理关系在编译时就确定了，而动态代理的代理关系是在编译期确定的。静态代理实现简单，适合于代理类较少且确定的情况，而动态代理则给我们提供了更大的灵活性。今天我们来探讨 Java 中两种常见的动态代理方式：**JDK 原生动态代理和 CGLIB 动态代理**。

## JDK 原生动态代理

先从直观的示例说起，假设我们有一个接口 `Hello` 和一个简单实现 `HelloImp`：

```java
// 接口
interface Hello{
    String sayHello(String str);
}
// 实现
class HelloImp implements Hello{
    @Override
    public String sayHello(String str) {
        return "HelloImp: " + str;
    }
}
```

这是 Java 种再常见不过的场景，使用接口制定协议，然后用不同的实现来实现具体行为。假设你已经拿到上述类库，如果我们想通过日志记录对 `sayHello()` 的调用，使用静态代理可以这样做：

```java
// 静态代理方式
class StaticProxiedHello implements Hello{
    ...
    private Hello hello = new HelloImp();
    @Override
    public String sayHello(String str) {
        logger.info("You said: " + str);
        return hello.sayHello(str);
    }
}
```

上例中静态代理类 `StaticProxiedHello` 作为 `HelloImp` 的代理，实现了相同的 `Hello` 接口。用 Java 动态代理可以这样做：

1. 首先实现一个 InvocationHandler，方法调用会被转发到该类的 invoke() 方法。
2. 然后在需要使用 Hello 的时候，通过 JDK 动态代理获取 Hello 的代理对象。

```java
// Java Proxy
// 1. 首先实现一个InvocationHandler，方法调用会被转发到该类的invoke()方法。
class LogInvocationHandler implements InvocationHandler{
    ...
    private Hello hello;
    public LogInvocationHandler(Hello hello) {
        this.hello = hello;
    }
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        if("sayHello".equals(method.getName())) {
            logger.info("You said: " + Arrays.toString(args));
        }
        return method.invoke(hello, args);
    }
}

// 2. 然后在需要使用Hello的时候，通过JDK动态代理获取Hello的代理对象。
Hello hello = (Hello)Proxy.newProxyInstance(
    getClass().getClassLoader(), // 1. 类加载器
    new Class<?>[] {Hello.class}, // 2. 代理需要实现的接口，可以有多个
    new LogInvocationHandler(new HelloImp()));// 3. 方法调用的实际处理者
System.out.println(hello.sayHello("I love you!"));
```

运行上述代码输出结果：

```java
日志信息: You said: [I love you!]
HelloImp: I love you!
```

上述代码的关键是 `Proxy.newProxyInstance(ClassLoader loader, Class<?>[] interfaces, InvocationHandler handler)` 方法，该方法会根据指定的参数动态创建代理对象。三个参数的意义如下：

1. `loader`，指定代理对象的类加载器；
2. `interfaces`，代理对象需要实现的接口，可以同时指定多个接口；
3. `handler`，方法调用的实际处理者，代理对象的方法调用都会转发到这里。

`newProxyInstance()` 会返回一个实现了指定接口的代理对象，对该对象的所有方法调用都会转发给 `InvocationHandler.invoke()` 方法。理解上述代码需要对 Java 反射机制有一定了解。动态代理神奇的地方就是：

1. 代理对象是在程序运行时产生的，而不是编译期；
2. **对代理对象的所有接口方法调用都会转发到 InvocationHandler.invoke() 方法**，在 `invoke()` 方法里我们可以加入任何逻辑，比如修改方法参数，加入日志功能、安全检查功能等；之后我们通过某种方式执行真正的方法体，示例中通过反射调用了 Hello 对象的相应方法，还可以通过 RPC 调用远程方法。

> 注意：对于从 Object 中继承的方法，JDK Proxy 会把 `hashCode()`、`equals()`、`toString()` 这三个非接口方法转发给 `InvocationHandler`，其余的 Object 方法则不会转发。详见 [JDK Proxy官方文档](https://docs.oracle.com/javase/7/docs/api/java/lang/reflect/Proxy.html)。

Java 动态代理为我们提供了非常灵活的代理机制，但 Java 动态代理是基于接口的，如果对象没有实现接口我们该如何代理呢？CGLIB 登场。

## CGLIB 动态代理

CGLIB (Code Generation Library) 是一个基于 ASM 的字节码生成库，它允许我们在运行时对字节码进行修改和动态生成。CGLIB 通过继承方式实现代理。

来看示例，假设我们有一个没有实现任何接口的类 `HelloConcrete`：

```java
public class HelloConcrete {
    public String sayHello(String str) {
        return "HelloConcrete: " + str;
    }
}
```

因为没有实现接口该类无法使用 JDK 代理，通过 CGLIB 代理实现如下：

1. 首先实现一个 MethodInterceptor，方法调用会被转发到该类的 intercept() 方法。
2. 然后在需要使用 HelloConcrete 的时候，通过 CGLIB 动态代理获取代理对象。

```java
// CGLIB动态代理
// 1. 首先实现一个MethodInterceptor，方法调用会被转发到该类的intercept()方法。
class MyMethodInterceptor implements MethodInterceptor{
  ...
    @Override
    public Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy) throws Throwable {
        logger.info("You said: " + Arrays.toString(args));
        return proxy.invokeSuper(obj, args);
    }
}
// 2. 然后在需要使用HelloConcrete的时候，通过CGLIB动态代理获取代理对象。
Enhancer enhancer = new Enhancer();
enhancer.setSuperclass(HelloConcrete.class);
enhancer.setCallback(new MyMethodInterceptor());

HelloConcrete hello = (HelloConcrete)enhancer.create();
System.out.println(hello.sayHello("I love you!"));
```

运行上述代码输出结果：

```
日志信息: You said: [I love you!]
HelloConcrete: I love you!
```

上述代码中，我们通过 CGLIB 的 `Enhancer` 来指定要代理的目标对象，实际处理代理逻辑的对象，最终通过调用 `create()` 方法得到代理对象，**对这个对象所有非 final 方法的调用都会转发给 MethodInterceptor.intercept() 方法**，在 `intercept()` 方法里我们可以加入任何逻辑，比如修改方法参数，加入日志功能、安全检查功能等；通过调用 `MethodProxy.invokeSuper()` 方法，我们将调用转发给原始对象，具体到本例，就是 `HelloConcrete` 的具体方法。CGLIG中 MethodInterceptor 的作用跟 JDK 代理中的 `InvocationHandler` 很类似，都是方法调用的中转站。

> 注意：对于从 Object 中继承的方法，CGLIB 代理也会进行代理，如 `hashCode()`、`equals()`、`toString()` 等，但是 `getClass()`、`wait()` 等方法不会，因为它是 final 方法，CGLIB 无法代理。

注意，既然是继承就不得不考虑 final 的问题。我们知道 final 类型不能有子类，所以 CGLIB 不能代理 final 类型，遇到这种情况会抛出类似如下异常：

```
java.lang.IllegalArgumentException: Cannot subclass final class cglib.HelloConcrete
```

同样的，final 方法是不能重载的，所以也不能通过 CGLIB 代理，遇到这种情况不会抛异常，而是会跳过 final 方法只代理其他方法。

## 结语

本文介绍了 Java 两种常见动态代理机制的用法和原理，JDK 原生动态代理是 Java 原生支持的，不需要任何外部依赖，但是它只能基于接口进行代理；CGLIB 通过继承的方式进行代理，无论目标对象有没有实现接口都可以代理，但是无法处理 final 的情况。

## SpringAOP 动态代理策略

- 如果目标对象实现了接口，默认情况下会采用 JDK 的动态代理实现 AOP ;
- 如果目标对象实现了接口，可以强制使用 CGLib 实现 AOP；
- 如果目标对象没有实现接口，必须采用 CGLib 库，Spring 会自动在 JDK 动态代理和 CGLib 之间转换；

## 参考文献

[Java Proxy和CGLIB动态代理原理](https://www.cnblogs.com/CarpenterLee/p/8241042.html)

