# Spring Bean 的循环依赖问题

## 1. 构造器参数循环依赖

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

## 2. setter 方式单例，默认方式

如果要说 setter 方式注入的话，我们最好先看一张 Spring 中 Bean 实例化的图

![img](https://camo.githubusercontent.com/a3d4415162d30d4659779f6db3717f9a68fd3c97/687474703a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f31382d392d31372f353439363430372e6a7067)

如图中前两步骤得知：**Spring 是先将 Bean 对象实例化之后再设置对象属性的**

修改配置文件为 setter 方式注入。

```xml
<!-- scope="singleton"(默认就是单例方式) -->
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

## 3. setter 方式 + prototype (原型模式)

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

## 参考资料

[Spring循环依赖的三种方式](https://blog.csdn.net/u010644448/article/details/59108799)