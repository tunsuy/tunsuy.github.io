# 注解Configuration、EnableAutoConfiguration、ComponentScan和Component的区别

### @ComponentScan VS @EnableAutoConfiguration

相同点：
两者都可以将带有@Component，@Service等注解的对象加入到ioc容器中。

不同点：
* 1.两者虽然都能将带有注解的对象放入ioc容器中，但是它们扫描的范围是不一样的。@ComponentScan扫描的范围默认是它所在的包以及子包中所有带有注解的对象，@EnableAutoConfiguration扫描的范围默认是它所在类。
* 2.它们作用的对象不一样，@EnableAutoConfiguration除了扫描本类带有的注解外，还会借助@Import的支持，收集和注册依赖包中相关的bean定义，将这些bean注入到ioc容器中，在springboot中注入的bean有两部分组成，一部分是自己在代码中写的标注有@Controller,@service,@Respority等注解的业务bean，这一部分bean就由@ComponentScan将它们加入到ioc容器中，还有一部分是springboot自带的相关bean，可以将这部分bean看成是工具bean，这部分bean就是由@EnableAutoConfiguration负责加入到容器中。
* 3.@ComponentScan初始化的方式总是在@EnableAutoConfiguration初始化方式之前

#### 1、@ComponentScan
在开发应用程序时，我们需要告诉Spring框架寻找Spring管理的组件。 @ComponentScan使Spring能够扫描诸如configurations, controllers, services和我们定义的其他组件之类的东西。

Spring可以从指定的包开始扫描，我们可以使用`basePackageClasses()`或`basePackages()`进行定义。如果未指定包，则它将声明@ComponentScan批注的类的包视为起始包：
```java
@Configuration
@ComponentScan(basePackages = {"com.baeldung.annotations.componentscanautoconfigure.healthcare",
  "com.baeldung.annotations.componentscanautoconfigure.employee"},
  basePackageClasses = Teacher.class)
public class EmployeeApplication {
    public static void main(String[] args) {
        ApplicationContext context = SpringApplication.run(EmployeeApplication.class, args);
        // ...
    }
}
```

Spring在指定的包及其所有子包中搜索以`@Configuration`注解的类。另外，`Configuration`类可以包含`@Bean`注解，这些注解将方法返回的类注册为Spring容器中的bean。之后，`@ComponentScan`注解可以自动检测到此类bean：
```java
@Configuration
public class Hospital {
    @Bean
    public Doctor getDoctor() {
        return new Doctor();
    }
}
```

此外，`@ComponentScan`注解还可以扫描，检测和注册Bean，以查找带有`@Component，@Controller，@ Service和@Repository`注解的类。

总结：
如果你的其他包都在使用了@SpringBootApplication注解的main app所在的包及其下级包，则你什么都不用做，SpringBoot会自动帮你把其他包都扫描了
如果你有一些bean所在的包，不在main app的包及其下级包，那么你需要手动加上@ComponentScan注解并指定那个bean所在的包


#### 2、@EnableAutoConfiguration
@EnableAutoConfiguration注解使Spring Boot能够自动配置应用程序上下文。因此，它会基于类路径中包含的jar文件和我们定义的bean自动创建和注册bean。
例如，当我们在类路径中定义spring-boot-starter-web依赖项时，Spring boot会自动配置Tomcat和Spring MVC。但是，如果我们定义自己的配置，则此自动配置的优先级较低。

通俗的说：该注解通过读取spring.factories文件里面的EnableAutoConfiguration下面指定的类，来初始化指定类下面的所有加了@bean的方法，并初始化这个bean

声明`@EnableAutoConfiguratio`注解的类的包被视为默认包。因此，我们应该始终在根包中应用`@EnableAutoConfiguration`注解，以便可以检查每个子包和类：
```java
@Configuration
@EnableAutoConfiguration
public class EmployeeApplication {
    public static void main(String[] args) {
        ApplicationContext context = SpringApplication.run(EmployeeApplication.class, args);
        // ...
    }
}
```
此外，`@EnableAutoConfiguration`注解提供了两个参数以手动排除任何参数：
我们可以使用exclude禁用我们不想自动配置的类的列表：
```java
@Configuration
@EnableAutoConfiguration(exclude={JdbcTemplateAutoConfiguration.class})
public class EmployeeApplication {
    public static void main(String[] args) {
        ApplicationContext context = SpringApplication.run(EmployeeApplication.class, args);
        // ...
    }
}
```

我们可以使用excludeName定义要从自动配置中排除的类名的完全限定列表：
```java
@Configuration
@EnableAutoConfiguration(excludeName = {"org.springframework.boot.autoconfigure.jdbc.JdbcTemplateAutoConfiguration"})
public class EmployeeApplication {
    public static void main(String[] args) {
        ApplicationContext context = SpringApplication.run(EmployeeApplication.class, args);
        // ...
    }
}
```
从Spring Boot 1.2.0开始，我们可以使用@SpringBootApplication注释，该注释是`@Configuration，@ EnableAutoConfiguration和@ComponentScan`这三个注解及其默认属性的组合：
```java
@SpringBootApplication
public class EmployeeApplication {
    public static void main(String[] args) {
        ApplicationContext context = SpringApplication.run(EmployeeApplication.class, args);
        // ...
    }
}
```
Spring Boot的自动配置均是通过spring.factories来指定的，它的优先级最低（执行时机是最晚的）；通过扫描进来的优先级是最高的。


### @Component vs @Configuration
一句话概括就是 @Configuration 中所有带 @Bean 注解的方法都会被动态代理，因此调用该方法返回的都是同一个实例。

#### 1、@Configuration注解
我们看下下面的例子：
```java
@Configuration
public static class Config {

    @Bean
    public SimpleBean simpleBean() {
        return new SimpleBean();
    }

    @Bean
    public SimpleBeanConsumer simpleBeanConsumer() {
        return new SimpleBeanConsumer(simpleBean());
    }
}
```
相信大多数人第一次看到上面 simpleBeanConsumer() 中调用 simpleBean() 时，会认为这里的 SimpleBean实例 和上面 @Bean 方法返回的 SimpleBean实例 可能不是同一个对象，因此可能会通过下面的方式来替代这种方式：
```java
@Autowired
private SimpleBean simpleBean;
```
实际上不需要这么做，为什么？因为直接调用 simpleBean() 方法返回的是同一个实例。下面我们来具体说说原因。

@Configuration注解的定义如下：
```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Component
public @interface Configuration {
    String value() default "";
}
```
从定义来看， @Configuration 注解本质上还是 @Component，因此 <context:component-scan/> 或者 @ComponentScan 都能处理@Configuration 注解的类。

@Configuration 标记的类必须符合下面的要求：
* 配置类必须以类的形式提供（不能是工厂方法返回的实例），允许通过生成子类在运行时增强（cglib 动态代理）。
* 配置类不能是 final 类（没法动态代理）。
* 配置注解通常为了通过 @Bean 注解生成 Spring 容器管理的类，
* 配置类必须是非本地的（即不能在方法中声明，不能是 private）。
* 任何嵌套配置类都必须声明为static。
* @Bean 方法可能不会反过来创建进一步的配置类（也就是返回的 bean 如果带有 @Configuration，也不会被特殊处理，只会作为普通的 bean）。

我们知道，Spring 容器在启动时，会加载默认的一些 PostProcessor，其中就有 ConfigurationClassPostProcessor，这个后置处理程序专门处理带有 @Configuration 注解的类，这个程序会在 bean 定义加载完成后，在 bean 初始化前进行处理。主要处理的过程就是使用 cglib 动态代理增强类，而且是对其中带有 @Bean 注解的方法进行处理。也就是说，所有带有 @Configuration 注解的 bean 会变成增强的类。
具体代码就不讲解了。

回到上面那个例子，总之：通过 beanFactory.getBean 获取 SimpleBean，如果已经创建了就会直接返回，如果没有执行过，就会通过 invokeSuper 首次执行。
因此我们在 @Configuration 注解定义的 bean 方法中可以直接调用方法，不需要 @Autowired 注入后使用。

#### 2、@Component注解
@Component 注解并没有通过 cglib 来代理@Bean 方法的调用，因此像下面这样配置时，就是两个不同的 SimpleBean。
```java
@Component
public static class Config {

    @Bean
    public SimpleBean simpleBean() {
        return new SimpleBean();
    }

    @Bean
    public SimpleBeanConsumer simpleBeanConsumer() {
        return new SimpleBeanConsumer(simpleBean());
    }
}
```

有些特殊情况下，我们不希望被代理（代理后会变成WebMvcConfig$$EnhancerBySpringCGLIB$$xxx）时，就得用 @Component，这种情况下，上面的写法就需要改成下面这样：
```java
@Component
public static class Config {
    @Autowired
    SimpleBean simpleBean;

    @Bean
    public SimpleBean simpleBean() {
        return new SimpleBean();
    }

    @Bean
    public SimpleBeanConsumer simpleBeanConsumer() {
        return new SimpleBeanConsumer(simpleBean);
    }
}
```