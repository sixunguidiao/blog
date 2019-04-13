# Java 反射如何获取私有属性并改变值

先简单介绍一下 Java 的反射：

Java 反射机制在程序**运行时**，对于任意一个类，都能够知道这个类的所有属性和方法；对于任意一个对象，都能够调用它的任意一个方法和属性。这种 **动态的获取信息** 以及 **动态调用对象的方法** 的功能称为 **java 的反射机制**。

反射机制很重要的一点就是“运行时”，其使得我们可以在程序运行时加载、探索以及使用编译期间完全未知的 `.class` 文件。换句话说，Java 程序可以加载一个运行时才得知名称的 `.class` 文件，然后获悉其完整构造，并生成其对象实体、或对其 fields（变量）设值、或调用其 methods（方法）。

下面是一个测试类：

```java
public class TestClass {

    private String MSG = "Original";

    private void privateMethod(String head , int tail){
        System.out.print(head + tail);
    }

    public String getMsg(){
        return MSG;
    }
}
```

## 1. 修改私有变量

以修改 `TestClass` 类中的私有变量 `MSG` 为例，其初始值为 "Original" ，我们要修改为 "Modified"。

```java
/**
 * 修改对象私有变量的值
 * 为简洁代码，在方法上抛出总的异常
 */
private static void modifyPrivateFiled() throws Exception {
    //1. 获取 Class 类实例
    TestClass testClass = new TestClass();
    Class mClass = testClass.getClass();

    //2. 获取私有变量
    Field privateField = mClass.getDeclaredField("MSG");

    //3. 操作私有变量
    if (privateField != null) {
        //获取私有变量的访问权
        privateField.setAccessible(true);

        //修改私有变量，并输出以测试
        System.out.println("Before Modify：MSG = " + testClass.getMsg());

        //调用 set(object , value) 修改变量的值
        //privateField 是获取到的私有变量
        //testClass 要操作的对象
        //"Modified" 为要修改成的值
        privateField.set(testClass, "Modified");
        System.out.println("After Modify：MSG = " + testClass.getMsg());
    }
}
```

```
Before Modify：MSG = Original
After Modify：MSG = Modified
```

从上面的输出信息可以看出修改私有变量成功。

## 2. 修改私有常量

### 1. 真的能修改吗

常量是指使用 `final` 修饰符修饰的成员属性，与变量的区别就在于有无 `final` 关键字修饰。在说之前，先补充一个知识点。

Java 虚拟机（JVM）在编译 `.java` 文件得到 `.class` 文件时，会优化我们的代码以提升效率。其中一个优化就是：JVM 在编译阶段会把引用常量的代码替换成具体的常量值，如下所示（部分代码）。

编译前的 `.java` 文件：

```java
//注意是 String  类型的值
private final String FINAL_VALUE = "hello";

if(FINAL_VALUE.equals("world")){
    //do something
}
```

编译后得到的 `.class` 文件（当然，编译后是没有注释的）：

```java
private final String FINAL_VALUE = "hello";
//替换为"hello"
if("hello".equals("world")){
    //do something
}
```

但是，并不是所有常量都会优化。经测试对于 `int` 、`long` 、`boolean` 以及 `String` 这些基本类型 JVM 会优化，而对于 `Integer` 、`Long` 、`Boolean` 这种包装类型，或者其他诸如 `Date` 、`Object` 类型则不会被优化。

总结来说：**对于基本类型的静态常量，JVM 在编译阶段会把引用此常量的代码替换成具体的常量值**。

这么说来，在实际开发中，如果我们想修改某个类的常量值，恰好那个常量是基本类型的，岂不是无能为力了？反正我个人认为除非修改源码，否则真没办法！

这里所谓的无能为力是指：**我们在程序运行时依然可以使用反射修改常量的值（后面会代码验证），但是 JVM 在编译阶段得到的 .class 文件已经将常量优化为具体的值，在运行阶段就直接使用具体的值了，所以即使修改了常量的值也已经毫无意义了**。

下面我们验证这一点，在测试类 `TestClass` 类中添加如下代码：

```java
//String 会被 JVM 优化
private final String FINAL_VALUE = "FINAL";

public String getFinalValue(){
    //剧透，会被优化为: return "FINAL" ,拭目以待吧
    return FINAL_VALUE;
}
```

接下来，是修改常量的值，先上代码，请仔细看注释：

```java
/**
 * 修改对象私有常量的值
 * 为简洁代码，在方法上抛出总的异常，实际开发别这样
 */
private static void modifyFinalFiled() throws Exception {
    //1. 获取 Class 类实例
    TestClass testClass = new TestClass();
    Class mClass = testClass.getClass();

    //2. 获取私有常量
    Field finalField = mClass.getDeclaredField("FINAL_VALUE");

    //3. 修改常量的值
    if (finalField != null) {

        //获取私有常量的访问权
        finalField.setAccessible(true);

        //调用 finalField 的 getter 方法
        //输出 FINAL_VALUE 修改前的值
        System.out.println("Before Modify：FINAL_VALUE = "
                + finalField.get(testClass));

        //修改私有常量
        finalField.set(testClass, "Modified");

        //调用 finalField 的 getter 方法
        //输出 FINAL_VALUE 修改后的值
        System.out.println("After Modify：FINAL_VALUE = "
                + finalField.get(testClass));

        //使用对象调用类的 getter 方法
        //获取值并输出
        System.out.println("Actually ：FINAL_VALUE = "
                + testClass.getFinalValue());
    }
}
```

```
Before Modify：FINAL_VALUE = FINAL
After Modify：FINAL_VALUE = Modified
Actually ：FINAL_VALUE = FINAL
```

结果出来了:

第一句打印修改前 `FINAL_VALUE` 的值，没有异议；

第二句打印修改后常量的值，说明`FINAL_VALUE`确实通过反射修改了；

第三句打印通过   `getFinalValue()` 方法获取的 `FINAL_VALUE` 的值，但还是初始值，导致修改无效！

下面是`TestClass.java` 文件编译后得到的 `TestClass.class` 文件：

![TestClass.class æä"¶](https://user-gold-cdn.xitu.io/2017/8/12/2bdcae6d3cde4638c23a0cfe34f85aa6?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

看到了吧，有图有真相，`getFinalValue()` 方法直接 `return "FINAL"`！同时也说明了，**程序运行时是根据编译后的 .class 来执行的**。

### 2. 想办法也要修改

#### 方法一：常量声明的时候不赋值，通过构造函数给常量复制

事实上，Java 允许我们声明常量时不赋值，但必须在构造函数中赋值。你可能会问我为什么要说这个，这就解释：

我们修改一下 `TestClass` 类，在声明常量时不赋值，然后添加构造函数并为其赋值，大概看一下修改后的代码（部分代码 ）：

```java
public class TestClass {

    //......
    private final String FINAL_VALUE;

    //构造函数内为常量赋值 
    public TestClass(){
        this.FINAL_VALUE = "FINAL";
    }
    //......
}
```

现在，我们再调用上面贴出的修改常量的方法，发现输出是这样的：

```
Before Modify：FINAL_VALUE = FINAL
After Modify：FINAL_VALUE = Modified
Actually ：FINAL_VALUE = Modified
```

![TestClass.class æä"¶](https://user-gold-cdn.xitu.io/2017/8/12/1bc016abdefa8f0822dde437fdd76323?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

解释一下：我们将赋值放在构造函数中，构造函数是我们运行时 new 对象才会调用的，所以就不会像之前直接为常量赋值那样，在编译阶段将 `getFinalValue()` 方法优化为返回常量值，而是指向 `FINAL_VALUE` ，这样我们在运行阶段通过反射修改敞亮的值就有意义啦。但是，看得出来，程序还是有优化的，将构造函数中的赋值语句优化了。再想想那句 **程序运行时是根据编译后的 .class 来执行的** ，相信你一定明白为什么这么输出了！

#### 方法二：使用三目运算符

略。

## 反射的用途

### 反射的优势和劣势

- 优势：运行期类型的判断，动态类加载，提高代码的灵活度。
- 劣势：性能瓶颈：反射相当于一系列解释操作，通知 JVM 要做的事情，性能比直接的 Java 代码要慢很多 (因为 JVM 无法对代码做编译期优化)。

### 反射的应用场景

- JDBC 的数据库的连接

在 JDBC 的操作中，如果要想进行数据库的连接，则必须按照以上的几步完成：

1. 通过 Class.forName() 加载数据库的驱动程序 (通过反射加载，前提是引入相关了 Jar 包)。
2. 通过 DriverManager 类进行数据库的连接，连接的时候要输入数据库的连接地址、用户名、密码。
3. 通过 Connection 接口接收连接。

- Spring 框架的使用

Spring 通过 XML 配置模式装载 Bean 的过程：

1. 将程序内所有 XML 或 Properties 配置文件加载入内存中。
2. Java 类里面解析 XML 或 Properties 里面的内容，得到对应实体类的字节码字符串以及相关的属性信息。
3. 使用反射机制，根据这个字符串获得某个类的 Class 实例。
4. 动态配置实例的属性。

## 参考资料

[Java 反射由浅入深 | 进阶必备](https://juejin.im/post/598ea9116fb9a03c335a99a4)

