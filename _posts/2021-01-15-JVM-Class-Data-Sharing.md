---
title: JVM-Class Data Sharing
description: 最初的 Class data sharing (CDS) 是 J2SE 5.0 新增的特性，主要是为了降低应用的启动时间，同时也能够减少内存的使用...
categories: 
 - code
tags:
 - jvm
 - jep
---

------

## 前言

​	最初的 **[Class data sharing (CDS)](https://docs.oracle.com/javase/1.5.0/docs/guide/vm/class-data-sharing.html)<sup>[1]</sup>** 是 **J2SE 5.0** 新增的特性，主要是为了降低应用的启动时间，同时也能够减少内存的使用。它通过加载系统的 **jar** 包到一个私有的内部表示，并转储为一个叫做`shared archive`的内存映射文件。开启该特性的 **jvm** 进程，会使用 **BootStrapClassLoader** 去加载`shared archive`文件中的类。多个 **jvm** 进程共享`shared archive`中的类的元数据，省去内存的额外消耗，以及免去解析这些固有的类所耗的时间。

​	它仅支持 **Java HotSpot Client VM**，垃圾收集器也仅支持 **serial garbage collector**。由于这些局限，早期并没有被广泛使用。在后续的更新迭代中，对 **CDS** 的一系列改进，才使得该特性慢慢受关注。

## 示例程序

```java
/**
 * 普通的 java 程序
 */ 
public class SimpleDemo {

    public static void main(String[] args) throws InterruptedException {
        RuntimeMXBean bean = ManagementFactory.getRuntimeMXBean();
        System.out.printf("it cost %d ms to start up\n", System.currentTimeMillis() - bean.getStartTime());
        new CountDownLatch(1).await();
    }

}

/**
 * Spring Boot 程序 —— 为了演示 AppCDS 加载更多的类
 */
@SpringBootApplication
public class AppCDSApplication {

	public static void main(String[] args) throws InterruptedException {
		SpringApplication.run(AppCDSApplication.class, args);
		RuntimeMXBean bean = ManagementFactory.getRuntimeMXBean();
		System.out.printf("it cost %d ms to start up\n", System.currentTimeMillis() - bean.getStartTime());
		new CountDownLatch(1).await();
	}

}
```

## 历程

### JDK 5 (基础版本)

​	在启用 **CDS** 之前，需要先创建`shared archive`文件。

```shell
shell> java -Xshare:dump
Loading classes to share ... done.
Rewriting and unlinking classes ... done.
Calculating hash values for String objects .. done.
Calculating fingerprints ... done.
Removing unshareable information ... done.
Moving most read-only objects to shared space at 0x2c380000 ... done.
Moving common symbols to shared space at 0x2c6e3e78 ... done.
Moving remaining symbols to shared space at 0x2c80a208 ... done.
Moving string char arrays to shared space at 0x2c80ac98 ... done.
Moving additional symbols to shared space at 0x2c88dd40 ... done.
Read-only space ends at 0x2c8e3ee8, 5652200 bytes.
Moving read-write objects to shared space at 0x2cb80000 ... done.
Moving String objects to shared space at 0x2d0f1820 ... done.
Read-write space ends at 0x2d130bf8, 5966840 bytes.
Updating references to shared objects ... done.
```

​	执行转储命令后，可以看到一些描述转储过程的日志，并在 **$JAVA_HOME/jre/bin/client** 目录下生成`classes.jsa`文件(在 **Windows** 环境 **jdk1.5** 版本下的结果。并且不同平台、不同版本都有差异，这里不细讲)。如果`classes.jsa`文件已存在，需要手动删除，否则会打印失败信息。

​	在满足条件的情况下，**CDS** 是默认开启的。可以通过以下选项，根据具体情况控制 **CDS**：

- **-Xshare:off**：关闭 **CDS**。
- **-Xshare:on**：开启 **CDS**，在条件不满足的情况下，会打印错误信息并退出。
- **-Xshare:auto**（default）：满足条件下，开启 **CDS**，否则不启用。

### JDK 6 (支持 server 模式)

​	在各版本的尝试中，发现从 **JDK 6** 版本开始，程序默认以 **server** 模式启动，且 **CDS** 不再以 **client** 虚拟机作为限制。

![jdk-6](https://github.com/guolanren/gallery/blob/master/found/2021-01-14-JVM-Class-Data-Sharing/jdk-6.png?raw=true)

​	值得一提的是，该版本的默认垃圾收集器是 **Parallel GC**。如果显式指定 **-Xshare:on**，虚拟机将使用 **Mark Sweep Compact GC(Serial Old)** 启动。

### JDK 8 (商业特性 AppCDS)

​	在 **JDK 8u40** 的 **[Release Notes](https://www.oracle.com/java/technologies/javase/8u40-relnotes.html)<sup>[2]</sup>** 中，介绍了 **AppCDS** 特性。它是对 **CDS** 的拓展，能够将 **ExtClassLoader(扩展类加载器)**、**AppClassLoader(应用类加载器)** 加载的类，也转储到`shared archive`中。使程序能够从 **CDS** 加载更多的类，缩短更多的启动时间。该阶段的 **AppCDS** 处于实验性阶段，暂不允许商用(那为什么要成为商业特性...)，启动时需要添加 **-XX:+UnlockCommercialFeatures** 选项(并且需要添加到商用选项之前)。

1. 导出程序锁加载的类

   ```shell
   -Xshare:off -XX:+UnlockCommercialFeatures -XX:+UseAppCDS -XX:DumpLoadedClassList=simple.lst -classpath ...
   ```

   添加上述虚拟机参数，启动示例程序，结果会生成一份`simple.lst`文件，里面的内容都是虚拟机加载的类。

2. 使用`simple.lst`转储为`simple.jsa`

   ```shell
   -Xshare:dump -XX:+UnlockCommercialFeatures -XX:+UseAppCDS -XX:SharedArchiveFile=simple.jsa -XX:SharedClassListFile=simple.lst -classpath ...
   ```

   使用上述虚拟机参数，执行转储。转储所指定的 **-classpath/-cp** 参数必须与运行时使用的相同，或者为运行时的前缀。

   > Note that the `-cp` parameter used at archive creation time must be the same as (or a prefix of) the `-cp` used at run time.<sup>[2]</sup>

   这时的转储过程日志与之前有所不同了。

   ```log
   Allocated shared space: 37879808 bytes at 0x0000000800000000
   Loading classes to share ...
   com/intellij/rt/execution/application/AppMainV2$1
   Preload Error: Failed to load com/intellij/rt/execution/application/AppMainV2$Agent
   Preload Error: Failed to load com/intellij/rt/execution/application/AppMainV2
   Preload Error: Failed to load com/intellij/rt/execution/application/AppMainV2$1
   Loading classes to share: done.
   Rewriting and linking classes ...
   Rewriting and linking classes: done
   Number of classes 701
       instance classes   =   687
       obj array classes  =     6
       type array classes =     8
   Calculating fingerprints ... done. 
   Removing unshareable information ... done. 
   Shared Lookup Cache Table Buckets = 8216 bytes
   Shared Lookup Cache Table Body = 120256 bytes
   ro space:   2228784 [ 37.5% of total] out of  16777216 bytes [13.3% used] at 0x0000000800000000
   rw space:   2872560 [ 48.3% of total] out of  16777216 bytes [17.1% used] at 0x0000000801000000
   md space:    811224 [ 13.6% of total] out of   4194304 bytes [19.3% used] at 0x0000000802000000
   mc space:     34053 [  0.6% of total] out of    131072 bytes [26.0% used] at 0x0000000802400000
   total   :   5946621 [100.0% of total] out of  37879808 bytes [15.7% used]
   ```

3. 使用生成的转储文件，启动程序，感受 **AppCDS**

   ```shell
   -Xshare:on -XX:+UnlockCommercialFeatures -XX:+UseAppCDS -XX:SharedArchiveFile=simple.jsa -classpath ... -verbose:class
   ```

   **-verbose:class** 选项可以观察程序类的加载情况，由图可知，**AppClassLoader** 加载的 `name.guolanren.SimpleDemo` 是从`shared archive`载入的。

   ![jdk-8-appcds](https://github.com/guolanren/gallery/blob/master/found/2021-01-14-JVM-Class-Data-Sharing/jdk-8-appcds.png?raw=true)

​	使用 **SimpleDemo** 示例程序分别运行 **CDS**、**AppCDS**。有意思的是，**AppCDS** 共享的已装入数反而比用 **CDS** 的还少。

![jdk-8-simple-cds-vs-appcds](https://github.com/guolanren/gallery/blob/master/found/2021-01-14-JVM-Class-Data-Sharing/jdk-8-simple-cds-vs-appcds.png?raw=true)

​	于是，选择运行 **SpringBoot** 程序(**AppClassLoader** 加载的类更多)来观察情况。

![jdk-8-sb-nocds-cds-appcds](https://github.com/guolanren/gallery/blob/master/found/2021-01-14-JVM-Class-Data-Sharing/jdk-8-sb-nocds-cds-appcds.png?raw=true)

![jdk-8-cost-time](https://github.com/guolanren/gallery/blob/master/found/2021-01-14-JVM-Class-Data-Sharing/jdk-8-cost-time.png?raw=true)

​	可以看出，**Metaspace** 方面，开启 **CDS** 耗费 **12.3 MB**，相比未开启的 **17.9 MB** 还是挺可观的。而开启 **AppCDS** 仅耗费 **2.4 MB**，直接下了一个量级。启动时间方面，仅开启 **CDS** 似乎已经没有多大优势了，再加上启动 **Spring** 容器所需的时间，**CDS** 耗费的时间有时甚至超过未开启 **CDS** 的情况（当然这只是因为示例程序算的启动消耗时间有问题而已...），至于开启 **AppCDS** 的启动时间则几乎减半。

### OpenJDK 10 (开源 AppCDS)

​	**[JEP 310](https://openjdk.java.net/jeps/310)<sup>[3]</sup>** 提出，要将 **AppCDS** 特性开源，在 **10** 版本发行。相比 **JDK 8** 中的 **AppCDS**，它将不再是商业特性，并根据提案描述，它还支持将自定义的类加载器所加载的类，也转储到`shared archive`中。

​	测试的版本为 **Oracle** **10.0.2**，已经是 **10** 的最新版本了。按照 **JEP 310** 的指示，这次使用 **-XX:+UseAppCDS** 时并没有指定 **-XX:+UnlockCommercialFeatures**，然而它报错了：

```log
Error: To use 'UseAppCDS', first unlock using -XX:+UnlockCommercialFeatures.
Error: The unlock option must precede 'UseAppCDS'.
Error: Could not create the Java Virtual Machine.
Error: A fatal exception has occurred. Program will exit.
```

### JDK 11 (默认开启 AppCDS)

​	显示指定 **-XX:+UseAppCDS** ，会得到 **Java HotSpot(TM) 64-Bit Server VM warning: Ignoring obsolete option UseAppCDS; AppCDS is automatically enabled** 的提示。

### JDK 12 (默认 CDS Archives)

​	**[JEP 341<sup>[4]</sup>](https://openjdk.java.net/jeps/341)** 的目标是为了缩短开箱即用的启动时间，用户也无需再使用 **-Xshare:dump** 也可从 **CDS** 中获益。安装完 **JDK 12** 后，在`$JAVA_HOME/bin/server`目录就已存在`classes.jsa`文件。用户无需指定任何虚拟机参数，都可以使用 CDS。

### JDK 13 (动态 CDS Archives)

​	...

## 转储过程

​	...

## 参考

​	\[1\] [Oracle. Class Data Sharing J2SE 5.0](<https://docs.oracle.com/javase/1.5.0/docs/guide/vm/class-data-sharing.html>)

​	\[2\] [Oracle. JDK 8u40 Release Notes](<https://www.oracle.com/java/technologies/javase/8u40-relnotes.html>)

​	\[3\] [OpenJDK. JEP 310](<https://openjdk.java.net/jeps/310>)

​	\[4\] [OpenJDK. JEP 341](<https://openjdk.java.net/jeps/341>)

​	\[5\] [OpenJDK. JEP 350](<https://openjdk.java.net/jeps/350>)

​	\[6\] [OpenJDKwiki. Caching Java Heap Objects](<https://wiki.openjdk.java.net/display/HotSpot/Caching+Java+Heap+Objects>)

​	\[7\] [OpenJDKwiki. How CDS Copies Class Metadata into the Archive ](<https://wiki.openjdk.java.net/display/HotSpot/How+CDS+Copies+Class+Metadata+into+the+Archive>)



