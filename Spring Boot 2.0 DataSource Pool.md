之前在做项目的时候用到了Spring Boot 2.0，这里简单介绍一下Spring Boot 2.0 连接池的使用和遇到的一些坑。

Spring Boot 默认使用的连接池是 [HikariCP](https://github.com/brettwooldridge/HikariCP) ，当在在依赖中引入了

`spring-boot-starter-jdbc` 或者 `spring-boot-starter-data-jpa` ，且application.properties中没有配置spring.datasource.type 为其他的DataSource时。Spring Boot 会默认使用HiKariCP，如果需要使用其他的连接池，例如tomcat 连接池，则需要引入 tomcat-jdbc 依赖：

```xml
<dependency>
    <groupId>org.apache.tomcat</groupId>
    <artifactId>tomcat-jdbc</artifactId>
    <version>8.5.23</version>
</dependency>
```

并且在application.properties中配置spring.datasource.type=org.apache.tomcat.jdbc.pool.DataSource

作者使用的yml：

```properties
spring:
  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    username: xxx
    password: xxxx
    url: jdbc:mysql://localhost:3306/friends?useUnicode=true&character_set_server=utf8mb4&characterEncoding=utf8&useLegacyDatetimeCode=false&serverTimezone=GMT%2B8

    type: org.apache.tomcat.jdbc.pool.DataSource
```



之前在做项目的时候，使用Spring Boot数据库连接池遇到过一些问题，在这里总结一下。

因为做项目用的是MySQL，MySQL有个默认机制是八个小时空闲连接会自动回收，这个回收是在MySQL的Server端，而Spring Boot 的连接池不知道这个连接已经不可用了，继续用这个连接查询的话就会报错，所以需要在配置数据库连接池的时候增加一些检测配置，把一些不可用的连接释放掉。 

用HiKariCP需要以下配置（当使用的jdbc不是jdbc4的时候）：

```properties
spring:
  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    username: xxx
    password: xxxx
    url: jdbc:mysql://localhost:3306/friends?useUnicode=true&character_set_server=utf8mb4&characterEncoding=utf8&useLegacyDatetimeCode=false&serverTimezone=GMT%2B8
    hikari:
      maximum-pool-size: 10 连接池最大连接数
      minimum-idle: 0 允许的最小空闲属
      idle-timeout: 180000 #空闲超时是 180000 毫秒，当数据库连接的空闲时间大于180000毫秒时，这些空闲超时的连接会被关闭，直到超时的空闲连接数达到 minimum-idle的值
      connection-test-query: select 1 # 测试连接是否可用的query 语句 在oracle是 select 1 from dual
```

如果是tomcat 可能还要加上一个检测空闲的间隔时间。

一下介绍一些常用的HiKariCP的配置参数，读者也可以自己浏览HiKariCP的[GitHub](https://github.com/brettwooldridge/HikariCP)：

|                     |                                                              |
| ------------------- | ------------------------------------------------------------ |
| dataSourceClassName | 这是JDBC驱动程序提供的DataSource类的名称。 请参阅特定JDBC驱动程序的文档以获取此类名.注意不支持XA数据源。 XA需要像bitronix这样的真正的事务管理器。 请注意，如果您使用jdbcUrl进行“old-school”基于DriverManager的JDBC驱动程序配置，则不需要此属性 |
| jdbcUrl             | 数据库连接url                                                |
| username            | 用户名                                                       |
| password            | 密码                                                         |
| autoCommit          | 自动提交，默认是true                                         |
| connectionTimeout   | 连接超时时间设置，默认是30s                                  |
| idleTimeout         | 最大空闲时间，超过最大空闲时间的连接在达到最小空闲连接数 minimumIdle 就不会被回收。默认十分钟 |
| minimumIdle         | 允许的最小空闲数。当minimumIdl大于等于 maximumPoolSize 时，空闲超时的连接不会被回收。默认是等于 maximumPoolSize |
| maximumPoolSize     | 最大连接数，默认10                                           |
| connectionTestQuery | 测试连接是否可用的query语句                                  |
| maxLifetime         | 最长生命周期，使用中的连接永远不会退役，只有当它关闭时才会被删除。默认30分钟 |

如有需要后续再补充一些tomcat 连接池的。