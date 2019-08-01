# Lambda 表达式 (Lambda expressions)

## Lambda 表达式简介

Lambda 表达式是一种匿名函数 (对 Java 而言这并不完全正确，但现在姑且这么认为)，简单地说，它是没有声明的方法，也即没有访问修饰符、返回值声明和名字。

你可以将其想做一种速记，在你需要使用某个方法的地方写上它。当某个方法只使用一次，而且定义很简短，使用这种速记替代之尤其有效，这样，你就不必在类中费力写声明与方法了。

Java 中的 Lambda 表达式通常使用 `(argument) -> (body)` 语法书写，例如：

```java
(arg1, arg2...) -> { body }

(type1 arg1, type2 arg2...) -> { body }
```

以下是一些 Lambda 表达式的例子：

```java
(int a, int b) -> {  return a + b; }

() -> System.out.println("Hello World");

(String s) -> { System.out.println(s); }

() -> 42

() -> { return 3.1415 };
```

## Lambda 表达式的结构

让我们了解一下 Lambda 表达式的结构。

- 一个 Lambda 表达式可以有零个或多个参数
- 参数的类型既可以明确声明，也可以根据上下文来推断。例如：`(int a)`与`(a)`效果相同
- 所有参数需包含在圆括号内，参数之间用逗号相隔。例如：`(a, b)` 或 `(int a, int b)` 或 `(String a, int b, float c)`
- 空圆括号代表参数集为空。例如：`() -> 42`
- 当只有一个参数，且其类型可推导时，圆括号（）可省略。例如：`a -> return a*a`
- Lambda 表达式的主体可包含零条或多条语句
- 如果 Lambda 表达式的主体只有一条语句，花括号{}可省略。匿名函数的返回类型与该主体表达式一致
- 如果 Lambda 表达式的主体包含一条以上语句，则表达式必须包含在花括号{}中（形成代码块）。匿名函数的返回类型与代码块的返回类型一致，若没有返回则为空

## 什么是函数式接口

函数式接口是只包含一个抽象方法声明的接口.

`java.lang.Runnable` 和 `java.util.concurrent.Callable` 就是一种函数式接口，在 Runnable 接口中只声明了一个方法 `void run()`，相似地，ActionListener 接口也是一种函数式接口。之前，我们使用匿名内部类来实例化函数式接口的对象，有了 Lambda 表达式，这一方式可以得到简化。

每个 Lambda 表达式都能隐式地赋值给函数式接口，例如，我们可以通过 Lambda 表达式创建 Runnable 接口的引用。

```java
Runnable r = () -> System.out.println("hello world");
```

当不指明函数式接口时，编译器会自动解释这种转化：

```java
new Thread(
   () -> System.out.println("hello world")
).start();
```

因此，在上面的代码中，编译器会自动推断：根据线程类的构造函数签名 `public Thread(Runnable r) { }`，将该 Lambda 表达式赋给 Runnable 接口。

以下是一些 Lambda 表达式及其函数式接口：

[@FunctionalInterface](http://download.java.net/jdk8/docs/api/java/lang/FunctionalInterface.html) 是 Java 8 新加入的一种接口，用于指明该接口类型声明是根据 Java 语言规范定义的函数式接口。Java 8 还声明了一些 Lambda 表达式可以使用的函数式接口，当你注释的接口不是有效的函数式接口时，可以使用 @FunctionalInterface 解决编译层面的错误。

以下是一种自定义的函数式接口： 

```java
@FunctionalInterface public interface WorkerInterface {

   public void doSomeWork();

}
```

根据定义，函数式接口只能有一个抽象方法，如果你尝试添加第二个抽象方法，将抛出编译时错误。例如：

```java
@FunctionalInterface
public interface WorkerInterface {

    public void doSomeWork();

    public void doSomeMoreWork();

}
```

错误：

```
Unexpected @FunctionalInterface annotation 
    @FunctionalInterface ^ WorkerInterface is not a functional interface multiple 
    non-overriding abstract methods found in interface WorkerInterface 1 error
```

函数式接口定义好后，我们可以在 API 中使用它，同时利用 Lambda 表达式。例如：

```java
// 定义一个函数式接口
@FunctionalInterface
public interface WorkerInterface {

   public void doSomeWork();

}


public class WorkerInterfaceTest {

public static void execute(WorkerInterface worker) {
    worker.doSomeWork();
}

public static void main(String [] args) {

    // invoke doSomeWork using Annonymous class
    execute(new WorkerInterface() {
        @Override
        public void doSomeWork() {
            System.out.println("Worker invoked using Anonymous class");
        }
    });

    // invoke doSomeWork using Lambda expression 
    execute( () -> System.out.println("Worker invoked using Lambda expression") );
}

}
```

输出：

```
Worker invoked using Anonymous class 
Worker invoked using Lambda expression
```

这上面的例子里，我们创建了自定义的函数式接口并与 Lambda 表达式一起使用。execute() 方法现在可以将 Lambda 表达式作为参数。

## 参考资料

[深入浅出 Java 8 Lambda 表达式](http://blog.oneapm.com/apm-tech/226.html)

