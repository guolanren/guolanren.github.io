---
title: Spring-Boot-Example-FastJson
description: Spring Boot Example FastJson 的变更记录以及问题解决...
categories: 
 - code
tags:
 - spring boot
 - spring boot example
 - fastjson
---

## FastJson

### examples

[Spring Boot Example FastJson](<https://github.com/guolanren/spring-boot-example/tree/master/fastjson>)

### releae note

### issue

- **String** 接收 **RequestBody** 参数，字段乱序

  使用 **FastJsonHttpMessageConverter** 作为 **HttpMessageConverter** 时，**Controller** 以 **String** 接收 **Request Body**。由于 **ContentType** 为 **application/json;charset=UTF-8**，消息转换器先反序列化了 **Body**，传给 **Controller** 参数时则再次序列化，由于 **fastjson** 的默认策略，导致 **json** 里的字段乱序。

  在定义 **FastJsonHttpMessageConverter** 时，设置一下 **Features** 即可解决问题：

  ```java
  // 按原始的顺序排序
  fastJsonConfig.setFeatures(Feature.OrderedField);
  ```