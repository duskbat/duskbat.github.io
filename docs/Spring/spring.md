
# <center>4. Spring</center>
Spring是一个框架，起到IOC容器的作用，帮我们承载了Bean对象，并且做了对象的整个生命周期的管理，可使用配置文件、注解等方式定义Bean信息。当程序启动之后，需要将定义的Bean对象转换成BeanDefinition，完成BeanDefinition的解析和加载并进行实例化，通过反射创建对象，后面进行初始化操作（aware接口，init-Method，BeanPostProcessor）

### @RestController 与 @Controller
- 单独使用 @Controller 不加 @ResponseBody的话一般用于要返回一个视图的情况，这种情况属于比较传统的Spring MVC 的应用，对应于前后端不分离的情况。
- @ResponseBody 注解的作用是将 Controller 的方法返回的对象通过适当的转换器转换为指定的格式之后，写入到HTTP 响应(Response)对象的 body 中，通常用来返回 JSON 或者 XML 数据，返回 JSON 数据的情况比较多。


### IoC & AOP
#### IoC 和 Dependency Injection
IoC(Inverse of Control) 控制反转是一种设计思想，将原本手动创建对象的控制权，交由Spring框架来管理。IoC 容器是 Spring 用来实现 IoC 的载体，IoC 容器实际上就是个Map(key，value), Map 中存放的是各种对象。

依赖注入: 将对象之间的相互依赖关系交给 IoC 容器来管理，并由 IoC 容器完成属性的注入。  
这样能够很大程度上简化应用的开发，把应用从复杂的依赖关系中解放出来。 IoC 容器像一个工厂，当我们需要创建一个对象的时候，只需要配置好配置文件/注解即可，完全不用考虑对象是如何被创建出来的。 在实际项目中一个 Service 类可能有几百甚至上千个类作为它的底层，假如我们需要实例化这个 Service，你可能要每次都要搞清这个 Service 所有底层类的构造函数。如果利用 IoC 的话，你只需要配置好，然后在需要的地方引用就行了，这大大增加了项目的可维护性且降低了开发难度。


#### IoC容器的初始化过程
启动时读取Bean配置资源信息，解析成BeanDefinition，并在Spring容器中生成一份相应的Bean配置注册表，根据注册表实例化Bean，Bean都会放到缓存池(HashMap)中。
```
XML |--读取--> Resource |--解析--> BeanDefinition |--注册--> BeanFactory
```
### 依赖注入过程


### 源码流程
1. 创建BeanFactory容器
2. 加载配置文件转换成Resource，Resource解析，解析bean定义信息，包装成BeanDefinition
3. 执行BeanFactoryPostProcessor
4. 准备工作：准备BeanPostProcessor、广播器、监听器
5. 实例化
6. 初始化
   - 填充属性 - populateBean
   - 实现Aware接口 - 获取容器对象属性(BeanFactroyAware BeanNameAware BeanClassLoaderAware) 
   - before - ApplicationContextAware
   - init-method
   - after - BeanPostProcessor( AOP 增强 )
7. 获取对象


### AOP
AOP(Aspect-Oriented Programming:面向切面编程) 能够将那些与业务无关，却为业务模块所共同调用的逻辑或责任(例如事务处理、日志管理、权限控制等)封装起来，便于减少系统的重复代码，降低模块间的耦合度，并有利于未来的可拓展性和可维护性。

Spring AOP是基于**动态代理**的:
- 如果要代理的对象实现了某个接口，那么Spring AOP会使用 JDK动态代理，去创建代理对象.
  使用反射生成一个实现代理接口的类, 通过重写方法的方式, 实现对代码的增强. 在调用方法前调用 InvocationHandler 来处理 
- 而对于没有实现接口的对象，就无法使用 JDK Proxy 去进行代理了，这时候Spring AOP会使用 Cglib 生成一个被代理对象的子类来作为代理.
  使用ASM三方框架, 操作字节码, 将代理对象类的class文件加载进来, 通过修改字节码生成子类, 然后重写父类的方法, 实现对代码的增强.

### aop核心概念
- advice 通知：定义了切面的工作以及何时使用
- pointcut 切点：定义连接点位置
- join point 连接点：方法，可被切入的位置
- Aspect 切面：通知和切点的结合


### Spring的设计模式
- 单例模式:
  Bean默认情况下是单例的
- 工厂模式:
  工厂模式主要是通过 BeanFactory 和 ApplicationContext 生产 Bean 对象。它们也是IOC容器, 对Bean进行管理.
- 代理模式:
  AOP 最常见的实现方式就是通过代理实现，Spring 主要使用 JDK 动态代理和 CGLIB 代理。
- 模板方法:
  主要是一些操作数据库的类用到，比如 JdbcTemplate、JpaTemplate，因为查询数据库的建立连接、执行查询、关闭连接几个过程，非常适用于模板方法。


###　Spring AOP 和 AspectJ AOP 区别
Spring AOP 基于动态代理实现，属于运行时增强。
AspectJ 则属于编译时增强，主要有3种方式：
1. 编译时织入：指的是增强的代码和源代码我们都有，直接使用 AspectJ 编译器编译就行了，编译之后生成一个新的类，他也会作为一个正常的 Java 类装载到JVM。
2. 编译后织入：指的是代码已经被编译成 class 文件或者已经打成 jar 包，这时候要增强的话，就是编译后织入，比如你依赖了第三方的类库，又想对他增强的话，就可以通过这种方式。
3. 类加载时织入：指的是在 JVM 加载类的时候进行织入。


### FactoryBean 和 BeanFactory 区别
BeanFactory 是 Bean 的工厂， ApplicationContext 的父类，IOC 容器的核心，负责生产和管理 Bean 对象。
FactoryBean 是 Bean，可以通过实现 FactoryBean 接口(实现接口的类是生成Bean的类, 并不是Bean类本身)去定制实例化 Bean 的逻辑. 


### Bean的生命周期
1. 实例化，创建一个Bean对象
2. 填充属性，为属性赋值populateBean
3. 初始化
  如果实现了xxxAware接口，通过不同类型的Aware接口拿到Spring容器的资源
  如果实现了BeanPostProcessor接口，则会回调该接口的 postProcessBeforeInitialzation() 和 postProcessAfterInitialization() 方法
  如果配置有自定义的 init-method 方法，则会执行 init-method 配置的方法
4. 销毁
  容器关闭后，如果Bean实现了DisposableBean接口，则会回调该接口的destroy()方法
  如果配置了destroy-method方法，则会执行destroy-method配置的方法


### 循环依赖
首先, 有两个前提条件:
1. 不全是构造器导致的循环依赖
2. 必须是单例

本质上解决方式是三级缓存, 通过三级缓存提前拿到未初始化完全的对象  

- 第一级缓存：用来保存实例化、初始化都完成的对象
- 第二级缓存：用来保存实例化完成，但是未初始化完成的对象
- 第三级缓存：保存一个对象工厂，提供一个匿名内部类，用于创建二级缓存中的对象
https://zhuanlan.zhihu.com/p/368769721


### 为什么要三级缓存？
Spring设计中，代理对象的创建是在实例化的最后完成的，而加入缓存的时候是没有进行填充属性和init的，此时不应生成代理对象，所以包一层工厂对象先放入缓存中。
没有代理的话就无所谓了，两级缓存也行。


### Spring事务传播机制(行为)
https://zhuanlan.zhihu.com/p/148504094
带有事务的方法调用之间事务的处理

PROPAGATION_REQUIRED：如果当前没有事务，就创建一个新事务，如果当前存在事务，就加入该事务，这也是通常我们的默认选择。
PROPAGATION_REQUIRES_NEW：创建新事务，无论当前存不存在事务，都创建新事务。
PROPAGATION_NESTED：如果当前存在事务，则在嵌套事务内执行。如果当前没有事务，则按REQUIRED属性执行。
PROPAGATION_NOT_SUPPORTED：以非事务方式执行操作，如果当前存在事务，就把当前事务挂起。
PROPAGATION_NEVER：以非事务方式执行，如果当前存在事务，则抛出异常。
PROPAGATION_MANDATORY：支持当前事务，如果当前存在事务，就加入该事务，如果当前不存在事务，就抛出异常。
PROPAGATION_SUPPORTS：支持当前事务，如果当前存在事务，就加入该事务，如果当前不存在事务，就以非事务执行。


### SpringBoot 启动过程
1. 准备环境，根据不同的环境创建不同的Environment
2. 准备、加载上下文，为不同的环境选择不同的Spring Context，然后加载资源，配置Bean
3. 初始化，这个阶段刷新Spring Context，启动应用
4. 最后结束流程


### Spring Bean 作用域
singleton : 唯一 bean 实例，Spring 中的 bean 默认都是单例的。
prototype : 每次请求都会创建一个新的 bean 实例。
request : 每一次HTTP请求都会产生一个新的bean，该bean仅在当前HTTP request 内有效。
session : 每一次HTTP请求都会产生一个新的 bean，该bean仅在当前 HTTP session 内有效。
global-session： 全局session作用域，仅仅在基于portlet的web应用中才有意义，Spring5已经没有了。Portlet是能够生成语义代码(例如：HTML)片段的小型Java Web插件。它们基于portlet容器，可以像servlet一样处理HTTP请求。但是，与 servlet 不同，每个 portlet 都有不同的会话


### 单例Bean的线程安全问题
通常使用的Bean都是无状态的, 不会有线程安全问题
解决方案:
1. 定义ThreadLocal成员变量, 将可变成员放到里面
2. 改变bean的作用域为prototype, 每次请求都创建新的Bean实例


### @Bean 和 @Component的区别
1. @Bean 用于方法, @Component 用于类
2. @Bean 标注的方法通常定义为产生这个Bean, 尤其是在使用第三方库的时候; @Component 自动侦测 自动装配到Spring容器中


### Spring事务隔离级别
跟MySQL类似, 只多一个default级别, 这个级别使用数据库本身的隔离级别

 
### SpringBoot与SpringMVC
 SpringBoot = Spring MVC + 无配置 + Server(Tomcat Jetty)
 不用手动构建配置(spring.xml)


### spring怎么做事物的，怎么传播的
- 编程式事务
  通过 TransactionTemplate或者TransactionManager手动管理事务
- 声明式事务
  基于 @Transactional 的全注解方式使用最多 实际是通过 AOP 实现
  Spring 容器启动的时候创建一个代理类，调用注解声明的方法的时候实际上是调用的拦截器 TransactionInterceptor 的invoke()方法。
  若同一类中的其他没有 @Transactional 注解的方法内部调用有 @Transactional 注解的方法，有@Transactional 注解的方法的事务会失效。


#### 事务管理接口
- PlatformTransactionManager: 事务策略的核心
- TransactionDefinition: 事务定义信息
  - 隔离级别
    - PROPAGATION_REQUIRED
    - PROPAGATION_REQUIRES_NEW
    - PROPAGATION_NESTED
  - 传播行为
  - 回滚规则：非受检异常(Runtime、Error)才会回滚
  - 是否只读：因为每一条查询语句存储引擎都作为一个事务处理，如果汇总统计类的查询不加事务在DB会有多个事务，数据有可能不一致
  - 事务超时
- TransactionStatus: 事务运行状态


### 注解实现原理，如果让你自己做类似注解怎么实现
- 元注解
  - @Retention（标明注解被保留的阶段）
  - @Target（标明注解使用的范围）
  - @Inherited（标明注解可继承）
  - @Documented（标明是否生成javadoc文档）
- Component
  第一步，初始化时设置了Component类型过滤器；
  第二步，根据指定扫描包扫描.class文件，生成Resource对象；
  第三步、解析.class文件并注解归类，生成MetadataReader对象；
  第四步、使用第一步的注解过滤器过滤出有@Component类；
  第五步、生成BeanDefinition；
  第六步、把BeanDefinition注册到Spring容器。
- AutoWired
  BeanPostProcessor

### Spring 动态代理 静态代理
AspectJ AOP


### Spring Cloud有那些组件
服务发现——Netflix Eureka
客服端负载均衡——Netflix Ribbon
断路器——Netflix Hystrix
服务网关——Netflix Zuul
分布式配置——Spring Cloud Config


### SpringMVC M V C
Model: 通常指的是数据
View: UI，页面
Controller: 负责处理请求和逻辑


### 用户请求MVC会发生什么
1. Http请求：浏览器请求提交到DispatcherServlet。
2. 寻找处理器：由DispatcherServlet控制器查询一个或多个HandlerMapping，找到处理请求的Controller。
3. 调用处理器：DispatcherServlet将请求提交到Controller。
4. 调用业务处理和返回结果：Controller调用业务逻辑处理后，返回ModelAndView。
5. 处理视图映射并返回模型： DispatcherServlet查询一个或多个ViewResoler视图解析器，找到ModelAndView指定的视图，将model传递给View显示。
6. Http响应：视图负责将结果显示到客户端。







