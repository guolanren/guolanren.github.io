---
title: JVM-类加载机制
description: 许多概念理解起来有些费劲，大篇幅原文，暂时是一篇读书笔记...
categories: 
 - code
tags:
 - jvm
---

------

## 前言

​	许多概念理解起来有些费劲，大篇幅原文，暂时是一篇读书笔记...

## 类加载

​	类加载按照以下阶段顺序开始，解析阶段除外。在某些情况下，解析可以在初始化之后开始，这是为了支持 **Java** 的动态绑定。阶段虽然是按顺续开始的，但通常是相互交叉地混合进行。阶段可以在执行过程中，调用激活另一个阶段。

### 加载

1. 通过一个类的全限定类名来获取定义此类的二进制字节流。
2. 将这个字节流所代表的静态存储结构转化为方法去的运行时数据结构。
3. 在内存生成一个代表这个类的 **java.lang.Class** 对象，作为方法区这个类的各种数据的访问入口。

### 连接

#### 验证

​	验证阶段的目的是确保 **Class** 文件的字节流中包含的信息符合 **《Java虚拟机规范》**的全部约束要求，保证这些信息不会危害虚拟机自身的安全。验证阶段对于虚拟机的类加载机制来说，是一个非常重要的、但却不是必须要执行的阶段，因为验证阶段只有通过或者不通过的差别，只要通过了验证，其后就对程序运行期没有任何影响了。如果程序运行的全部代码（包括自己编写的、第三方包中的、从外部加载的、动态生成的等所有代码）都已经被反复使用和验证过，在生产环境的实施阶段就可以考虑使用 `-Xverify：none` 参数来关闭大部分的类验证措施，以缩短虚拟机类加载的时间。

1. 文件格式验证

   验证字节流是否符合 **Class** 文件的格式规范，且能被当前版本的虚拟机处理。

   - 是否以魔数 **0xCAFEBABE** 开头
   - 主、次版本在当前虚拟机接受范围内
   - 常量池的常量中是否有不被支持的常量类型(检查常量 **tag** 标志)
   - 指向常量的各种索引值中是否有指向不存在的常量或不符合类型的常量
   - **CONSTANT_Utf8_info **型的常量中是否有不符合 **UTF-8** 编码的数据
   - **Class** 文件中各个部分及文件本身是否有被删除的或附加的其他信息
   - ...

2. 元数据验证

   对字节码描述的信息进行语义分析。

   - 是否有父类(除了 **Object** 外，都应该有父类)
   - 该类的父类是否 **final** 修饰
   - 如果这个类不是抽象类，其是否实现了父类和接口中要求实现的所有方法
   - 类的字段、方法是否与父类产生矛盾(例如覆盖了父类的 **final** 字段，或者不符合规范的方法重载)
   - ...

3. 字节码验证

   对类的方法体进行校验分析，通过数据流分析和控制流分析，确定程序语义是合法的、符合逻辑的。

   - 保证任意时刻操作数栈的数据类型与指令代码序列都能配合工作
   - 保证任何跳转指令都不会跳转到方法体意外的字节码指令上
   - 保证方法体中的类型转换总是有效
   - ...

4. 符号引用验证

   最后一个阶段的校验行为发生在虚拟机将符号引用转化为直接引用的时候，这个转化动作将在连接的第三阶段——解析阶段中发送。符号引用验证的主要目的是确保解析行为能正常执行，如果无法通过符号引用验证，**Java** 虚拟机将会抛出一个 **java.lang.IncompatibleClassChangeError** 的子类异常，典型的如：**java.lang.IllegalAccessError**、**java.lang.NoSuchFieldError**、**java.lang.NoSuchMethodError** 等。

   - 符号引用中通过字符串描述的全限定名是否能找到对应的类
   - 在指定类中是否存在符合方法的字段描述符及简单名称锁描述的方法和字段
   - 符号引用中的类、字段、方法的可访问性(**private**、**protected**、**public**、**\<package\>**)是否可被当前类访问

#### 准备

​	准备阶段是正式为类中定义的变量（即静态变量，被 **static** 修饰的变量）分配内存并设置类变量初始值的阶段，从概念上讲，这些变量所使用的内存都应当在方法区中进行分配，但必须注意到方法区本身是一个逻辑上的区域，在 JDK 7 及之前，**HotSpot** 使用永久代来实现方法区时，实现是完全符合这种逻辑概念的；而在 JDK 8 及之后，类变量则会随着 **Class** 对象一起存放在 **Java** 堆中，这时候“类变量在方法区”就完全是一种对逻辑概念的表述了。

#### 解析

​	解析阶段是Java虚拟机将常量池内的符号引用替换为直接引用的过程...

1. 类或接口的解析
2. 字段解析
3. 方法解析
4. 接口方法解析

### 初始化

​	在准备阶段，类变量被初始化为零值，而在初始化阶段，则会根据程序去初始化类变量。初始化阶段，其实就是执行 **\<clinit\>()** 方法的过程，其是由 **javac** 根据类中定义的类变量的赋值动作，以及静态代码块，按源文件中的编写顺序，自动生成的。

#### 主动引用

​	**《Java 虚拟机规范》**严格规定有且只有 **6** 种情况必须立即对类进行"初始化"，这些行为被称为对一个类型的**"主动引用"**。

1. 遇到 **new**、**getstatic**、**putstatic**、**invokestatic** 指令时。这 **4** 条指令典型 **Java** 代码场景有：
   - **new**：使用 **new** 实例化对象。
   - **getstatic**、**putstatic**：读取或设置一个 **static** 变量时(**final** 修饰的常量不在范围内，它在编译器就被放入到常量池中)。
   - **invokestatic**：调用某个类的 **static** 方法。
2. 使用 **java.lang.reflect** 包下的方法对类进行发射调用时。
3. 初始化某个类，其父类还未初始化时，会先触发父类初始化。
4. 虚拟机启动时，用户指定的主类(包含 **main()** 方法的类)，会先被初始化。
5. 当使用 JDK 7 新加入的动态语言支持时，如果一个 **java.lang.invoke.MethodHandle** 实例最后的解析结果为 **REF_getStatic**、**REF_putStatic**、**REF_invokeStatic**、**REF_newInvokeSpecial** 四种类型的方法句柄，且这个方法句柄对应的类未初始化。
6. 当接口中定义了 JDK 8 新加入的默认方法，其实现类在初始化之前，必须先初始化该接口。

#### 被动引用

- 通过子类引用父类的静态字段，不会导致子类初始化(通过 `-XX: +TraceClassLoading` 观察，该操作还是会导致 **SubClass** 加载)

  ```java
  /**
   * output:
   * SuperClass init!
   * 123
   */
  public class NotInitialization {
      public static void main(String[] args) {
          System.out.println(SubClass.value);
      }
  }
  class SuperClass {
      static {
          System.out.println("SuperClass init!");
      }
      public static int value = 123;
  }
  class SubClass extends SuperClass {
      static {
          System.out.println("SubClass init!");
      }
  }
  ```

- 通过数组定义来引用类，不会触发此类的初始化。且触发一个名为 **[Lname.guolanren.SuperClass** 初始化。它由虚拟机自动生成、直接继承于 **java.lang.Object**。创建动作由字节码指令 **newarray** 触发。

  ```java
  /**
   * output:
   */
  public class NotInitialization {
      public static void main(String[] args) {
          SuperClass[] sca = new SuperClass[10];
      }
  }
  ```

- 常量在编译阶段会存入调用类的常量池中，本质上没有直接引用到定义常量的类，因此不会触发定义常量的类的初始化。

  ```java
  /**
   * output:
   * Hello World!
   */
  public class NotInitialization {
      public static void main(String[] args) {
          System.out.println(ConstClass.HELLO_WORLD);
      }
  }
  class ConstClass {
      static {
          System.out.println("ConstClass init!");
      }
      public static final String HELLO_WORLD = "Hello World!";
  }
  ```

#### 接口初始化

​	接口的初始化与类的初始化类似，区别于 **6** 种**"有且仅有"**中的第 **3** 种：当一个接口在初始化时，不要求其父接口全部都完成初始化。

## 类加载器

### ClassLoader

- `BootStrapClassLoader`
- `ExtClassLoader`
- `AppClassLoader`

### 双亲委派机制

### 打破双亲委派机制

## 参考

​	\[1\] [周志明. 深入理解Java虚拟机：JVM高级特性与最佳实践（第3版）.  第7章 虚拟机类加载机制](<https://book.douban.com/subject/34907497/>)
