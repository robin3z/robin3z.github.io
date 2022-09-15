---
title: "微服务利器之Nacos"
slug: "spring-boot-nacos-integration"
date: 2022-09-15T12:42:02Z
tags: [Java, Nacos]
---

阿里巴巴开源的 Nacos 同时拥有配置管理和服务管理的功能。 写个帖子，记录一下近几天学习 nacos 的收获。

## Srping Boot、Spring Cloud、Nacos 之间的版本兼容问题

发现用较新的 Spring Boot、Spring Cloud 版本玩不转 Nacos 后，于是开始了寻找兼容版本之旅。

Nacos 官方有这么一行说明:

> 注意：版本 0.2.x.RELEASE 对应的是 Spring Boot 2.x 版本， 版本 0.1.x.RELEASE 对应的是 Spring Boot 1.x 版本。

从 maven 中央仓库查到 spring-cloud-alibaba 的最新版本是 2.2.8.RELEASE，根据上面的那行 Nacos 官网“注意”说明， 莫不是 Spring Boot 得用 2.x 才能兼容当前版本？

最后版本选择如下：

* Nacos Server 2.1.1
* Spring Boot 2.2.9.RELEASE
* Spring Cloud Hoxton.SR10
* Spring Cloud Alibaba 2.2.8.RELEASE

## 实现动态切换 MySQL 数据源

在 Nacos 后台改了配置后， Nacos 会主动推送给应用。 Spring 应用只要对依赖于 Nacos 配置的 Bean 都加上 Spring Cloud 注解 @RefreshScope， 就能做到自动刷新配置了。 实际上，加了 @RefreshScope 的 Bean 会先销毁，然后重新创建。然而，使用了 Spring Boot 官方的自动数据源配置（DataSourceAutoConfiguration），是不会加 @RefreshScope 的。 只能自己配置数据源了，比如，可以按如下配置 HikariDataSource 数据源:

以下代码中， 没有使用 @Configuration 注解，而是使用了 @Component 注解 ， 这样 Spring Boot 的自动数据源配置就不会生效了。

```
@RefreshScope
@Component("dataSource")
public class DataSourceConfig extends HikariDataSource implements InitializingBean {

    @Value("${spring.datasource.url}")
    String url;

    @Value("${spring.datasource.username}")
    String username;

    @Value("${spring.datasource.password}")
    String password;

    @Value("${spring.datasource.driver-class-name}")
    String driverClassName;

    @Value("${spring.datasource.hikari.minimum-idle}")
    Integer minimumIdle;

    @Value("${spring.datasource.hikari.idle-timeout}")
    Long idleTimeout;

    @Value("${spring.datasource.hikari.maximum-pool-size}")
    Integer maximumPoolSize;

    @Value("${spring.datasource.hikari.auto-commit}")
    Boolean autoCommit;

    @Value("${spring.datasource.hikari.pool-name}")
    String poolName;

    @Value("${spring.datasource.hikari.max-lifetime}")
    Long maxLifetime;

    @Value("${spring.datasource.hikari.connection-timeout}")
    Long connectionTimeout;

    @Value("${spring.datasource.hikari.connection-test-query}")
    String connectionTestQuery;

    @Override
    public void afterPropertiesSet() {
        this.setJdbcUrl(url);
        this.setUsername(username);
        this.setPassword(password);
        this.setDriverClassName(driverClassName);
        this.setMaximumPoolSize(maximumPoolSize);
        this.setAutoCommit(autoCommit);
        this.setMaxLifetime(maxLifetime);
        this.setConnectionTimeout(connectionTimeout);
        this.setPoolName(poolName);
        this.setConnectionTestQuery(connectionTestQuery);
        this.setIdleTimeout(idleTimeout);
        this.setMinimumIdle(minimumIdle);
    }

}
```

## 使用 OpenFeign 调用微服务

有了 Nacos、OpenFeign、Ribbon， “服务发现、故障转移、负载均衡”这些功能通过添加几个注解 @EnableDiscoveryClient、@EnableFeignClients、@FeignClient、@LoadBalanced 就做到了。再也不用自己手动调用 RestTemplate、手动解析 JSON 了。
