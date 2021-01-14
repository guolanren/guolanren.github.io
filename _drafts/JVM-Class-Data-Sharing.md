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
		RuntimeMXBean bean = ManagementFactory.getRuntimeMXBean();
		System.out.printf("it cost %d ms to start up\n", System.currentTimeMillis() - bean.getStartTime());
		SpringApplication.run(AppCDSApplication.class, args);
		new CountDownLatch(1).await();
	}

}
```

## JDK 5 (基础版本)

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

## JDK 6 (支持 server 模式)

​	在各版本的尝试中，发现从 **JDK 6** 版本开始，程序默认以 **server** 模式启动，且 **CDS** 不再以 **client** 虚拟机作为限制。

![jdk-6]()

​	值得一提的是，该版本的默认垃圾收集器是 **Parallel GC**。如果显式指定 **-Xshare:on**，虚拟机将使用 **Mark Sweep Compact GC(Serial Old)** 启动。

## JDK 8 (商业特性 AppCDS)

​	在 **JDK 8u40** 的 **[Release Notes](https://www.oracle.com/java/technologies/javase/8u40-relnotes.html)<sup>[2]</sup>** 中，介绍了 **AppCDS** 特性。它是对 **CDS** 的拓展，能够将 **ExtClassLoader(扩展类加载器)**、**AppClassLoader(应用类加载器)** 加载的类，也转储到`shared archive`中。使程序能够从 **CDS** 加载更多的类，缩短更多的启动时间。该阶段的 **AppCDS** 处于实验性阶段，暂不允许商用(那为什么要成为商业特性...)，启动时需要添加 **-XX:+UnlockCommercialFeatures** 选项(并且需要添加到商用选项之前)。

1. 导出程序锁加载的类

   ```shell
   -Xshare:off
   -XX:+UnlockCommercialFeatures
   -XX:+UseAppCDS
   -XX:DumpLoadedClassList=simple.lst
   -classpath
   E:\Java\jdk1.8.0_261\jre\lib\charsets.jar;E:\Java\jdk1.8.0_261\jre\lib\deploy.jar;E:\Java\jdk1.8.0_261\jre\lib\ext\access-bridge-64.jar;E:\Java\jdk1.8.0_261\jre\lib\ext\cldrdata.jar;E:\Java\jdk1.8.0_261\jre\lib\ext\dnsns.jar;E:\Java\jdk1.8.0_261\jre\lib\ext\jaccess.jar;E:\Java\jdk1.8.0_261\jre\lib\ext\jfxrt.jar;E:\Java\jdk1.8.0_261\jre\lib\ext\localedata.jar;E:\Java\jdk1.8.0_261\jre\lib\ext\nashorn.jar;E:\Java\jdk1.8.0_261\jre\lib\ext\sunec.jar;E:\Java\jdk1.8.0_261\jre\lib\ext\sunjce_provider.jar;E:\Java\jdk1.8.0_261\jre\lib\ext\sunmscapi.jar;E:\Java\jdk1.8.0_261\jre\lib\ext\sunpkcs11.jar;E:\Java\jdk1.8.0_261\jre\lib\ext\zipfs.jar;E:\Java\jdk1.8.0_261\jre\lib\javaws.jar;E:\Java\jdk1.8.0_261\jre\lib\jce.jar;E:\Java\jdk1.8.0_261\jre\lib\jfr.jar;E:\Java\jdk1.8.0_261\jre\lib\jfxswt.jar;E:\Java\jdk1.8.0_261\jre\lib\jsse.jar;E:\Java\jdk1.8.0_261\jre\lib\management-agent.jar;E:\Java\jdk1.8.0_261\jre\lib\plugin.jar;E:\Java\jdk1.8.0_261\jre\lib\resources.jar;E:\Java\jdk1.8.0_261\jre\lib\rt.jar;F:\IdeaProjects\cds\simple\simple-1.0-SNAPSHOT.jar
   ```

   添加上述虚拟机参数，启动示例程序，结果会生成一份`simple.lst`文件，里面的内容都是虚拟机加载的类。

2. 使用`simple.lst`转储为`shared archive`

   ```shell
   -Xshare:dump
   -XX:+UnlockCommercialFeatures
   -XX:+UseAppCDS
   -XX:SharedArchiveFile=simple.jsa
   -XX:SharedClassListFile=simple.lst
   -classpath
   E:\Java\jdk1.8.0_261\jre\lib\charsets.jar;E:\Java\jdk1.8.0_261\jre\lib\deploy.jar;E:\Java\jdk1.8.0_261\jre\lib\ext\access-bridge-64.jar;E:\Java\jdk1.8.0_261\jre\lib\ext\cldrdata.jar;E:\Java\jdk1.8.0_261\jre\lib\ext\dnsns.jar;E:\Java\jdk1.8.0_261\jre\lib\ext\jaccess.jar;E:\Java\jdk1.8.0_261\jre\lib\ext\jfxrt.jar;E:\Java\jdk1.8.0_261\jre\lib\ext\localedata.jar;E:\Java\jdk1.8.0_261\jre\lib\ext\nashorn.jar;E:\Java\jdk1.8.0_261\jre\lib\ext\sunec.jar;E:\Java\jdk1.8.0_261\jre\lib\ext\sunjce_provider.jar;E:\Java\jdk1.8.0_261\jre\lib\ext\sunmscapi.jar;E:\Java\jdk1.8.0_261\jre\lib\ext\sunpkcs11.jar;E:\Java\jdk1.8.0_261\jre\lib\ext\zipfs.jar;E:\Java\jdk1.8.0_261\jre\lib\javaws.jar;E:\Java\jdk1.8.0_261\jre\lib\jce.jar;E:\Java\jdk1.8.0_261\jre\lib\jfr.jar;E:\Java\jdk1.8.0_261\jre\lib\jfxswt.jar;E:\Java\jdk1.8.0_261\jre\lib\jsse.jar;E:\Java\jdk1.8.0_261\jre\lib\management-agent.jar;E:\Java\jdk1.8.0_261\jre\lib\plugin.jar;E:\Java\jdk1.8.0_261\jre\lib\resources.jar;E:\Java\jdk1.8.0_261\jre\lib\rt.jar;F:\IdeaProjects\cds\simple\simple-1.0-SNAPSHOT.jar
   ```

   使用上述虚拟机参数，执行转储。转储所指定的 **-classpath/-cp** 参数必须与运行时使用的相同，或者为运行时的前缀。

   > Note that the `-cp` parameter used at archive creation time must be the same as (or a prefix of) the `-cp` used at run time.<sup>[2]</sup>

3. 使用生成的转储文件，启动程序，感受 **AppCDS**

   ```shell
   -Xshare:on
   -XX:+UnlockCommercialFeatures
   -XX:+UseAppCDS
   -XX:SharedArchiveFile=simple.jsa
   -classpath
   E:\Java\jdk1.8.0_261\jre\lib\charsets.jar;E:\Java\jdk1.8.0_261\jre\lib\deploy.jar;E:\Java\jdk1.8.0_261\jre\lib\ext\access-bridge-64.jar;E:\Java\jdk1.8.0_261\jre\lib\ext\cldrdata.jar;E:\Java\jdk1.8.0_261\jre\lib\ext\dnsns.jar;E:\Java\jdk1.8.0_261\jre\lib\ext\jaccess.jar;E:\Java\jdk1.8.0_261\jre\lib\ext\jfxrt.jar;E:\Java\jdk1.8.0_261\jre\lib\ext\localedata.jar;E:\Java\jdk1.8.0_261\jre\lib\ext\nashorn.jar;E:\Java\jdk1.8.0_261\jre\lib\ext\sunec.jar;E:\Java\jdk1.8.0_261\jre\lib\ext\sunjce_provider.jar;E:\Java\jdk1.8.0_261\jre\lib\ext\sunmscapi.jar;E:\Java\jdk1.8.0_261\jre\lib\ext\sunpkcs11.jar;E:\Java\jdk1.8.0_261\jre\lib\ext\zipfs.jar;E:\Java\jdk1.8.0_261\jre\lib\javaws.jar;E:\Java\jdk1.8.0_261\jre\lib\jce.jar;E:\Java\jdk1.8.0_261\jre\lib\jfr.jar;E:\Java\jdk1.8.0_261\jre\lib\jfxswt.jar;E:\Java\jdk1.8.0_261\jre\lib\jsse.jar;E:\Java\jdk1.8.0_261\jre\lib\management-agent.jar;E:\Java\jdk1.8.0_261\jre\lib\plugin.jar;E:\Java\jdk1.8.0_261\jre\lib\resources.jar;E:\Java\jdk1.8.0_261\jre\lib\rt.jar;F:\IdeaProjects\cds\simple\simple-1.0-SNAPSHOT.jar
   -verbose:class
   ```

   

## JDK 10 (开源 AppCDS)

## 参考

​	\[1\] [](<https://book.douban.com/subject/34907497/>)

