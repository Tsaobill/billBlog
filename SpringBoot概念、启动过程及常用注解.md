### SpringBoot概念

是一个用来简化 Spring 应用的初始搭建以及开发过程的框架，整合并默认配置了很多框架的使用方式。使用SpringBoot可以通过很少的配置快速搭建并启动一个项目。



### SpringBoot应用启动流程

#### 启动类

```java
/**
 * @SpringBootApplication 来标注一个主程序类，说明这是一个Spring Boot应用
 */
@SpringBootApplication
public class HelloWorldMainApplication {
    public static void main(String[] args) {
        // Spring应用启动起来
        SpringApplication.run(HelloWorldMainApplication.class, args);
    }
}
```

使用`@SpringBootApplication` 注解标注一个类是SpringBoot的启动类，该类包含一个main函数，通过调用静态方法`run()`，接受启动类的Class和启动命令项args作为参数，委托SpringApplication进行启动。可以看出启动Springboot应用启动过程有两个重点：

#### `@SpringBootApplication` 注解

单个`@SpringBootApplication`注释可用于启用这三个功能，即：

- `@EnableAutoConfiguration`：启用[Spring Boot的自动配置机制](https://www.springcloud.cc/spring-boot.html#using-boot-auto-configuration)
- `@ComponentScan`：对应用程序所在的软件包启用`@Component`扫描（请参阅[最佳实践](https://www.springcloud.cc/spring-boot.html#using-boot-structuring-your-code)）
- `@Configuration`：允许在上下文中注册额外的beans或导入其他配置类

使用单个`@SpringBootApplication`注解等于使用`@Configuration`，`@EnableAutoConfiguration`和`@ComponentScan`及其默认属性。

#### `SpringApplication.run` 方法

该静态方法对SpringApplication类的同名实例方法进行包装，

```java
// 启动类的main函数中调用该run方法
public static ConfigurableApplicationContext run(Class<?> primarySource,
			String... args) {
  // 继续调用重载的run方法
		return run(new Class<?>[] { primarySource }, args);
	}
// 重载的静态run方法
	public static ConfigurableApplicationContext run(Class<?>[] primarySources,
			String[] args) {
		return new SpringApplication(primarySources).run(args);
	}
// 实例run方法
	public ConfigurableApplicationContext run(String... args) {
    // 启动计时器
		StopWatch stopWatch = new StopWatch();
		stopWatch.start();
		ConfigurableApplicationContext context = null;
    // 异常报告
		Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
    configureHeadlessProperty();
    // 获取监听器并启动
		SpringApplicationRunListeners listeners = getRunListeners(args);
		listeners.starting();
		try {
      // 封装命令行参数
			ApplicationArguments applicationArguments = new DefaultApplicationArguments(
					args);
      // 准备环境
			ConfigurableEnvironment environment = prepareEnvironment(listeners,
					applicationArguments);
			configureIgnoreBeanInfo(environment);
      // 打印banner
			Banner printedBanner = printBanner(environment);
      // 创建ApplicationContext，根据应用类型创建不同的ioc容器
			context = createApplicationContext();
			exceptionReporters = getSpringFactoriesInstances(
					SpringBootExceptionReporter.class,
					new Class[] { ConfigurableApplicationContext.class }, context);
			// 准备上下文，将环境保存到ioc容器，getBeanFactory，注册单例bean
      prepareContext(context, environment, listeners, applicationArguments,
					printedBanner);
      // 刷新容器：ioc容器初始化（如果是web应用还会创建嵌入式的Tomcat）
      // 进行自动配置的地方（扫描、创建、加载组件）
			refreshContext(context);
      //从ioc容器中获取所有的ApplicationRunner和CommandLineRunner进行回调
			//ApplicationRunner先回调，CommandLineRunner再回调
			afterRefresh(context, applicationArguments);
      // 停止计时
			stopWatch.stop();
			if (this.logStartupInfo) {
				new StartupInfoLogger(this.mainApplicationClass)
						.logStarted(getApplicationLog(), stopWatch);
			}
      // 回调listener的finish方法
			listeners.started(context);
			callRunners(context, applicationArguments);
		}
		catch (Throwable ex) {
			handleRunFailure(context, ex, exceptionReporters, listeners);
			throw new IllegalStateException(ex);
		}

		try {
			listeners.running(context);
		}
		catch (Throwable ex) {
			handleRunFailure(context, ex, exceptionReporters, null);
			throw new IllegalStateException(ex);
		}
		return context;
	}

```

#### 自动配置

自动配置主要是通过`AutoConfigurationImportSelector`类的`selectImports()`方法想容器中导入组件，将类路径下 **META-INF/spring.factories** 里面配置的所有**EnableAutoConﬁguration**的值加入到了容器中；



### 常用注解

##### bean相关

**`@Autowired`**：自动导入实例对象到类中，被注入的类

> @Autowired注解是按类型装配依赖对象，默认情况下它要求依赖对象必须存在，如果允许null值，可以设置它required属性为false。
> 如果我们想使用按名称装配，可以结合@Qualifier注解制定名称使用。
>
> 可以通过required=false来允许对象为null



**`@Resource`**：与`@Autowired`一样，但装配方式不同

> @Resource默认按名称装配，当找不到与名称匹配的bean才会按类型装配。



`@Component`,`@Repository`,`@Service`,`@Controller`

> 通过上述注解，可以将类标记成能够被spring容器自动装配的类
>
> - `@Component` ：通用的注解，可标注任意类为 `Spring` 组件。如果一个 Bean 不知道属于哪个层，可以使用`@Component` 注解标注。
> - `@Repository` : 对应持久层即 Dao 层，主要用于数据库相关操作。
> - `@Service` : 对应服务层，主要涉及一些复杂的逻辑，需要用到 Dao 层。
> - `@Controller` : 对应 Spring MVC 控制层，主要用户接受用户请求并调用 Service 层返回数据给前端页面。

##### 其他注解

**`@RestController`**

> `@RestController`注解是`@Controller`和`@ResponseBody`的合集,表示这是个控制器 bean,用来处理HTTP请求，并且将字符串直接填入 HTTP 响应体中,是 REST 风格的控制器。



**`@RequestMapping`**

> 其参数用来指定请求路径将会分配到被其标注的方法上


**`@Configuration`**

一般用来声明配置类，可以使用 `@Component`注解替代，不过使用`Configuration`注解声明配置类更加语义化。

**`@Scope`**

声明 Spring Bean 的作用域，使用方法:

```java
@Bean
@Scope("singleton")
public Person personSingleton() {
    return new Person();
}
```

**四种常见的 Spring Bean 的作用域：**

- singleton : 唯一 bean 实例，Spring 中的 bean 默认都是单例的。
- prototype : 每次请求都会创建一个新的 bean 实例。
- request : 每一次 HTTP 请求都会产生一个新的 bean，该 bean 仅在当前 HTTP request 内有效。
- session : 每一次 HTTP 请求都会产生一个新的 bean，该 bean 仅在当前 HTTP session 内有效。

 **`@Configuration`**

一般用来声明配置类，可以使用 `@Component`注解替代，不过使用`Configuration`注解声明配置类更加语义化。

```java
@Configuration
public class AppConfig {
    @Bean
    public TransferService transferService() {
        return new TransferServiceImpl();
    }

}
```

