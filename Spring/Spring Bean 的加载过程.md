# Spring Bean 的加载过程

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

## 参考资料

[【Spring】详解Spring中Bean的加载](https://www.jianshu.com/p/5fd1922ccab1)