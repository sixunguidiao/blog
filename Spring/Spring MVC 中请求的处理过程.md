# Spring MVC 中请求的处理过程

![1553007418195](C:\Users\guidiao\AppData\Roaming\Typora\typora-user-images\1553007418195.png)

在请求离开浏览器时，会带有用户所请求内容的信息，至少会包含请求的 URL。但是还可能带有其他的信息，例如用户提交的表单信息。

请求旅程的第一站是 Spring 的 DispatcherServlet。与大多数基于 Java 的 Web 框架一样，Spring MVC 所有的请求都会通过一个前端控制器 (front controller) Servlet。前端控制器是常用的 Web 应用程序模式，在这里一个单实例的 Servlet 将请求委托给应用程序的其他组件来执行实际的处理。在 Spring MVC 中，DispatcherServlet 就是前端控制器。

DispatcherServlet 的任务是将请求发送给 Spring MVC 控制器 controller。控制器是一个用于处理请求的 Spring 组件。在典型的应用程序中可能会有多个控制器，DispatcherServlet 需要知道应该将请求发送给哪个控制器。所以 DispatcherServlet 会查询一个或多个处理器映射 (handler mapping) 来确定请求的下一站在哪里。处理器映射会根据请求所携带的 URL 信息来进行决策。

一旦选择了合适的控制器，DispatcherServlet 会将请求发送给选中的控制器。到了控制器，请求会卸下其负载 (用户提交的信息) 并耐心等待控制器处理这些信息。(实际上，设计良好的控制器本身只处理很少甚至不处理工作，而是将业务逻辑委托给一个或多个服务对象进行处理。)

控制器在完成逻辑处理后，通常会产生一些信息，这些信息需要返回给用户并在浏览器上显示。这些信息被称为模型 (model)。不过仅仅给用户返回原始的信息是不够的——这些信息需要以用户友好的方式进行格式化，一般会是 HTML。所以，信息需要发送给一个视图 (view)，通常会是 JSP。

控制器所做的最后一件事就是将模型数据打包，并且标示出用于渲染输出的视图名。它接下来会将请求连同模型和视图名发送回 DispatcherServlet。

这样，控制器就不会与特定的视图相耦合，传递给 DispatcherServlet 的视图名并不直接表示某个特定的 JSP。实际上，它甚至并不能确定视图就是 JSP。相反，它仅仅传递了一个逻辑名称，这个名字将会用来查找产生结果的真正视图。DispatcherServlet 将会使用视图解析器 (view resolver) 来将逻辑视图名匹配为一个特定的视图实现，它可能是也可能不是 JSP。

既然 DispatcherServlet 已经知道由哪个视图渲染结果，那请求的任务基本上也就完成了。它的最后一站是视图的实现 (可能是JSP) ，在这里它交付模型数据。请求的任务就完成了。视图将使用模型数据渲染输出，这个输出会通过响应对象传递给客户端 (不会像听上去那样硬编码) 。

## 参考资料

《Spring 实战》

