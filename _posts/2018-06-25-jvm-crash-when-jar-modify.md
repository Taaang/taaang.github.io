---
title: Java Jar包变更导致JVM崩溃问题
date: 2018-04-02 10:23:10
categories:
- JVM
tags: JVM
---

最近部分线上JAVA项目和Tomcat出现无规律性崩溃，崩溃信息主要为：
```
异常（一）
JException in thread "data_send_thread_1" java.lang.NoClassDefFoundError: com/****/StreamResetException
        ...
        at java.lang.ClassLoader.loadClass(ClassLoader.java:424)
        at sun.misc.Launcher$AppClassLoader.loadClass(Launcher.java:331)
        at java.lang.ClassLoader.loadClass(ClassLoader.java:357)
        ... 10 more
```
或者
```
异常（二）
Java frames: (J=compiled Java code, j=interpreted, Vv=VM code)
J 128  java.util.zip.ZipFile.getEntry(J[BZ)J (0 bytes) @ 0x00007f62450db198 [0x00007f62450db140+0x58]
J 103670 C2 sun.misc.URLClassPath$JarLoader.getResource(Ljava/lang/String;Z)Lsun/misc/Resource; (85 bytes)
J 92261 C2 java.net.URLClassLoader$2.run()Ljava/lang/Object; (5 bytes) @ 0x00007f624b5aa3c0 [0x00007f624b5aa280+0x140]
v  ~StubRoutines::call_stub
J 933  java.security.AccessController.doPrivileged(Ljava/security/PrivilegedAction;Ljava/security/AccessControlContext;)Ljava/lang/Object;
J 107440 C2 java.net.URLClassLoader.findResource(Ljava/lang/String;)Ljava/net/URL; (37 bytes) @ 0x00007f624df35394 [0x00007f624df35300+0x94]
J 107175 C2 java.lang.ClassLoader.getResource(Ljava/lang/String;)Ljava/net/URL; (36 bytes) @ 0x00007f6249f14f28 [0x00007f6249f14da0+0x188]
J 107117 C2 org.apache.catalina.loader.WebappClassLoaderBase.getResourceAsStream(Ljava/lang/String;)Ljava/io/InputStream; (354 bytes)
J 107084 C2 org.apache.catalina.startup.ContextConfig.checkHandlesTypes(Lorg/apache/tomcat/util/bcel/classfile/JavaClass;)V (462 bytes)
```


可以看出，崩溃原因均可归结为Class加载失败导致的，而加载的这个Class是一个JAVA Agent中的一个类。

## 异常（一）：线程进行类加载时抛出

当对象创建时，ClassLoader会先判断对象对应的类是否已经加载过，如果没有，则会优先进行加载。但在当前场景下，ClassLoader进行类加载时，抛出了NoClassDefFoundError异常。

其中，NoClassDefFoundError与ClassNotFoundException是有区别的。ClassNotFoundException是在进行动态类加载时出现，往往实现不知道这个类是否存在，比如调用Class.forName，通过反射进行类加载时容易出现这个异常；NoClassDefFoundError遇到的并不多，一般是在编译时明确该类存在，但是在运行时进行加载的时候，找不到该类的定义。

## 异常（二）：Tomcat类加载时抛出

Tomcat作为Web应用容器，为每一个Web应用创建一个单独的WebAppClassLoader，用于加载这个应用所需要的类，同时也以此将不同项目所需要的类隔离开来。从异常信息可以看出，当ClassLoader尝试去加载一个类时，首先进行Jar包扫描，找到对应的Class在哪个Jar报中，然后通过特殊的权限控制方式，读取Jar包进行类加载。而Jar包本身以Zip格式为基础，所以通过ZipFile获取文件入口，而此时抛出异常。


## 异常分析

&emsp;1.  NoClassDefFoundError为JVM运行时异常，是在尝试加载Class时无法找到抛出的；

&emsp;2.  对Tomcat源码进行调试，发现在AppClassLoader中，包含宝对应Java Agent的JAR包路径，并在加载过程中检查该JAR包中是否包含对应的Class，相关代码如下：

```
URLClassLoader.java


protected Class<?> findClass(final String name)
    throws ClassNotFoundException
{
    final Class<?> result;
    try {
        result = AccessController.doPrivileged(
            new PrivilegedExceptionAction<Class<?>>() {
                public Class<?> run() throws ClassNotFoundException {
                    String path = name.replace('.', '/').concat(".class");
                    Resource res = ucp.getResource(path, false);
                    if (res != null) {
                        try {
                            return defineClass(name, res);
                        } catch (IOException e) {
                            throw new ClassNotFoundException(name, e);
                        }
                    } else {
                        return null;
                    }
                }
            }, acc);
    } catch (java.security.PrivilegedActionException pae) {
        throw (ClassNotFoundException) pae.getException();
    }
    if (result == null) {
        throw new ClassNotFoundException(name);
    }
    return result;
}
```

其中，`ucp.getResource(path, false)`尝试对指定类的ClassPath进行加载，遍历当前ClassLoader所包含的所有JAR包资源，利用ZipFile的getEntry方法，从jar包中搜索对应的Class信息，相关代码如下：

```
ZipFile.java


/**
 * Returns the zip file entry for the specified name, or null
 * if not found.
 *
 * @param name the name of the entry
 * @return the zip file entry, or null if not found
 * @throws IllegalStateException if the zip file has been closed
 */
public ZipEntry getEntry(String name) {
    if (name == null) {
        throw new NullPointerException("name");
    }
    long jzentry = 0;
    synchronized (this) {
        ensureOpen();
        jzentry = getEntry(jzfile, zc.getBytes(name), true);
        if (jzentry != 0) {
            ZipEntry ze = getZipEntry(name, jzentry);
            freeEntry(jzfile, jzentry);
            return ze;
        }
    }
    return null;
}
```

正常情况下在JAVA Agent的JAR包中搜索，能找到对应的Class信息，即getEntry方法返回非0；而出现异常时，getEntry方法返回0。

&emsp;3.  使用strace查看getEntry时对应的系统调用，结果如下：

__（1）Class加载成功时，可以从对应的JAR包中读取到Class信息__
![Strace_normal](https://github.com/Taaang/blog/blob/master/assets/images/post_imgs/img_jvm_crash_1.png?raw=true)

__（2）Class加载异常时，无法读取到Class信息__
![Strace_error](https://github.com/Taaang/blog/blob/master/assets/images/post_imgs/img_jvm_crash_2.png?raw=true)


可以看到在对JAR包中Class进行加载时，找不到对应的Class。但是，实际上使用解压或者使用JD-GUI查看其内容时，是能够找到编译后的Class文件，所以该类是存在的。那么问题来了，为什么明明有却会说找不到？

## 问题原因

最终，根据错误的异常信息，在Java官网找到了相似的异常，其实这是JAVA本身就存在的一个问题：

```
ID 1296729.1

Java Virtual Machine (JVM) crashes in java.util.zip.ZipFile.getEntry() during Class Loading (文档 ID 1296729.1） -  Random crashes during classloading while a jar/zip file is being accessed.  Here is a typical stack trace. Please notice that a custom classloader calls java.util.zip.ZipFile.getEntry() and the crash happens somewhere in libzip or libc or a native windows dll：

tack: [0xfffffffe4e900000,0xfffffffe4e940000], sp=0xfffffffe4e93b1a0, free space=236k
Native frames: (J=compiled Java code, j=interpreted, Vv=VM code, C=native code)
C [libc_psr.so.1+0xbf4]
C [libzip.so+0xe280]
C [libzip.so+0x28e8]
C [libzip.so+0x2d9c]
J java.util.zip.ZipFile.getEntry(JLjava/lang/String;Z)J
J java.util.zip.ZipFile.getEntry(Ljava/lang/String;)Ljava/util/zip/ZipEntry;
J weblogic.utils.classloaders.JarClassFinder.getSource(Ljava/lang/String;)Lweblogic/utils/classloaders/Source;
J weblogic.utils.classloaders.AbstractClassFinder.getClassSource(Ljava/lang/String;)Lweblogic/utils/classloaders/Source;
J weblogic.utils.classloaders.MultiClassFinder.getClassSource(Ljava/lang/String;)Lweblogic/utils/classloaders/Source;
J weblogic.utils.classloaders.MultiClassFinder.getClassSource(Ljava/lang/String;)Lweblogic/utils/classloaders/Source;
J weblogic.utils.classloaders.MultiClassFinder.getClassSource(Ljava/lang/String;)Lweblogic/utils/classloaders/Source;
j weblogic.application.utils.CompositeWebAppFinder.getClassSource(Ljava/lang/String;)Lweblogic/utils/classloaders/Source;
J weblogic.utils.classloaders.MultiClassFinder.getClassSource(Ljava/lang/String;)Lweblogic/utils/classloaders/Source;
J weblogic.utils.classloaders.MultiClassFinder.getClassSource(Ljava/lang/String;)Lweblogic/utils/classloaders/Source;
j weblogic.utils.classloaders.CodeGenClassFinder.getClassSource(Ljava/lang/String;)Lweblogic/utils/classloaders/Source;
j weblogic.utils.classloaders.GenericClassLoader.findLocalClass(Ljava/lang/String;)Ljava/lang/Class;
j weblogic.utils.classloaders.GenericClassLoader.findClass(Ljava/lang/String;)Ljava/lang/Class;
j weblogic.utils.classloaders.ChangeAwareClassLoader.findClass(Ljava/lang/String;)Ljava/lang/Class;
J java.lang.ClassLoader.loadClass(Ljava/lang/String;Z)Ljava/lang/Class;
J weblogic.utils.classloaders.ChangeAwareClassLoader.loadClass(Ljava/lang/String;)Ljava/lang/Class;
j java.lang.ClassLoader.loadClassInternal(Ljava/lang/String;)Ljava/lang/Class;
```

其中，三种情况可能导致该问题发生：

```
There are three possible scenarios here:

1. While a class is in use it is dynamically reloaded from a jar file.
2. While a jar file is being accessed by the class loader, the jar file is being modified.
3. A Jarfile which was bigger than 4GB was accessed (applies to Java 6 and earlier only)


Please note that a crash may happen even a long time after a jarfile was modified as classloaders keep references to jarfiles.

Another possible sceanrio is when Java or the application itself is being patched while the application is running.
```

而这次问题的发生是由于第二点引起的，我们在对JAVA Agent版本进行更新时，是使用覆盖源文件的方式进行，随着Tomcat或者JVM重启去重新加载新的Agent JAR包。

在更新JAR包，由于JVM还是保持着原JAR包的引用，所以再尝试从JAR包中进行Class加载时抛出异常，导致JVM崩溃。

其中有一点比较关键的是，即使在JAR包变更后很长一段时间，也会出现这个问题，原因是因为在正常情况下，业务主要流程已经都跑过一次，依赖的类已经加载过，所以很少触发新的类加载，而当应用走到某个很少触发的业务逻辑或者抛出某个未加载过的异常，需要从该变更过的JAR包中进行Class加载时，就会产生这个现象。


## 解决方案

```
1. StackOverFlow中有大佬表示可通过升级使用JAVA 9来解决，JDK9 early access builds已经解决该问题。
（https://stackoverflow.com/questions/38326183/jvm-crashed-in-java-util-zip-zipfile-getentry）
2. 启动时关闭MemoryMapping。JAVA Bug Fixs - 6929479
（http://www.oracle.com/us/technologies/java/overview-156328.html）
```
