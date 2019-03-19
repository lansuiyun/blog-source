---
title: spring 注解
date: 2019-03-07 17:01:07
tags: spring
categories: spring
---
spring 中常用的注解，不包括spring mvc、spring boot中的注解。
<!-- more-->
### 声明bean
- `@Component` 通用bean注解，可以设置bean名称
- `@Service` 业务逻辑bean注解
- `@Repository` 数据bean注解
- `@Controller` web controller 注解

`@Component`是通用bean注解，其他三个是特定组件的bean注解。应尽量使用对应组件的注解，这样通过工具进行处理、与切面相关联会更适合。并且spring可能会在后续版本对`@Service`、`@Repository`、`@Controller`增加语义。

spring中的很多注解都可以作为`元注解（meta-annotation）`来使用。例如，`@Service`就是通过`@Component`进行元注释的。
```
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Component
public @interface Service {

	/**
	 * The value may indicate a suggestion for a logical component name,
	 * to be turned into a Spring bean in case of an autodetected component.
	 * @return the suggested component name, if any (or empty String otherwise)
	 */
	@AliasFor(annotation = Component.class)
	String value() default "";

}
```

class在添加了上述的bean注解后，还需要被spring扫描才会注册为bean，有以下两种方式：
1. 通过`ComponentScan`注解
```
@Configuration
@ComponentScan(basePackages = "org.example")
public class AppConfig  {
    ...
}
```
2. 通过xml配置
```
<context:component-scan base-package="org.example"/>
```

### 注入bean
- `@Autowired` spring提供。通过`required`声明依赖是否必须。
- `@Inject`  JSR-330提供的注解
- `@Resource` JSR-250提供的注解

`@Autowired`与`@Inject`在功能上一致，可以注解在构造函数、属性、方法、参数上。

匹配逻辑：
1. 按照类型匹配
2. 使用限定符`Qualifier`进行限定
3. 按照名称匹配


`@Resource`可以注解在属性、方法、参数上，匹配逻辑与其他两个有所不同：
1. 按照名称匹配。首先匹配`name`属性，如果`name`没有设置，则匹配属性或者方法名称
2. 按照类型匹配
3. 使用限定符进行限定（但如果名称匹配成功的话这条会被忽略）

`@Autowired`、`@Inject`、`@Value`由[AutowiredAnnotationBeanPostProcessor](https://docs.spring.io/spring/docs/5.1.5.RELEASE/javadoc-api/org/springframework/beans/factory/annotation/AutowiredAnnotationBeanPostProcessor.html)类实现注入。`@Resource`、`@PostConstruct`、`@PreDestroy`由[CommonAnnotationBeanPostProcessor](https://docs.spring.io/spring/docs/5.1.5.RELEASE/javadoc-api/org/springframework/context/annotation/CommonAnnotationBeanPostProcessor.html)类实现实现注入。所以使用这些注解不能将bean注入到`BeanPostProcessor`、`BeanFactoryPostProcessor`类型中。
可以通过xml配置`context:annotation-config`或者`context:component-scan`注册`AutowiredAnnotationBeanPostProcessor`和`CommonAnnotationBeanPostProcessor`，开启上述注解。

#### 相关注解`@Qualifier`
`@Qualifier`可以与上述注解一起使用，对注入的bean做限制。


### java配置相关
- `@Configuration`
作用于class，代表被注解的类为bean定义源，相当于xml配置文件中的<beans>。
被注释的class本身也会被注册为bean，所以可以在内部使用`@Autowired`注入其他bean。
- `@Bean`
作用于方法上，代表方法的返回值会被作为一个bean，相当于xml配置中的<bean/>。
- `@ComponentScan`
配合`@Configuration`使用，用来扫描指定路径，检测bean声明、注入等，相当于xml配置中的`<context:component-scan>`。可以指定扫描路径、过滤器。
```
@Configuration
@ComponentScan(basePackages = "org.example",
        includeFilters = @Filter(type = FilterType.REGEX, pattern = ".*Stub.*Repository"),
        excludeFilters = @Filter(Repository.class))
public class AppConfig {
    ...
}
```
- `@Import`
配合`@Configuration`使用，可以从另一个配置类中加载bean定义。相当于xml配置中的`<import/>`
```
@Configuration
public class ConfigA {

    @Bean
    public A a() {
        return new A();
    }
}

@Configuration
@Import(ConfigA.class)
public class ConfigB {

    @Bean
    public B b() {
        return new B();
    }
}
```

除了被`@Configuration`注解的class可以作为`@Import`的参数外，`ImportSelector`、`ImportBeanDefinitionRegistrar`实现类也可以作为参数。

### bean行为相关
1. `@Scope`
指定bean的作用范围（即生命周期，默认为`singleton`），可以配合`@Component`或`@Bean`使用。等同于xml中的`scope`，如下：
```
<bean id="accountService" class="com.something.DefaultAccountService" scope="singleton"/>
```
有以下几种scope
  - `singleton` 默认值，容器内一个bean定义只有一个对象实例
  - `prototype` 容器内一个bean定义可以有任意多个实例
  - `request` 一次request请求就有一个新的实例
  - `session` sessin生命周期内只有一个实例
  - `application` ServletContext生命周期内只有一个实例


2. `@PostConstruct` `@PreDestory`
两个注解由`JSR-250`提供，分别在设置bean属性后与销毁前执行。作用相当于xml配置的`init-method`、`destroy-method`，接口`InitializingBean`、`DisposableBean`。同时设置后，调用顺序如下：
启动：
	1. `@PostConstruct`
	2. `InitializingBean接口`
	3. `init-method`
关闭：
	1. `@PreDestroy`
	2. `DisposableBean接口`
	3. `destroy-method`


3. `@DependsOn`
指定当前bean依赖于某个或某些bean，确保容器在创建当前bean前创建依赖的bean。当某些bean没有直接依赖关系时，使用`@DependsOn`可以明确依赖关系。相当于xml配置中的`<bean ... depends-on=""/>`.

4. `@Lazy`
spring 在默认情况下会尽可能早的创建bean，使用`@Lazy`可以使用spring在bean第一次被请求时才创建。
配合`@Component`、`@COnfiguration`、`@Bean`使用。
相当于xml配置中的`<bean ... lazy-init="true"/>`

### AOP相关
1. `@Aspect` 注释在类上，用于将一个普通的java类声明为切面。
2. `@Pointcut` 注释在方法上，定义切点表达式，即定义什么方法会被认为是切点。
3. 切点表达式相关注解
	`@target` 匹配被指定注解标记的被代理类的方法。
	`@args` 匹配运行时实际参数被指定注解标记的方法。
	`@within` 匹配被指定注解标记类的方法。
	`@annotation` 匹配被指定注解标记的方法。
4. 增强相关注解，定义何时执行增强。
	`@Before` 方法调用前增强。
	`@AfterReturning` 方法正常返回后增强，不能抛出异常。
	`@AfterThrowing` 方式抛出异常后。
	`@After` 方法返回后，需要程序去区分正常返回与异常返回。
5. `@DeclareParents` 为被增强的对象实现一个给定接口，并提供默认的实现.
6. `@EnableAspectJAutoProxy` 开启对`@Aspect`的支持，作用相当于xml配置中的`<aop:aspectj-autoproxy>`，与`@Configuration`配合使用。

### profile相关
1. `@Profile` 指定class的profile
2. `@ActiveProfiles` spring test中指定激活的profile


### 测试相关
1. `@RunWith` `@RunWith(SpringJUnit4ClassRunner.class)`
2. `@ContextConfiguration` 注解在测试类上，用于指定如何加载配置。
3. `@ActiveProfiles` spring test中指定激活的profile

### 资源相关
1. `@PropertySource`
将代表指定资源的`PropertySource` 加入到 Spring的`Environment`中。之后可以通过`@Value("${key}")`或者注入`Environment` 访问属性。该注解必须与`@Component`或其相关注解（`@Configuration`等）配合使用，否则无效。例如，通过xml配置定义bean，并在类上添加`@PropertySource`注解，结果并不能获取到属性。
```
@Component
@PropertySource({"classpath:dev/config.properties","classpath:dev/config1.properties"})
public class TestBeanImpl2 implements TestBean {

    @Value("${pro.name}")
    private String name;

    ....
}
```
2. `@ImportResource`
类似与xml配置中的`<import/>`，在使用java配置，通过`AnnotationConfigApplicationContext`初始化时，通过`@ImportResource`配合`@Configuration`可以导入xml的定义。
```
@Configuration
@ImportResource("classpath:/com/acme/properties-config.xml")
public class AppConfig {

    @Value("${jdbc.url}")
    private String url;

    @Value("${jdbc.username}")
    private String username;

    @Value("${jdbc.password}")
    private String password;

    @Bean
    public DataSource dataSource() {
        return new DriverManagerDataSource(url, username, password);
    }
}
```
properties-config.xml
```
<beans>
    <context:property-placeholder location="classpath:/com/acme/jdbc.properties"/>
</beans>
```

### 其他
#### `@Value`
为属性注入值，有以下集中使用方式：
1. 固定值
```
@Value("hello")
private String m;
```
2. System的属性，相当于`System.getProperty(key);`
```
@Value("#{systemProperties.message}")
private String m;
```
3. 其他bean属性，
```
@Value("#{bean.m}")
private String m;
```
4. Resource
```
@Value("classpath:pro/config1.properties")
private Resource resource;
```

5. 表达式结果
```
@Value("#{ T(java.util.concurrent.ThreadLocalRandom).current().nextInt() }")
private int num;
```
