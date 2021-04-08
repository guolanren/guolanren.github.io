---
title: Spring-Field Injection 不推荐使用
description: 很早之前，在学习 Spring 的依赖注入、控制反转时，被告知有两种依赖注入方式：构造器注入、Setter 注入（也许你会说还有接口注入）。很快的，在 Spring 的 xml 配置文件中就配置了起来，让原本独立的 Bean 有了联系。后来的 Spring 引入了注解的方式进行配置，同时也提供了 @Autowired 支持了属性注入(Field Injection)。当然，构造器注入、Setter 注入依然能够通过 @Autowired 的方式得到支持。Field-Injection 的原理很简单，在 Spring 完成一个 Bean 的实例化后，会通过反射，对该 Bean 进行依赖注入。这种方式很简洁，不需要额外的样板代码，比如构造器注入的构造方法、Setter 注入的各属性的 setter 方法。只需一行依赖定义的代码、一个 @Autowired 注解，剩下的事情交给 Spring 容器...
categories: 
 - code
tags:
 - spring
---

------

## 前言

​	很早之前，在学习 **Spring** 的依赖注入**([DI](https://en.wikipedia.org/wiki/Dependency_injection))**<sup>[1]</sup>、控制反转**([IOC](https://en.wikipedia.org/wiki/Inversion_of_control))**<sup>[2]</sup>时，被告知有两种依赖注入方式：构造器注入、**Setter** 注入（也许你会说还有接口注入）。很快的，在 **Spring** 的 **xml** 配置文件中就配置了起来，让原本独立的 **Bean** 有了联系。后来的 **Spring** 引入了注解的方式进行配置，同时也提供了 **@Autowired** 支持了属性注入**(Field Injection)**。当然，构造器注入、**Setter** 注入依然能够通过 **@Autowired** 的方式得到支持。**Field-Injection** 的原理很简单，在 **Spring** 完成一个 **Bean** 的实例化后，会通过反射，对该 **Bean** 进行依赖注入。这种方式很简洁，不需要额外的样板代码，比如构造器注入的构造方法、**Setter** 注入的各属性的 **setter** 方法。只需一行依赖定义的代码、一个 **@Autowired** 注解，剩下的事情交给 **Spring** 容器。

​	不过最近开发中，**IDEA** 总是会给我提示：

![ide-inspection](https://github.com/guolanren/gallery/blob/master/found/2021-04-08-Spring-Field-Injection-%E4%B8%8D%E6%8E%A8%E8%8D%90%E4%BD%BF%E7%94%A8/ide-inspection.png?raw=true)

​	**Field-Injection** 不再推荐，使用构造器注入替代，并且这是 **Spring** 官方的建议。用了这么久的 **Field-Injection** ，并且学习 **Spring** 官方文档中，经常也能发现 **Field-Injection** 的样例代码。所以，去看看为什么吧~

## [官方](https://spring.io/blog/2015/11/29/how-not-to-hate-spring-in-2016)<sup>[3]</sup>

> ## Injection
>
> Always use [constructor based dependency injection](https://docs.spring.io/spring-framework/docs/current/spring-framework-reference/core.html#beans-constructor-injection) in your beans. Always use [assertions](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/util/Assert.html) for mandatory dependencies. For more background on why field based injection is evil, you can read [this article by Oliver Gierke](http://olivergierke.de/2013/11/why-field-injection-is-evil/).
>
> As always there is one exception to the rule, it’s fine to use field based injection in tests when you’re [using the `SpringJUnit4ClassRunner`](https://docs.spring.io/spring-framework/docs/current/spring-framework-reference/htmlsingle/#integration-testing).

## 依赖注入

​	首先，简单看一下支持的依赖注入方式<sup>[4]</sup>。

### Constructor-based dependency injection

```java
@Component
public class ConstructorBasedInjection {
    
    private final InjectedBean injectedBean;
    
    @Autowired
    public ConstructorBasedInjection(InjectedBean injectedBean) {
        this.injectedBean = injectedBean;
    }
    
}
```

​	可以看到构造器注入的方式中，依赖项被 final 修饰，这样做可以定义一个不可变对象**(immutable)**，一旦初始化后就不再改变。需要提供一个构造函数。

### Setter-based dependency injection

```java
@Component
public class SetterBasedInjection {
    
    private InjectedBean injectedBean;
    
    @Autowired
    public void setInjectedBean(InjectedBean injectedBean) {
        this.injectedBean = injectedBean;
    }
    
}
```

​	**Setter** 注入，需要为每一个依赖项提供对应的 **setter** 方法。

### Field-based dependency injection

```java
@Component
public class FieldBasedInjection {

    @Autowired
    private InjectedBean injectedBean;

}
```

​	最简单的方式，不需要样板式代码。

## Field-Injection 缺点

​	上面可以知道 **Field-Injection** 是最简洁的，但官方仍不推荐，需要理由。

### 空指针<sup>[5]</sup>

```java
FieldBasedInjection fieldBasedInjection = new FieldBasedInjection();
fieldBasedInjection.doSth(); // -> NullPointerException
```

​	考虑在脱离依赖注入容器的情况下，出现这样的代码。它会引发空指针异常。出现这种问题的原因，在于没有对对象的依赖做检查。如果使用的是构造器注入，那么在构造对象时，可以进行断言：

```java
@Component
public class ConstructorBasedInjection {
    
    private final InjectedBean injectedBean;
    
    @Autowired
    public ConstructorBasedInjection(InjectedBean injectedBean) {
        Assert.notNull(injectedBean, "InjectedBean must not be null");
        this.injectedBean = injectedBean;
    }
    
}
```

### 无法使用 final 修饰

​	如果想构建一个不可变 **Bean**，需要对 **field** 进行 **final** 修饰。这个特性，限制只能使用构造器注入 **Bean** 的依赖注入。

### 容易忽视单一职责原则

​	面向对象编程中的五大设计原则 [**SOLID**](https://en.wikipedia.org/wiki/SOLID_(object-oriented_design))<sup>[6]</sup> 中的 [**S**](https://en.wikipedia.org/wiki/Single_responsibility_principle)<sup>[7]</sup> 单一职责原则，表示一个类、对象只干一件事。由于 **Field-Injection**  太过简单，容易不知觉引入过多依赖，导致类、对象膨胀，不易维护。而使用构造器注入，在编写过多入数的构造器过程中，就能够发觉类设计的不合理，进而更好地遵守单一职责原则。

### 与依赖注入容器耦合

​	依赖注入的目的本身就是要解耦各个 **Bean**，将创建依赖的过程转移到依赖注入容器中。意味着，如果脱离依赖注入容器，例如在单元测试中，我们需要自己手动创建依赖项，然后使用反射进行属性设置，以完成注入。

## 我的看法

​	看了不少博客，也在 [**Stack Exchange**](https://softwareengineering.stackexchange.com/questions/300706/dependency-injection-field-injection-vs-constructor-injection)<sup>[8]</sup>看各路大神的讨论，支持使用构造器注入的人，他们提出的理由，仍然不能说服我不再使用 **Field-Injection**。所以，我决定继续使用这种简洁的方式，来完成我的依赖注入，直到后面有人出来告诉我， **Field-Injection** 有致命的缺点。

### 空指针

​	在使用 **Spring** 容器，而且是 **Spring Boot** 作为主流框架的今天，使用 **@Autowired** 进行 **Field-Injection** 依赖注入，如果找不到对应的 **Bean**，默认是会报错的。这种情况是不会存在空指针的。

```java
public @interface Autowired {

	/**
	 * Declares whether the annotated dependency is required.
	 * <p>Defaults to {@code true}.
	 */
	boolean required() default true;

}

***************************
APPLICATION FAILED TO START
***************************

Description:

Field xxx in YYY required a bean of type 'XXX' that could not be found.

The injection point has the following annotations:
	- @org.springframework.beans.factory.annotation.Autowired(required=true)

Action:

Consider defining a bean of type 'XXX' in your configuration.

```

​	脱离 **Spring** 容器，直接通过构造器 **new** 对象。有这样的场景吗？单元测试？就算是进行单元测试，也可以依赖 **Spring** 容器呀，`org.springframework.test.context.junit4.SpringRunner` 不香了吗？讨论中，[Maciej Chałapuk](https://softwareengineering.stackexchange.com/users/100148/maciej-chałapuk) 老哥的一个理由比较能说服我：

> Performance affects [maintainability](https://en.wikipedia.org/wiki/Maintainability). If each test suite must be run in some sort of dependency injection container, a test run of few thousands unit tests may last tens of minutes. Depending of size of a code base, this may be an issue.

​	确实，为单元测试创造依赖注入容器环境的成本很大，如果单元测试过多，耗时会很长。可是，这不应该往单元测试共享容器的方向去思考吗，为所有单元测试套件创建一个环境。另外，就算不使用容器，通过构造器构造完整 **Bean** 所需的代码，也不会比通过反射完成属性设置的代码少很多吧。

### final

​	有必要对依赖进行 **final** 修饰吗？至少我现在还没遇到这样的场景。

### 单一职责原则

​	这个简直是扯淡，代码简洁反而成为坏事了。没能很好的遵守设计原则，难道不是程序员的编程习惯与经验不足的原因吗？对于这样的程序员，就算构造器入参膨胀，他也不太可能会觉得这样的构造函数有问题。

### 与容器耦合

​	如果我在一些特定的场景下，无法使用 **Spring** 或者其他类似的框架，放心，我不会使用 **Field-Injection** ，因为我也无法使用 **Field-Injection** 。

## 参考

​	\[1\] [wiki. DI](https://en.wikipedia.org/wiki/Dependency_injection)

​	\[2\] [wiki. IOC](https://en.wikipedia.org/wiki/Inversion_of_control)

​	[3] [spring 官方博客. How not to hate Spring in 2016](https://spring.io/blog/2015/11/29/how-not-to-hate-spring-in-2016)

​	[4] [Marc Nuri. Field injection is not recommended - Spring IOC](https://odrotbohm.de/2013/11/why-field-injection-is-evil/)

​	[5] [Oliveer Drotbohm. Why field injection is evil](https://odrotbohm.de/2013/11/why-field-injection-is-evil/)

​	[6] [wiki. SOLID](https://en.wikipedia.org/wiki/SOLID)

​	[7] [wiki. Single-responsibility principle](https://en.wikipedia.org/wiki/Single_responsibility_principle)

​	[8] [Stack Exchange. Dependency Injection: Field Injection vs Constructor Injection](https://softwareengineering.stackexchange.com/questions/300706/dependency-injection-field-injection-vs-constructor-injection)