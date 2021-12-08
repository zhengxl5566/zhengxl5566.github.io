---
layout: post
title:  "同学，你的多数据源事务失效了"
date:   2021-11-30 08:00:00 +0800
category: Java
author: Java课代表
excerpt: 多数据源的设计与实现中，一定要考虑事务管理和声明式事务，否则就是前人挖坑，后人认栽
---

## 一、引言

说起多数据源，一般会在如下两个场景中用到：

一是业务特殊，需要连接多个库。课代表曾做过一次新老系统迁移，由 `SQLServer` 迁移到 `MySQL` ，中间涉及一些业务运算，常用数据抽取工具无法满足业务需求，只能徒手撸。

二是数据库读写分离，在数据库主从架构下，写操作落到主库，读操作交给从库，用于分担主库压力。

多数据源的实现，从简单到复杂，有多种方案。

本文将以`SpringBoot(2.5.X)+Mybatis+H2`为例，演示一个**简单可靠**的多数据源实现。

读完本文你将收获：

1. `SpringBoot`是怎么自动配置数据源的
2. `SpringBoot`里的`Mybatis`是如何自动配置的
3. 多数据源下的事务如何使用
4. 得到一个可靠的多数据源样例工程

## 二、自动配置的数据源

`SpringBoot`的自动配置~~几乎~~帮我们完成了所有工作，只需要引入相关依赖即可完成所有工作

```java
<dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>org.mybatis.spring.boot</groupId>
    <artifactId>mybatis-spring-boot-starter</artifactId>
    <version>2.2.0</version>
</dependency>
```

当依赖中引入了`H2`数据库后，`DataSourceAutoConfiguration.java`会自动配置一个默认数据源:`HikariDataSource`，先贴源码：

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass({ DataSource.class, EmbeddedDatabaseType.class })
@ConditionalOnMissingBean(type = "io.r2dbc.spi.ConnectionFactory")
// 1、加载数据源配置
@EnableConfigurationProperties(DataSourceProperties.class)
@Import({ DataSourcePoolMetadataProvidersConfiguration.class,
      DataSourceInitializationConfiguration.InitializationSpecificCredentialsDataSourceInitializationConfiguration.class,
      DataSourceInitializationConfiguration.SharedCredentialsDataSourceInitializationConfiguration.class })
public class DataSourceAutoConfiguration {

   @Configuration(proxyBeanMethods = false)
   // 内嵌数据库依赖条件，默认存在 HikariDataSource 所以不会生效，详见下文
   @Conditional(EmbeddedDatabaseCondition.class)
   @ConditionalOnMissingBean({ DataSource.class, XADataSource.class })
   @Import(EmbeddedDataSourceConfiguration.class)
   protected static class EmbeddedDatabaseConfiguration {

   }

   @Configuration(proxyBeanMethods = false)
   @Conditional(PooledDataSourceCondition.class)
   @ConditionalOnMissingBean({ DataSource.class, XADataSource.class })
   @Import({ DataSourceConfiguration.Hikari.class, DataSourceConfiguration.Tomcat.class,
         DataSourceConfiguration.Dbcp2.class, DataSourceConfiguration.OracleUcp.class,
         DataSourceConfiguration.Generic.class, DataSourceJmxConfiguration.class })
   protected static class PooledDataSourceConfiguration {
   //2、初始化带池化的数据源：Hikari、Tomcat、Dbcp2等
   }
   // 省略其他
}
```

其原理如下：

### 1、加载数据源配置

通过`@EnableConfigurationProperties(DataSourceProperties.class)`加载配置信息，观察`DataSourceProperties`的类定义：

```java
@ConfigurationProperties(prefix = "spring.datasource")
public class DataSourceProperties implements BeanClassLoaderAware, InitializingBean
```

可以得到到两个信息：

1. 配置的前缀为`spring.datasource`;
2. 实现了`InitializingBean`接口，有初始化操作。

其实是根据用户配置初始化了一下默认的内嵌数据库连接：

```java
	@Override
	public void afterPropertiesSet() throws Exception {
		if (this.embeddedDatabaseConnection == null) {
			this.embeddedDatabaseConnection = EmbeddedDatabaseConnection.get(this.classLoader);
		}
	}
```

通过`EmbeddedDatabaseConnection.get`方法遍历内置的数据库枚举，找到最适合当前环境的内嵌数据库连接，由于我们引入了`H2`，所以返回值也是`H2`数据库的枚举信息：

```java
public static EmbeddedDatabaseConnection get(ClassLoader classLoader) {
		for (EmbeddedDatabaseConnection candidate : EmbeddedDatabaseConnection.values()) {
			if (candidate != NONE && ClassUtils.isPresent(candidate.getDriverClassName(), classLoader)) {
				return candidate;
			}
		}
		return NONE;
	}
```

这就是`SpringBoot` 的**convention over configuration** （约定优于配置）的思想，`SpringBoot`发现我们引入了`H2`数据库，就立马准备好了默认的连接信息。

### 2、创建数据源

默认情况下由于`SpringBoot`内置池化数据源`HikariDataSource`，所以`@Import(EmbeddedDataSourceConfiguration.class)`不会被加载，只会初始化一个`HikariDataSource`，原因是`@Conditional(EmbeddedDatabaseCondition.class)`在当前环境下不成立。这点在源码里的注释已经解释了：

```java
/**
 * {@link Condition} to detect when an embedded {@link DataSource} type can be used.
 
 * If a pooled {@link DataSource} is available, it will always be preferred to an
 * {@code EmbeddedDatabase}.
 * 如果存在池化 DataSource，其优先级将高于 EmbeddedDatabase
 */
static class EmbeddedDatabaseCondition extends SpringBootCondition {
// 省略源码
}
```

所以默认数据源的初始化是通过：`@Import({ DataSourceConfiguration.Hikari.class,//省略其他} `来实现的。代码也比较简单：

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass(HikariDataSource.class)
@ConditionalOnMissingBean(DataSource.class)
@ConditionalOnProperty(name = "spring.datasource.type", havingValue = "com.zaxxer.hikari.HikariDataSource",
      matchIfMissing = true)
static class Hikari {

   @Bean
   @ConfigurationProperties(prefix = "spring.datasource.hikari")
   HikariDataSource dataSource(DataSourceProperties properties) {
   //创建 HikariDataSource 实例 
      HikariDataSource dataSource = createDataSource(properties, HikariDataSource.class);
      if (StringUtils.hasText(properties.getName())) {
         dataSource.setPoolName(properties.getName());
      }
      return dataSource;
   }

}
```

```java
protected static <T> T createDataSource(DataSourceProperties properties, Class<? extends DataSource> type) {
// 在 initializeDataSourceBuilder 里面会用到默认的连接信息
return (T) properties.initializeDataSourceBuilder().type(type).build();
}
```

```java
public DataSourceBuilder<?> initializeDataSourceBuilder() {
   return DataSourceBuilder.create(getClassLoader()).type(getType()).driverClassName(determineDriverClassName())
         .url(determineUrl()).username(determineUsername()).password(determinePassword());
}
```

默认连接信息的使用都是同样的思想：优先使用用户指定的配置，如果用户没写，那就用默认的，以`determineDriverClassName()`为例：

```java
public String determineDriverClassName() {
    // 如果配置了 driverClassName 则返回
		if (StringUtils.hasText(this.driverClassName)) {
			Assert.state(driverClassIsLoadable(), () -> "Cannot load driver class: " + this.driverClassName);
			return this.driverClassName;
		}
		String driverClassName = null;
    // 如果配置了 url 则根据 url推导出 driverClassName
		if (StringUtils.hasText(this.url)) {
			driverClassName = DatabaseDriver.fromJdbcUrl(this.url).getDriverClassName();
		}
    // 还没有的话就用数据源配置类初始化时获取的枚举信息填充
		if (!StringUtils.hasText(driverClassName)) {
			driverClassName = this.embeddedDatabaseConnection.getDriverClassName();
		}
		if (!StringUtils.hasText(driverClassName)) {
			throw new DataSourceBeanCreationException("Failed to determine a suitable driver class", this,
					this.embeddedDatabaseConnection);
		}
		return driverClassName;
	}
```

其他诸如`determineUrl()`，`determineUsername()`，`determinePassword()`道理都一样，不再赘述。

至此，默认的`HikariDataSource`就自动配置好了！

接下来看一下`Mybatis`在`SpringBoot`中是如何自动配置起来的

## 三、自动配置`Mybatis`

要想在`Spring`中使用`Mybatis`，至少需要一个`SqlSessionFactory` 和一个` mapper`接口，所以，`MyBatis-Spring-Boot-Starter `为我们做了这些事：

1. 自动发现已有的`DataSource`
2. 将`DataSource`传递给`SqlSessionFactoryBean` 从而创建并注册一个`SqlSessionFactory` 实例
3. 利用`sqlSessionFactory` 创建并注册 `SqlSessionTemplate` 实例
4. 自动扫描`mapper`，将他们与`SqlSessionTemplate` 链接起来并注册到`Spring` 容器中供其他`Bean`注入

结合源码加深印象：

```
public class MybatisAutoConfiguration implements InitializingBean {
    @Bean
    @ConditionalOnMissingBean
    //1.自动发现已有的`DataSource`
    public SqlSessionFactory sqlSessionFactory(DataSource dataSource) throws Exception {
        SqlSessionFactoryBean factory = new SqlSessionFactoryBean();
        //2.将 DataSource 传递给 SqlSessionFactoryBean 从而创建并注册一个 SqlSessionFactory 实例
        factory.setDataSource(dataSource);
       // 省略其他...
        return factory.getObject();
    }

    @Bean
    @ConditionalOnMissingBean
    //3.利用 sqlSessionFactory 创建并注册 SqlSessionTemplate 实例
    public SqlSessionTemplate sqlSessionTemplate(SqlSessionFactory sqlSessionFactory) {
        ExecutorType executorType = this.properties.getExecutorType();
        if (executorType != null) {
            return new SqlSessionTemplate(sqlSessionFactory, executorType);
        } else {
            return new SqlSessionTemplate(sqlSessionFactory);
        }
    }

    /**
     * This will just scan the same base package as Spring Boot does. If you want more power, you can explicitly use
     * {@link org.mybatis.spring.annotation.MapperScan} but this will get typed mappers working correctly, out-of-the-box,
     * similar to using Spring Data JPA repositories.
     */
     //4.自动扫描`mapper`，将他们与`SqlSessionTemplate` 链接起来并注册到`Spring` 容器中供其他`Bean`注入
    public static class AutoConfiguredMapperScannerRegistrar implements BeanFactoryAware, ImportBeanDefinitionRegistrar {
	// 省略其他...

    }

}
```

一图胜千言，其本质就是层层注入：

![mybatis-inject](https://zhengxl5566.github.io/img/article-img/2021-11/mybatis-inject.png)

## 四、由单变多

有了二、三小结的知识储备，创建多数据源的理论基础就有了：搞两套`DataSource`，搞两套层层注入，如图：

![mybatis-inject](https://zhengxl5566.github.io/img/article-img/2021-11/mybatis-inject2.png)

接下来我们就照搬自动配置单数据源的套路配置一下多数据源，顺序如下：

![step](https://zhengxl5566.github.io/img/article-img/2021-11/step.png)

首先设计一下配置信息，单数据源时，配置前缀为`spring.datasource`，为了支持多个，我们在后面再加一层，`yml`如下：

```yaml
spring:
  datasource:
    first:
      driver-class-name: org.h2.Driver
      jdbc-url: jdbc:h2:mem:db1
      username: sa
      password:
    second:
      driver-class-name: org.h2.Driver
      jdbc-url: jdbc:h2:mem:db2
      username: sa
      password:
```

`first`数据源的配置

```java
/**
 * @description:
 * @author:Java课代表
 * @createTime:2021/11/3 23:13
 */
@Configuration
//配置 mapper 的扫描位置，指定相应的 sqlSessionTemplate
@MapperScan(basePackages = "top.javahelper.multidatasources.mapper.first", sqlSessionTemplateRef = "firstSqlSessionTemplate")
public class FirstDataSourceConfig {

    @Bean
    @Primary
    // 读取配置，创建数据源
    @ConfigurationProperties(prefix = "spring.datasource.first")
    public DataSource firstDataSource() {
        return DataSourceBuilder.create().build();
    }

    @Bean
    @Primary
    // 创建 SqlSessionFactory
    public SqlSessionFactory firstSqlSessionFactory(DataSource dataSource) throws Exception {
        SqlSessionFactoryBean bean = new SqlSessionFactoryBean();
        bean.setDataSource(dataSource);
        // 设置 xml 的扫描路径
        bean.setMapperLocations(new PathMatchingResourcePatternResolver().getResources("classpath:mybatis/first/*.xml"));
        bean.setTypeAliasesPackage("top.javahelper.multidatasources.entity");
        org.apache.ibatis.session.Configuration config = new org.apache.ibatis.session.Configuration();
        config.setMapUnderscoreToCamelCase(true);
        bean.setConfiguration(config);
        return bean.getObject();
    }

    @Bean
    @Primary
    // 创建 SqlSessionTemplate
    public SqlSessionTemplate firstSqlSessionTemplate(SqlSessionFactory sqlSessionFactory) {
        return new SqlSessionTemplate(sqlSessionFactory);
    }

    @Bean
    @Primary
    // 创建 DataSourceTransactionManager 用于事务管理
    public DataSourceTransactionManager firstTransactionManager(DataSource dataSource) {
        return new DataSourceTransactionManager(dataSource);
    }
}
```

这里每个`@Bean`都添加了`@Primary`使其成为默认`Bean`，`@MapperScan`使用的时候指定`SqlSessionTemplate`，将`mapper`与`firstSqlSessionTemplate`联系起来。

> 小贴士：
>
> 最后还为该数据源创建了一个`DataSourceTransactionManager`，用于事务管理，在多数据源场景下使用事务时通过`@Transactional(transactionManager = "firstTransactionManager")`用来指定该事务使用哪个事务管理。

至此，第一个数据源就配置好了，第二个数据源也是配置这些项目，因为配置的Bean类型相同，所以需要使用`@Qualifier`来限定装载的`Bean`，例如：

```java
@Bean
// 创建 SqlSessionTemplate
public SqlSessionTemplate secondSqlSessionTemplate(@Qualifier("secondSqlSessionFactory") SqlSessionFactory sqlSessionFactory) {
    return new SqlSessionTemplate(sqlSessionFactory);
}
```

完整代码可查看[课代表的GitHub](https://github.com/zhengxl5566/springboot-demo)

## 五、多数据源下的事务

`Spring`为我们提供了简单易用的声明式事务，使我们可以更专注于业务开发，但是想要用对用好却并不容易，本文只聚焦多数据源，关于事务补课请戳：[Spring 声明式事务应该怎么学？](https://mp.weixin.qq.com/s/O-haW7q2nfRRCJVbad4aOQ)

前文的小贴士里已经提到了开启声明式事务时由于有多个事务管理器存在，需要显示指定使用哪个事务管理器，比如下面的例子：

```java
// 不显式指定参数 transactionManager 则会使用设置为 Primary 的 firstTransactionManager
// 如下代码只会回滚 firstUserMapper.insert， secondUserMapper.insert(user2);会正常插入
@Transactional(rollbackFor = Throwable.class,transactionManager = "firstTransactionManager")
public void insertTwoDBWithTX(String name) {
    User user = new User();
    user.setName(name);
    // 回滚
    firstUserMapper.insert(user);
    // 不回滚
    secondUserMapper.insert(user);

    // 主动触发回滚
    int i = 1/0;
}
```

该事务默认使用`firstTransactionManager`作为事务管理器，只会控制`FristDataSource`的事务，所以当我们从内部手动抛出异常用于回滚事务时，`firstUserMapper.insert(user);`回滚，`secondUserMapper.insert(user);`不回滚。

框架代码均已上传，小伙伴们可以按照自己的想法设计用例验证。

## 六、回顾

至此，`SpringBoot+Mybatis+H2`的多数据源样例就演示完了，这应该是一个最基础的多数据源配置，事实上，线上很少这么用，除非是极其简单的一次性业务。

因为这个方式缺点非常明显：代码侵入性太强！有多少数据源，就要实现多少套组件，代码量成倍增长。

写这个案例更多地是总结回顾`SpringBoot`的自动配置，注解式声明`Bean`，`Spring`声明式事务等基础知识，为后面的多数据源进阶做铺垫。

Spring 官方为我们提供了一个`AbstractRoutingDataSource`类，通过对`DataSource`进行路由，实现多数据源的切换。这也是目前，大多数轻量级多数据源实现的底层支撑。

关注课代表，下一篇演示基于`AbstractRoutingDataSource+AOP`的多数据源实现！

## 参考

 [mybatis-spring](https://mybatis.org/spring/zh/index.html)

[mybatis-spring-boot-autoconfigure](http://mybatis.org/spring-boot-starter/mybatis-spring-boot-autoconfigure/#)

[课代表的GitHub](https://github.com/zhengxl5566/springboot-demo)

