---
title: Spring profile
date: 2019-03-05 19:45:25
tags: spring
categories: spring
---
### 概念
一个`profile`就是一个有名称的`bean`逻辑定义组，只有`profile`被激活后，该组`bean`才会被注册到容器中。当我们需要在不同的环境中使用不同的bean时，`profile`就非常有用，例如：开发、生产环境使用不同的配置。
<!-- more -->
### 定义`profile`
#### 使用`@Profile`
`@Profile`注解可以被用在class或者method上，代表只有当指定的`profile`激活，该bean才会被注册。当`@Profile`与`@Configuration`同时使用时，代表所有`@Bean`与`@Import`关联的class只有指定`profile`激活才能被注册为bean。
```
@Configuration
@Profile("development")
public class StandaloneDataConfig {

    @Bean
    public DataSource dataSource() {
        return new EmbeddedDatabaseBuilder()
            .setType(EmbeddedDatabaseType.HSQL)
            .addScript("classpath:com/bank/config/sql/schema.sql")
            .addScript("classpath:com/bank/config/sql/test-data.sql")
            .build();
    }
}
```
```
@Configuration
@Profile("production")
public class JndiDataConfig {

    @Bean(destroyMethod="")
    public DataSource dataSource() throws Exception {
        Context ctx = new InitialContext();
        return (DataSource) ctx.lookup("java:comp/env/jdbc/datasource");
    }
}
```
```
@Configuration
public class AppConfig {

    @Bean("dataSource")
    @Profile("development")
    public DataSource standaloneDataSource() {
        return new EmbeddedDatabaseBuilder()
            .setType(EmbeddedDatabaseType.HSQL)
            .addScript("classpath:com/bank/config/sql/schema.sql")
            .addScript("classpath:com/bank/config/sql/test-data.sql")
            .build();
    }

    @Bean("dataSource")
    @Profile("production")
    public DataSource jndiDataSource() throws Exception {
        Context ctx = new InitialContext();
        return (DataSource) ctx.lookup("java:comp/env/jdbc/datasource");
    }
}
```
`@Profile`中的的字符串，可以是`profile`的名称或者`profile`表达式。`profile`支持以下表达式：
- `!` ：非
- `&` ：与
- `|` ：或

如果不适用括号，不能同时使用`&`和`|`，例如：`production & us-east | eu-central`，必须为`production & (us-east | eu-central)`。
`@Profile({"dev","pro"})` 代表`dev`或`pro` profile。
可以使用`@Profile`作为元注解创建复合注解，例如，使用`Production`代替`@Profile("production")`
```
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Profile("production")
public @interface Production {
}
```
#### 使用xml定义
通过`<beans/>`的`profile`属性定义。
```
<beans profile="development"
    xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:jdbc="http://www.springframework.org/schema/jdbc"
    xsi:schemaLocation="...">

    <jdbc:embedded-database id="dataSource">
        <jdbc:script location="classpath:com/bank/config/sql/schema.sql"/>
        <jdbc:script location="classpath:com/bank/config/sql/test-data.sql"/>
    </jdbc:embedded-database>
</beans>
```
或者
```
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:jdbc="http://www.springframework.org/schema/jdbc"
    xmlns:jee="http://www.springframework.org/schema/jee"
    xsi:schemaLocation="...">

    <!-- other bean definitions -->

    <beans profile="development">
        <jdbc:embedded-database id="dataSource">
            <jdbc:script location="classpath:com/bank/config/sql/schema.sql"/>
            <jdbc:script location="classpath:com/bank/config/sql/test-data.sql"/>
        </jdbc:embedded-database>
    </beans>

    <beans profile="production">
        <jee:jndi-lookup id="dataSource" jndi-name="java:comp/env/jdbc/datasource"/>
    </beans>
</beans>
```
在上述方式中，`<beans>`只能放在文件的最后，否则会报解析错误。
xml中只能使用`!`操作符，不能使用`&`、`|`。可以通过以下方式达到`&`的效果
```
<beans profile="production">
    <beans profile="us-east">
        <jee:jndi-lookup id="dataSource" jndi-name="java:comp/env/jdbc/datasource"/>
    </beans>
</beans>
```

### 激活profile
在定义了profile后，还需要指定激活哪一个profile。有以下几种激活方式：
1. 通过`Environment` API
```
AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext();
ctx.getEnvironment().setActiveProfiles("development");
ctx.refresh();
```
2. `spring.profiles.active`属性
通过设置`系统环境变量`、`JVM系统参数`、`servlet上下文参数`（web.xml中）的`spring.profiles.active`属性
```
System.setProperty("spring.profiles.active", "pro");

//jvm启动参数
-Dspring.profiles.active="profile1,profile2"

//servlet环境
<context-param>  
    <param-name>spring.profiles.active</param-name>  
    <param-value>dev</param-value>  
</context-param>  
```
3. 测试模式中使用`spring-test`的`@ActiveProfiles`注解。
```
/**
 * Test profile
 *
 * @author fei
 */
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration("classpath:application.xml")
@ActiveProfiles("pro")
public class SpringDemoApplicationTest {

    @Autowired
    private TestBean testBean;

    @Test
    public void testProfile() {
        testBean.hello();
    }
}
```

### 默认profile
如果没有激活任何profile，则默认profile会生效。
默认profile的名称默认为`default`，可以通过以下方式修改默认名称：
1. 通过设置`系统环境变量`、`JVM系统参数`、`servlet上下文参数`（web.xml中）的`spring.profiles.default`属性。
2. `Environment.setDefaultProfiles()`方法。
