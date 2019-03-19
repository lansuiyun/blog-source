---
title: Spring-properties
date: 2019-03-05 19:48:09
tags: spring
categories: spring
---
在spring中，有多种配置、使用`Properties`的方式。

<!-- more -->
### 准备属性文件
在`resurces`下增加`dev/config.properties`文件，添加内容：
```
server.name=gxj
```

### util:properties
#### 配置：
```
<util:properties id="config" location="classpath:dev/config.properties"/>
```
bean 配置方式
```
<!-- creates a java.util.Properties instance with values loaded from the supplied location -->
<bean id="config" class="org.springframework.beans.factory.config.PropertiesFactoryBean">
    <property name="location" value="classpath:dev/config.properties"/>
</bean>
```
#### 读取：
```
@Value("#{config['server.name']}")
```
或者在xml配置中读取：
```
<bean id="name" class="java.lang.String">
  <constructor-arg value="#{config['server.name']}"/>
</bean>
```
注意：
```
ConfigurableEnvironment environment = context.getEnvironment();
//name为null
String name = environment.getProperty("server.name")
```
结果为null。

### context:property-placeholder
#### 配置：
```
<context:property-placeholder location="classpath:dev/config.properties"/>
```
如果有多个配置文件，用`,`隔开：
```
<context:property-placeholder location="classpath:dev/config.properties,classpath:dev/config1.properties"/>
```
bean 配置方式：
```
<bean id="propertyConfigurer" class="org.springframework.beans.factory.config.PropertyPlaceholderConfigurer">
  <property name="location" value="classpath:dev/config.properties"/>
</bean>
```
多个配置文件使用`locations`属性：
```
<bean id="propertyConfigurer" class="org.springframework.beans.factory.config.PropertyPlaceholderConfigurer">
  <property name="locations">
    <list>
      <value>classpath:dev/config.properties</value>
      <value>classpath:dev/config1.properties</value>
    </list>
  </property>
</bean>
```

#### 读取：
```
@Value("${server.name}")
private String name;
```
或者在xml配置中读取：
```
<bean id="name" class="java.lang.String">
  <constructor-arg value="${pro.name}"/>
</bean>
```
同样，直接用`ConfigurableEnvironment`获取为null。

### `@PropertySource`方式
```
@Component
@PropertySource({"classpath:dev/config.properties","classpath:dev/config1.properties"})
public class TestBeanImpl2 implements TestBean {

    @Value("${pro.name}")
    private String name;

    ....
}
```
使用这种方式可以使用`Environment` 获取属性。
