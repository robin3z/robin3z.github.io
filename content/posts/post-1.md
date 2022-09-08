---
title: "Spring Boot 线程池的配置及使用场景"
filename: "spring-boot-thread-pool-config-and-basic-usage"
slug: "spring-boot-thread-pool-configuration-and-basic-usage"
date: 2022-09-08T11:26:00+08:30
description: ""
tags: [Java]
---

Spring Boot 的线程池是在自动配置类 “org.springframework.boot.autoconfigure.task.TaskExecutionAutoConfiguration” 中完成配置。
默认配置的线程池是 “org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor”。
这是自动完成的， 除非通过以下2中方式中的一种禁用 TaskExecutionAutoConfiguration 自动配置。

* 在应用的配置文件中添加如下配置项

`spring.autoconfigure.exclude=org.springframework.boot.autoconfigure.task.TaskExecutionAutoConfiguration`

* 在启动类中添加如下代码

`@SpringBootApplication(exclude={org.springframework.boot.autoconfigure.task.TaskExecutionAutoConfiguration.class})`

即使不使用线程池，其实也没必要去禁用 TaskExecutionAutoConfiguration 自动配置， “如何禁用”有时在调试时可以用到, 知道就好了。

---

## 1. 使用线程池首先需要做的事情就是在启动类添加 @EnableAsync 注解

比如以下是我常用的启动类配置
```
@EnableAsync
@SpringBootApplication
public class Application extends SpringBootServletInitializer {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }

    @Override
    protected SpringApplicationBuilder configure(SpringApplicationBuilder builder) {
        return builder.sources(Application.class);
    }

}
```
## 2. 在需要异步执行的bean对象方法上添加 @Async 注解。

添加了 @Async 后， 就可以通过打印线程id或者设置断点的方式， 测试一下方法是否按预期那样在线程池中执行了。 

因为 @Async 是通过动态代理技术实现的， 所以 @Async 不能加在静态方法上， 并且同一个类的不同方法间通过this相互调用是不会生效的。 这跟使用 @Transactional 注解开启事务时需要注意的事项是相同的。

## 3. 所有 spring boot 的线程池配置项

* spring.task.execution.pool.core-size

`核心线程数量，默认值为8。`

* spring.task.execution.pool.max-size

`救急线程最大数量，默认值为 Integer.MAX_VALUE， 也就是 Integer 类型所能取到的最大值。`

* spring.task.execution.pool.queue-capacity

`队列容量， 默认值为 Integer.MAX_VALUE 。只有核心线程用完，并且队列也已满了后，才会根据 max-size 创建新线程来执行任务。 如果 queue-capacity 设置的值很大， 那么 max-size 是没有机会生效的。`

* spring.task.execution.pool.allow-core-thread-timeout

`核心线程空闲超过一定时间后是否销毁，默认值为 true。`

* spring.task.execution.pool.keep-alive

`线程空闲多久后销毁，默认值 60s。`

* spring.task.execution.shutdown.await-termination

`关闭线程池时是否先等待任务执行完毕，默认值false。`

* spring.task.execution.shutdown.await-termination-period

`如果前面的 await-termination 设置为 true， 那么再设置 await-termination-period 值， 表示在关闭线程池时，等待任务执行完毕，最长等多久。`

* spring.task.execution.thread-name-prefix

`线程名称的前缀，默认值为 “task-”。`

**一般情况下，我会设置这几个重要参数 core-size、max-size、queue-capacity 的值；另外再设置 thread-name-prefix 的值， 方便于调试时查找线程。**

## 4. 线程池使用场景

### 4.1 配合消息队列的监听线程，加快消息的处理速度

从消息队列里获取消息的操作，速度一般是很快的，所有只要一个线程去监听队列就可以了。 但取到消息后， 根据消息执行业务逻辑一般是比较耗时的，这时可以把业务任务交给线程池处理，这样总体可以加快消息的处理速度。

有一种情况，当任务过多，线程池处理不过来、队列被填满，这时 Spring 会抛出 TaskRejectedException 异常。 这是因为消息获取的速度远远大于任务处理的速度，这时使用消息队列的手动ack机制，可以让两边的速度保持同步。

### 4.2 处理用户http请求时，将耗时的操作提交给线程池处理

比如处理完主要业务逻辑后，有一些收尾工作是不需要用户等待的。比如调用外部接口通知其他系统， 这时就可以交给线程池处理， 这样用户感受到的页面响应就变快了。

## 5. 自定义线程池以及异常处理

可以自己写一个配置类， 并实现 AsyncConfigurer 接口， 其中 getAsyncExecutor() 返回自定义的线程池， getAsyncUncaughtExceptionHandler() 返回实现了 AsyncUncaughtExceptionHandler 接口的类对象。 

这两个接口都可以返回null, 表示还是使用 Spring 的默认配置。 

我没碰到过需要自定义线程池的情况，所以研究不多， 后续研究线程池监控、动态修改线程池参数时，再回头看看是否需要用到自定义线程池。 

推荐阅读“美团技术团队”博客的一篇文章[Java线程池实现原理及其在美团业务中的实践](https://tech.meituan.com/2020/04/02/java-pooling-pratice-in-meituan.html)，美团对线程池的监控和参数可动态修改非常注重。
