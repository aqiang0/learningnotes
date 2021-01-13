# spring源码

## 1 看源码前置知识

### 1.1 Servlet规范

**web容器（Tomcat）启动时干了啥活：**

- 把我们写的程序打包，会打包成一个个war包，每个war包都对应一个web应用，放到web容器中，如拷贝到tomcat的webapp目录
- web容器启动的时候，会初始化web应用，即创建ServletContext对象
- 加载解析web.xml文件，获取该应用的Filters，Listener，Servlet等组件的配置并创建对象实例，**作为ServletContext的属性**，保存在ServletContext当中
- web容器接收到客户端请求时，使用应用配置的Filters对这个请求先进行过滤
- 过滤之后，会根据请求信息，匹配到处理这个请求的Servlet，最后才交给servlet处理
- 其实一个spring项目就是对应web容器里的一个ServletContext，**所以在ServletContext对象的创建和初始化的时候，就需要一种机制来触发spring相关组件的创建和初始化**

### 1.2 Listener监听器机制：ContextLoaderListener

- 创建ServletContext对象时，会产生一个**ServletContextEvent**事件，ServletContextEvent包含该ServletContext对象的引用，然后根据web.xml中配置的，给这个ServletContext注册一个监听器ServletContextListener。

```java
/**
 * Implementations of this interface receive notifications about changes to the
 * servlet context of the web application they are part of. To receive
 * notification events, the implementation class must be configured in the
 * deployment descriptor for the web application.
 *
 * @see ServletContextEvent
 * @since v 2.3
 */
public interface ServletContextListener extends EventListener {

    /**
     ** Notification that the web application initialization process is starting.
     * All ServletContextListeners are notified of context initialization before
     * any filter or servlet in the web application is initialized.
     * @param sce Information about the ServletContext that was initialized
     */
    public void contextInitialized(ServletContextEvent sce);

    /**
     ** Notification that the servlet context is about to be shut down. All
     * servlets and filters have been destroy()ed before any
     * ServletContextListeners are notified of context destruction.
     * @param sce Information about the ServletContext that was destroyed
     */
    public void contextDestroyed(ServletContextEvent sce);
}
```

- 从contextInitialized的注释可知：通知所有的ServletContextListeners，当前的web应用正在启动，而且这些**ServletContextListeners是在Filters和Servlets创建之前接收到通知的**。所以在这个时候，**web应用还不能接收请求**，故可以在这里完成底层处理请求的组件的加载，这样等之后接收请求的Filters和Servlets创建时，则可以使用这些创建好的组件了。spring相关的bean就是这里所说的底层处理请求的组件，如数据库连接池，数据库事务管理器等。

- **ContextLoaderListener**：spring-web包的ContextLoaderListener就是一个ServletContextListener的实现类。ContextLoaderListener主要用来获取spring项目的整体配置信息，并**创建对应的WebApplicationContext**来保存bean的信息，以及创建这些bean的对象实例。默认去WEB-INF下加载applicationContext.xml配置，如果applicationContext.xml放在其他位置，或者使用其他不同的名称，或者使用多个xml文件，则与指定contextConfigLocation。

### 1.3 DispatcherServlet：前端控制器

- 在web容器中，web.xml中的加载顺序：**context-param -> listener -> filter -> servlet**。其中ContextLoaderListener是属于listener阶段。我们通常需要在项目的web.xml中配置一个DispatcherServlet，并配置拦截包含“/”路径的请求，即拦截所有请求。这样在web容器启动应用时，在servlet阶段会创建这个servlet，由Servlet规范中servlet的生命周期方法可知：

```java
public interface Servlet {
    void init(ServletConfig var1) throws ServletException;

    ServletConfig getServletConfig();

    void service(ServletRequest var1, ServletResponse var2) throws ServletException, IOException;

    String getServletInfo();

    void destroy();
}
```

- web容器在创建这个servlet的时候，会调用其init方法，故可以在DispatcherServlet的init方法中定义初始化逻辑，核心实现了创建DispatcherServlet自身的一个WebApplicationContext，注意在spring中每个servlet可以包含一个独立的WebApplicationContext来维护自身的组件，而上面通过ContextLoaderListener创建的WebApplicationContext为共有的，通常也是最顶层，即root WebApplicationContext，servlet的WebApplicationContext可以通过setParent方法设值到自身的一个属性。DispatcherServlet默认是加载WEB-INF下面的“servletName”-servlet.xml，来获取配置信息的，也可以与ContextLoaderListener一样通过contextLoaderConfig来指定位置

### 1.4 总结

- 从上面的分析，可知spring相关配置解析和组件创建其实是在web容器中，启动一个web应用的时候，即在其ServletContext组件创建的时候，首先解析web.xml获取该应用配置的listeners列表和servlet列表，然后保存在自身的一个属性中，然后通过分发生命周期事件ServletContextEvent给这些listeners，从而在listeners感知到应用在启动了，然后自定义自身的处理逻辑，如spring的ContextLoaderListener就是解析spring的配置文件并创建相关的bean，这样其实也是实现了一种代码的解耦；其次是创建配置的servlet列表，调用servlet的init方法，这样servlet可以自定义初始化逻辑，DispatcherServlet就是其中一个servlet。
- 所以在ContextLoaderListener和DispatcherServlet的创建时，都会进行WebApplicationContext的创建，这里其实就是IOC容器的创建了，即会交给spring-context，spring-beans包相关的类进行处理了，故可以从这里作为一个入口，一层一层地剥spring的源码了