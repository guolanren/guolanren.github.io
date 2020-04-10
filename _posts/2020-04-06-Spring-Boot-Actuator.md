---
title: Spring-Boot-Actuator
description: 当应用程序投入生产时，可以使用 Spring Boot Actuator 来监视和管理。它提供了 HTTP 端点和 JMX 来管理和监视应用程序。
categories: 
 - code
tags:
 - spring boot
 - spring boot example
 - actuator
updated: 2020-04-10
---

## 概述

​	当应用程序投入生产时，可以使用 [Spring Boot Actuator](<https://docs.spring.io/spring-boot/docs/current/reference/html/production-ready-features.html>) 来监视和管理。它提供了 **HTTP** 端点和 **JMX** 来管理和监视应用程序。

## 依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

## 端点

​	通常选择使用 **HTTP** 端点。下面会列举一系列的端点，它们默认前缀是 `/actuator`。

### 通用

| ID               | **Description**                                              |
| ---------------- | ------------------------------------------------------------ |
| auditevents      | Exposes audit events information for the current application. Requires an `AuditEventRepository` bean. |
| beans            | Displays a complete list of all the Spring beans in your application. |
| caches           | Exposes available caches.                                    |
| conditions       | Shows the conditions that were evaluated on configuration and auto-configuration classes and the reasons why they did or did not match. |
| configprops      | Displays a collated list of all `@ConfigurationProperties`.  |
| env              | Exposes properties from Spring’s `ConfigurableEnvironment`.  |
| flyway           | Shows any Flyway database migrations that have been applied. Requires one or more `Flyway` beans. |
| health           | Shows application health information.                        |
| httptrace        | Displays HTTP trace information (by default, the last 100 HTTP request-response exchanges). Requires an `HttpTraceRepository` bean. |
| info             | Displays arbitrary application info.                         |
| integrationgraph | Shows the Spring Integration graph. Requires a dependency on `spring-integration-core`. |
| loggers          | Shows and modifies the configuration of loggers in the application. |
| liquibase        | Shows any Liquibase database migrations that have been applied. Requires one or more `Liquibase` beans. |
| metrics          | Shows ‘metrics’ information for the current application.     |
| mappings         | Displays a collated list of all `@RequestMapping` paths.     |
| scheduledtasks   | Displays the scheduled tasks in your application.            |
| sessions         | Allows retrieval and deletion of user sessions from a Spring Session-backed session store. Requires a Servlet-based web application using Spring Session. |
| shutdown         | Lets the application be gracefully shutdown. Disabled by default. |
| threaddump       | Performs a thread dump.                                      |

### web

| ID         | **Description**                                              |
| ---------- | ------------------------------------------------------------ |
| heapdump   | Returns an `hprof` heap dump file.                           |
| jolokia    | Exposes JMX beans over HTTP (when Jolokia is on the classpath, not available for WebFlux). Requires a dependency on `jolokia-core`. |
| logfile    | Returns the contents of the logfile (if `logging.file.name` or `logging.file.path` properties have been set). Supports the use of the HTTP `Range` header to retrieve part of the log file’s content. |
| prometheus | Exposes metrics in a format that can be scraped by a Prometheus server. Requires a dependency on `micrometer-registry-prometheus`. |

### 暴露

​	端点可能包含一些敏感信息，需要小心谨慎暴露，下面给出默认暴露的内建端点。

| ID               | JMX  | Web  |
| :--------------- | :--- | :--- |
| auditevents      | Yes  | No   |
| beans            | Yes  | No   |
| caches           | Yes  | No   |
| conditions       | Yes  | No   |
| configprops      | Yes  | No   |
| env              | Yes  | No   |
| flyway           | Yes  | No   |
| health           | Yes  | Yes  |
| heapdump         | N/A  | No   |
| httptrace        | Yes  | No   |
| info             | Yes  | Yes  |
| integrationgraph | Yes  | No   |
| jolokia          | N/A  | No   |
| logfile          | N/A  | No   |
| loggers          | Yes  | No   |
| liquibase        | Yes  | No   |
| metrics          | Yes  | No   |
| mappings         | Yes  | No   |
| prometheus       | N/A  | No   |
| scheduledtasks   | Yes  | No   |
| sessions         | Yes  | No   |
| shutdown         | Yes  | No   |
| threaddump       | Yes  | No   |

### 详情

​	关于更多 **Actuator** 的端点信息，可以参考[官方 API 文档](<https://docs.spring.io/spring-boot/docs/2.2.6.RELEASE/actuator-api//html/>)。

## health

### 自定义 HealthIndicator

```java
@Component
public class MyHealthIndicator implements HealthIndicator {

    @Override
    public Health health() {
        // perform some specific health check
        int errorCode = check();
        if (errorCode != 0) {
            return Health.down().withDetail("Error Code", errorCode).build();
        }
        return Health.up().build();
    }

    public int check() {
        return new Random().nextInt(2);
    }
}
```

### 自定义 ReactiveHealthIndicator

```java
@Component
public class MyReactiveHealthIndicator implements ReactiveHealthIndicator {

    @Override
    public Mono<Health> health() {
        //perform some specific health check that returns a Mono<Health>
        return doHealthCheck(new Health.Builder())
                .onErrorResume(ex -> Mono.just(new Health.Builder().down(ex).build()));
    }

    private Mono<Health> doHealthCheck(Health.Builder builder) {
        return Mono.fromSupplier(() -> {
            int errorCode = check();
            if (errorCode != 0) {
                return Health.down().withDetail("Error Code", errorCode).build();
            }
            return Health.up().build();
        });
    }

    public int check() {
        return new Random().nextInt(2);
    }
}
```

## Example

[actuator](<https://github.com/guolanren/spring-boot-example/tree/master/actuator>)

## 参考

​	\[1\] [Spring Boot Actuator 官方文档](<https://docs.spring.io/spring-boot/docs/current/reference/html/production-ready-features.html>)