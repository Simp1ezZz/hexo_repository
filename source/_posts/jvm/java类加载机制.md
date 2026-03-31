---
title: 类加载机制：双亲委派模型与自定义类加载器
date: 2026-04-03 10:00:00
tags:
  - Java进阶
  - JVM
categories: 学习
keywords: JVM,类加载,双亲委派,ClassLoader
description: 深入理解Java类加载机制与双亲委派模型原理
cover:
---

## 前言

当我们 `new Object()` 时，Object.class 是如何被加载到JVM的？类加载器如何保证类的唯一性？自定义类加载器有什么作用？本文深入解析类加载机制。

## 类加载的五个阶段

类加载过程分为五个阶段：**加载 → 验证 → 准备 → 解析 → 初始化**

### 1. 加载（Loading）

**任务：**
- 通过类的全限定名获取类的二进制字节流
- 将字节流转化为方法区的运行时数据结构
- 在堆中生成对应的Class对象作为访问入口

**类加载来源：**
- Class文件（本地磁盘）
- JAR/WAR包
- 网络获取（Applet）
- 动态代理生成（ProxyGenerator）
- 其他文件（如.jsp转换的类）

### 2. 验证（Verification）

**目的：** 确保Class文件字节流符合JVM规范，不会危害JVM安全

**阶段：**

| 阶段 | 验证内容 |
|------|---------|
| 文件格式验证 | 魔数(0xCAFEBABE)、版本号、主次版本号 |
| 元数据验证 | 语法语义分析（是否有父类、是否继承final类等） |
| 字节码验证 | 数据流、控制流分析（方法调用是否合法） |
| 符号引用验证 | 常量池中引用是否存在（类、方法、字段是否存在） |

### 3. 准备（Preparation）

**任务：** 为类变量（static）分配内存并设置初始值

```java
public class PrepareDemo {
    // 准备阶段：value=0（int默认零值）
    // 初始化阶段：value=100
    public static int value = 100;

    // 常量：准备阶段直接赋值为200（编译期确定）
    public static final int CONST = 200;

    // 引用类型默认null
    public static String str;
}
```

**注意：** static final编译时常量在准备阶段就赋值，普通static变量在初始化阶段才赋值为指定值。

### 4. 解析（Resolution）

**任务：** 将符号引用替换为直接引用

**符号引用：** 用字符串表示的类/方法/字段引用，不定位实际内存地址
**直接引用：** 实际的内存地址/偏移量/能定位到目标对象的指针

**解析内容：**
- 类/接口解析
- 字段解析
- 类方法解析
- 接口方法解析

**解析时机：** 解析可能在初始化之后（动态绑定，如多态）

### 5. 初始化（Initialization）

**任务：** 执行类构造器 `<clinit>()`，真正赋值为初始值

**`<clinit>() vs <init>()`
:**
- `<clinit>()`：类初始化器，JVM自动生成，收集所有static变量赋值和static{}语句
- `<init>()`：实例构造函数，就是我们写的构造方法

**触发初始化的时机（主动引用）：**
```java
public class InitDemo {
    static {
        System.out.println("InitDemo类初始化");
    }

    public static void main(String[] args) {
        // 触发初始化的几种情况：

        // 1. new对象
        new InitDemo();

        // 2. 读取/设置静态字段（final除外）
        System.out.println(InitDemo.value);  // 触发
        // InitDemo.CONST 不会触发（编译时常量）

        // 3. 调用静态方法
        InitDemo.test();

        // 4. 反射
        Class.forName("InitDemo");

        // 5. 初始化子类时父类先初始化
        // new SubInitDemo(); // 父类先初始化

        // 6. JVM启动的启动类（main所在类）
    }
}
```

---

## 双亲委派模型（Parents Delegation Model）

### 核心原理

当类加载器收到加载请求时，首先委派给父类加载器处理，父类再委派其父类，直到Bootstrap ClassLoader。只有父加载器无法完成时，才自己尝试加载。

### 类加载器层次

```
┌─────────────────────────────────────────────────────────────┐
│           Bootstrap ClassLoader (启动类加载器)                │
│   C++实现，加载 JAVA_HOME/jre/lib/rt.jar 中的核心类          │
│   以及 -Xbootclasspath 参数指定的类                          │
├─────────────────────────────────────────────────────────────┤
│           Extension ClassLoader (扩展类加载器)                  │
│   Java实现，加载 JAVA_HOME/jre/lib/ext/*.jar                 │
│   以及 java.ext.dirs 系统属性指定的类                         │
├─────────────────────────────────────────────────────────────┤
│           Application ClassLoader (应用类加载器)              │
│   Java实现，加载 classpath (-cp) 中的用户类                   │
├─────────────────────────────────────────────────────────────┤
│           自定义ClassLoader                                   │
│   用户自定义的类加载器，继承ClassLoader                       │
│   可打破双亲委派（如Tomcat、OSGI）                           │
└─────────────────────────────────────────────────────────────┘
```

### 为什么需要双亲委派？

**1. 防止核心类被篡改**
```java
// 用户尝试自定义java.lang.String
package java.lang;
public class String {
    public static void main(String[] args) {
        System.out.println("我的String");
    }
}
```
> 由于双亲委派，java.lang.String由Bootstrap加载器加载，用户自定义的String永远不会被加载。即使写同样的包名和类名，也无法替换核心类。这保证了Java核心API的安全性。

**2. 保证类的唯一性**
> 父加载器加载的类对子加载器可见，子加载器加载的类对父加载器不可见。因此用户编写的Object、String等类永远无法覆盖JDK核心类。

### 双亲委派实现

```java
protected Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException {
    synchronized (getClassLoadingLock(name)) {
        // 1. 检查类是否已加载（缓存中）
        Class<?> c = findLoadedClass(name);
        if (c != null) {
            return c;
        }

        // 2. 父加载器优先加载
        try {
            if (parent != null) {
                c = parent.loadClass(name, false);
            } else {
                // 3. 父为null，尝试Bootstrap加载器
                c = findBootstrapClassOrNull(name);
            }
        } catch (ClassNotFoundException e) {
            // 父加载器找不到，可能抛异常
        }

        // 4. 父加载不了，自己加载
        if (c == null) {
            c = findClass(name);
        }

        // 5. 解析（可选）
        if (resolve) {
            resolveClass(c);
        }
        return c;
    }
}
```

### 打破双亲委派的场景

某些场景下需要子加载器先尝试加载：

| 场景 | 原因 |
|------|------|
| Tomcat | 多个Web应用可能使用不同版本的Spring，需要隔离 |
| OSGI | 模块化架构，每个模块有自己的类加载器 |
| JDBC SPI | DriverManager由Bootstrap加载，但驱动实现类由AppClassLoader加载 |
| 热部署 | 同一类需要重新加载而不重启JVM |

---

## 自定义类加载器

### 适用场景

- **热部署**：应用运行期间动态加载新版本类（如Tomcat）
- **字节码加密/解密**：保护源代码，运行时解密加载
- **动态类增强**：AOP埋点、字节码插桩
- **隔离不同版本库**：同一JVM中加载不同版本的同名类（如数据库驱动）

### 实现步骤

1. 继承 `java.lang.ClassLoader`
2. 重写 `findClass()` 方法
3. 调用 `defineClass()` 将字节数组转为Class对象

```java
public class MyClassLoader extends ClassLoader {

    private String classPath;

    public MyClassLoader(String classPath) {
        this.classPath = classPath;
    }

    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        byte[] classData = loadClassData(name);
        if (classData == null) {
            throw new ClassNotFoundException(name);
        }
        return defineClass(name, classData, 0, classData.length);
    }

    private byte[] loadClassData(String name) {
        String fileName = classPath + File.separator
            + name.replace('.', File.separatorChar) + ".class";
        try (InputStream is = new FileInputStream(fileName);
             ByteArrayOutputStream bos = new ByteArrayOutputStream()) {
            byte[] buffer = new byte[1024];
            int len;
            while ((len = is.read(buffer)) != -1) {
                bos.write(buffer, 0, len);
            }
            return bos.toByteArray();
        } catch (IOException e) {
            return null;
        }
    }
}
```

### 使用自定义类加载器

```java
public class ClassLoaderTest {
    public static void main(String[] args) throws Exception {
        // 创建自定义类加载器
        MyClassLoader loader = new MyClassLoader("/Users/test/classes");

        // 加载类
        Class<?> clazz = loader.loadClass("com.example.Test");

        // 创建实例
        Object obj = clazz.newInstance();
        System.out.println("类加载器: " + clazz.getClassLoader());
    }
}
```

---

## SPI机制与线程上下文类加载器

### JDBC驱动加载（经典案例）

```java
// java.sql.DriverManager由Bootstrap ClassLoader加载
// 但数据库驱动（如com.mysql.jdbc.Driver）由AppClassLoader加载
// 这是因为DriverManager在加载时使用了线程上下文类加载器

public class DriverManager {
    static {
        // 使用线程上下文类加载器加载SPI实现
        ClassLoader cl = Thread.currentThread().getContextClassLoader();
        if (cl == null) {
            cl = DriverManager.class.getClassLoader();
        }
        // 加载 jdbc driver 实现类
        ServiceLoader<Driver> loadedDrivers = ServiceLoader.load(Driver.class, cl);
    }
}
```

### 线程上下文类加载器

每个线程都有一个类加载器属性，可以通过 `Thread.currentThread().setContextClassLoader()` 设置。

**SPI机制原理：**
1. Bootstrap类加载器加载核心SPI类（如DriverManager）
2. SPI类使用线程上下文类加载器（通常是AppClassLoader）加载具体实现
3. 这样即使核心类由Bootstrap加载，具体实现仍由用户类加载器加载

**这实际上是对双亲委派的"妥协"，解决了核心类需要加载用户实现的问题。**

---

## 常见问题与解答

### Tomcat类加载器结构

```
Bootstrap ClassLoader
    ↓
Extension ClassLoader
    ↓
Application ClassLoader
    ↓
Common ClassLoader (Tomcat共享库)
    ↓
Catalina ClassLoader (Tomcat容器)
    ↓
WebApp ClassLoader (每个Web应用独立)
    ↓
JSP ClassLoader (每个JSP页面独立)
```

**Tomcat打破双亲委派的方式：**
- WebAppClassLoader先尝试自己加载，失败后再委托父类
- 这样每个Web应用可以有自己的类版本

### 类加载器与命名空间

每个类加载器有自己的命名空间，存储它加载的类。
- 相同类被不同类加载器加载，产生不同的Class对象
- 相同类加载器只能加载一次同名类

---

## 面试高频问题

**Q1：类加载过程？**

> 答：加载→验证→准备→解析→初始化。加载获取字节流，验证确保格式安全，准备分配内存设初始值，解析将符号引用转直接引用，初始化执行\<clinit\>。

**Q2：双亲委派模型好处？**

> 答：保证类的唯一性和安全性。例如用户自定义的java.lang.String不会被加载，防止核心API被篡改。父加载器加载的类对子加载器可见，反之不可见。

**Q3：如何打破双亲委派？**

> 答：重写loadClass()方法，不优先调用父加载器。Tomcat、OSGI、JDBC SPI都打破了双亲委派。Tomcat每个Web应用有自己的类加载器，先加载本地再委托父类。

**Q4：为什么需要线程上下文类加载器？**

> 答：SPI机制中，核心类由Bootstrap加载，但具体实现由用户提供。通过线程上下文类加载器，可以让核心SPI类加载用户实现类，如JDBC驱动加载。

**Q5：ClassLoader与Class的区别？**

> 答：ClassLoader负责加载Class文件，生成Class对象。Class对象是JVM中的类元数据镜像，包含类的完整信息。同一个类多次加载可能产生不同的Class对象（不同类加载器）。

---

## 总结

类加载机制是JVM的核心能力：
- **五个阶段**：加载→验证→准备→解析→初始化，各司其职
- **双亲委派**：保证类唯一性和安全性，防止核心API被篡改
- **打破场景**：Tomcat热部署、OSGI、JDBC SPI都需要打破
- **自定义加载器**：继承ClassLoader，重写findClass()，调用defineClass()
- **SPI机制**：通过线程上下文类加载器解决核心类加载用户实现的问题
