---
layout:     post
title:      Springboot之多数据源的配置使用
subtitle:   记录自己的成长
date:       2019-05-13
author:     BY
header-img: img/post-bg-law2.jpg
catalog: true
tags:

    - blog
    - springboot
    - java
---




# Springboot之多数据源的配置使用

原创（微信）： bboyHan 

公众号：23号杂货铺 

*1月22日*



## 01引入

现在的企业服务逐渐地呈现出数据的指数级增长趋势，无论从数据库的选型还是搭建，大多数的团队都开始考虑多样化的数据库来支撑存储服务。例如分布式数据库、Nosql数据库、内存数据库、关系型数据库等等。再到后端开发来说，服务的增多，必定需要考虑到多数据源的切换使用来兼容服务之间的调用。为解决这一难题，今天就来分享一个关于多数据源的切换使用配置。



**使用前提：**



- JDK8+
- Springboot
- IDEA
- Mysql5.5+
- lombok





## 02具体配置

使用spring AOP的思想，在进入orm之前，进行DataSource的Type切换。 添加maven依赖,我使用的是springboot-2.0.5版本。



```java
         <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-jdbc</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-aop</artifactId>
        </dependency>
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>druid-spring-boot-starter</artifactId>
            <version>1.1.10</version>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>

```

创建application.yml

```java
server:
  port: 8080

spring:
  profiles:
    active: dev
  application:
    name: @pom.artifactId@

  jpa:
    generate-ddl: false
    show-sql: true
    hibernate:
      ddl-auto: none

  datasource:
    default:
      url: jdbc:mysql://localhost:3306/bboyhan?characterEncoding=UTF-8&useUnicode=true&useSSL=false
      username: root
      password: root
      driver: com.mysql.jdbc.Driver
      type: com.alibaba.druid.pool.DruidDataSource

    names: tb1,tb2
    tb1:
      url: jdbc:mysql://localhost:3306/tb1?characterEncoding=UTF-8&useUnicode=true&useSSL=false
      username: root
      password: root
      driver: com.mysql.jdbc.Driver
    tb2:
      url: jdbc:mysql://localhost:3306/tb2?characterEncoding=UTF-8&useUnicode=true&useSSL=false
      username: root
      password: root
      driver: com.mysql.jdbc.Driver

```

创建注解类@DbName

```java
import java.lang.annotation.*;
/**
 * @Auther: bboyHan
 * @Date: 2019/1/22 18:12
 * @Description:
 */
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface DbName {
    String value();
}
```

创建aop切面类ChooseDbAspect

```java
import com.jkzx.common.annotation.DbName;
import com.jkzx.common.config.dbsource.DynamicDataSourceContextHolder;
import lombok.extern.slf4j.Slf4j;
import org.aspectj.lang.JoinPoint;
import org.aspectj.lang.annotation.After;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Before;
import org.aspectj.lang.annotation.Pointcut;
import org.springframework.core.annotation.Order;
import org.springframework.stereotype.Component;
/**
 * @Auther: bboyHan
 * @Date: 2019/1/22 18:13
 * @Description:
 */
@Component
@Order(-1)//保证在@Transactional之前执行
@Aspect
@Slf4j
public class ChooseDbAspect {

    @Pointcut("@annotation(com.bboyhan.annotation.DbName)")
    public void chooseDbPointCut(){}

    //改变数据源
    @Before("@annotation(dbName)")
    public void changeDataSource(JoinPoint joinPoint, DbName dbName) {
        String dbid = dbName.value();

        if (!DynamicDataSourceContextHolder.isContainsDataSource(dbid)) {
            //joinPoint.getSignature() ：获取连接点的方法签名对象
            log.error("数据源 " + dbid + " 不存在使用默认的数据源 -> " + joinPoint.getSignature());
        } else {
            log.debug("使用数据源：" + dbid);
            DynamicDataSourceContextHolder.setDataSourceType(dbid);
        }
    }

    @After("@annotation(dbName)")
    public void clearDataSource(JoinPoint joinPoint, DbName dbName) {
        log.debug("清除数据源 " + dbName.value() + " ! - start");
        DynamicDataSourceContextHolder.clearDataSourceType();
    }

}
```

使用ThreadLocal创建一个线程安全的类

```java
import lombok.extern.slf4j.Slf4j;

import java.util.ArrayList;
import java.util.List;

/**
 * @Auther: bboyHan
 * @Date: 2019/1/22 18:13
 * @Description:
 */
@Slf4j
public class DynamicDataSourceContextHolder {

    //存放当前线程使用的数据源类型信息
    private static final ThreadLocal<String> contextHolder = new ThreadLocal<String>();
    //存放数据源id
    public static List<String> dataSourceIds = new ArrayList<String>();

    //设置数据源
    public static void setDataSourceType(String dataSourceType) {
        contextHolder.set(dataSourceType);
    }

    //获取数据源
    public static String getDataSourceType() {
        return contextHolder.get();
    }

    //清除数据源
    public static void clearDataSourceType() {
        contextHolder.remove();
        log.info("清除数据源:{}", contextHolder.get() + " - end");
    }

    //判断当前数据源是否存在
    public static boolean isContainsDataSource(String dataSourceId) {
        return dataSourceIds.contains(dataSourceId);
    }
}

```

创建一个Register实现ImportBeanDefinitionRegistrar, EnvironmentAware

```java
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.MutablePropertyValues;
import org.springframework.beans.factory.support.BeanDefinitionRegistry;
import org.springframework.beans.factory.support.GenericBeanDefinition;
import org.springframework.boot.jdbc.DataSourceBuilder;
import org.springframework.context.EnvironmentAware;
import org.springframework.context.annotation.ImportBeanDefinitionRegistrar;
import org.springframework.core.env.Environment;
import org.springframework.core.type.AnnotationMetadata;

import javax.sql.DataSource;
import java.util.HashMap;
import java.util.Map;

/**
 * @Auther: bboyHan
 * @Date: 2019/1/22 18:19
 * @Description:
 */
@Slf4j
public class DynamicDataSourceRegister implements ImportBeanDefinitionRegistrar, EnvironmentAware {


    //指定默认数据源(springboot2.0默认数据源是hikari，在这里我使用DruidDataSource)
    private static final String DATASOURCE_TYPE_DEFAULT = "com.alibaba.druid.pool.DruidDataSource";
    //默认数据源
    private DataSource defaultDataSource;
    //用户自定义数据源
    private Map<String, DataSource> slaveDataSources = new HashMap<>();

    @Override
    public void setEnvironment(Environment environment) {
        initDefaultDataSource(environment);
        initMutilDataSources(environment);
    }

    private void initDefaultDataSource(Environment env) {
        // 读取主数据源,解析yml文件
        Map<String, Object> dsMap = new HashMap<>();
        dsMap.put("driver", env.getProperty("spring.datasource.default.driver"));
        dsMap.put("url", env.getProperty("spring.datasource.default.url"));
        dsMap.put("username", env.getProperty("spring.datasource.default.username"));
        dsMap.put("password", env.getProperty("spring.datasource.default.password"));
        dsMap.put("type", env.getProperty("spring.datasource.default.type"));
        defaultDataSource = buildDataSource(dsMap);
    }


    private void initMutilDataSources(Environment env) {
        // 读取配置文件获取更多数据源
        String dsPrefixs = env.getProperty("spring.datasource.names");
        for (String dsPrefix : dsPrefixs.split(",")) {
            // 多个数据源
            Map<String, Object> dsMap = new HashMap<>();
            dsMap.put("driver", env.getProperty("spring.datasource." + dsPrefix + ".driver"));
            dsMap.put("url", env.getProperty("spring.datasource." + dsPrefix + ".url"));
            dsMap.put("username", env.getProperty("spring.datasource." + dsPrefix + ".username"));
            dsMap.put("password", env.getProperty("spring.datasource." + dsPrefix + ".password"));
            DataSource ds = buildDataSource(dsMap);
            slaveDataSources.put(dsPrefix, ds);
        }
    }

    @Override
    public void registerBeanDefinitions(AnnotationMetadata annotationMetadata, BeanDefinitionRegistry beanDefinitionRegistry) {
        Map<Object, Object> targetDataSources = new HashMap<Object, Object>();
        //添加默认数据源
        targetDataSources.put("dataSource", this.defaultDataSource);
        DynamicDataSourceContextHolder.dataSourceIds.add("dataSource");
        //添加其他数据源
        targetDataSources.putAll(slaveDataSources);
        DynamicDataSourceContextHolder.dataSourceIds.addAll(slaveDataSources.keySet());

        //创建DynamicDataSource
        GenericBeanDefinition beanDefinition = new GenericBeanDefinition();
        beanDefinition.setBeanClass(DynamicDataSource.class);
        beanDefinition.setSynthetic(true);
        MutablePropertyValues mpv = beanDefinition.getPropertyValues();
        mpv.addPropertyValue("defaultTargetDataSource", defaultDataSource);
        mpv.addPropertyValue("targetDataSources", targetDataSources);
        //注册 - BeanDefinitionRegistry
        beanDefinitionRegistry.registerBeanDefinition("dataSource", beanDefinition);

        log.info("Dynamic DataSource Registry");
    }

    public DataSource buildDataSource(Map<String, Object> dataSourceMap) {
        try {
            Object type = dataSourceMap.get("type");
            if (type == null) {
                type = DATASOURCE_TYPE_DEFAULT;// 默认DataSource
            }

            Class<? extends DataSource> dataSourceType;
            dataSourceType = (Class<? extends DataSource>) Class.forName((String) type);
            String driverClassName = dataSourceMap.get("driver").toString();
            String url = dataSourceMap.get("url").toString();
            String username = dataSourceMap.get("username").toString();
            String password = dataSourceMap.get("password").toString();
            // 自定义DataSource配置
            DataSourceBuilder factory = DataSourceBuilder.create().driverClassName(driverClassName).url(url)
                    .username(username).password(password).type(dataSourceType);
            return factory.build();
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }
        return null;
    }
}
```

创建一个DynamicDataSource重写AbstractRoutingDataSource的determineCurrentLookupKey()方法

```java
import org.springframework.jdbc.datasource.lookup.AbstractRoutingDataSource;

/**
 * @Auther: bboyHan
 * @Date: 2019/1/22 18:20
 * @Description:
 */
public class DynamicDataSource extends AbstractRoutingDataSource {
    @Override
    protected Object determineCurrentLookupKey() {
        return DynamicDataSourceContextHolder.getDataSourceType();
    }
}
```

