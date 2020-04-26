---
title: Spring-Boot-Example-MyBatis
description: Spring Boot Example MyBatis 的变更记录以及问题解决...
categories: 
 - code
tags:
 - spring boot
 - spring boot example
 - mybatis
---

## mybatis-curd

### examples

[Spring Boot Example MyBatis](<https://github.com/guolanren/spring-boot-example/tree/master/mybatis>)

### releae note

1. **h2** 切换成 **mysql** 测试事务隔离性 -- **2020.04.17**

### issue

- 初始化 sql 脚本不生效

  使用 **h2** 时`spring.datasource.schema`、`spring.datasource.data` 配置生效且正常初始化。

  切换成 **mysql**，配置生效，但无法初始化。

  配置 `spring.datasource.initialization-mode = always` 可以解决问题。

