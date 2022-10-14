> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7153063640176803853)

持续创作，加速成长！这是我参与「掘金日新计划 · 10 月更文挑战」的第 11 天，[点击查看活动详情](https://juejin.cn/post/7147654075599978532 "https://juejin.cn/post/7147654075599978532")

类加载器
====

概述
--

> 类加载器负责读取 Java 字节代码，并转换成 java.lang.Class 类的一个实例的代码模块。

> 类加载器除了用于加载类外，还可用于确定类在 Java 虚拟机中的唯一性。

> 任意一个类，都由加载它的类加载器和这个类本身一同确定其在 Java 虚拟机中的唯一性，每一个类加载器，都有一个独立的类名称空间，而不同类加载器中是允许同名 (指全限定名相同) 类存在的。

> 比较两个类是否 “相等”，前提是这两个类由同一个类加载器加载，否则，即使这两个类来源于同一个 Class 文件，被同一个虚拟机加载，只要加载它们的类加载器不同，那么这两个类就必定不相等。

> 这里 “相等” 是指：类的 Class 对象的 equals()方法、isInstance()方法的返回结果，使用 instanceof 关键字做对象所属关系判定等情况。

加载器的种类
------

1. 启动类加载器：Bootstrap ClassLoader

> 最顶层的加载类，由 C++ 实现，负责加载`%JAVA_HOME%/lib`目录下的 jar 包和类或者被 `-Xbootclasspath`参数指定的路径中的所有类。

2. 拓展类加载器：Extension ClassLoader

> 负责加载 java 平台中扩展功能的一些 jar 包，如加载`%JRE_HOME%/lib/ext`目录下的 jar 包和类，或`-Djava.ext.dirs`所指定的路径下的 jar 包。

3. 系统类加载器 / 应用程序加载器：App ClassLoader

> 负责加载当前应用 classpath 中指定的 jar 包及`-Djava.class.path`所指定目录下的类和 jar 包。开发者可以直接使用这个类加载器，如果应用程序中没有自定义过自己的类加载器，一般情况下这个就是程序中默认的类加载器。

4. 自定义类加载器：Custom ClassLoader

> 通过`java.lang.ClassLoader`的子类自定义加载 class，属于应用程序根据自身需要自定义的 ClassLoader，如 tomcat、jboss 都会根据 j2ee 规范自行实现 ClassLoader

验证不同加载器
-------

每个类加载都有一个父类加载器，可以通过程序来验证

```
public static void main(String[] args) {
        // App ClassLoader
        System.out.println(new User().getClass().getClassLoader());
        // Ext ClassLoader
        System.out.println(new User().getClass().getClassLoader().getParent());
        // Bootstrap ClassLoader
        System.out.println(new User().getClass().getClassLoader().getParent().getParent());
        // Bootstrap ClassLoader
        System.out.println(new String().getClass().getClassLoader());
    }
复制代码
```

AppClassLoader 的父类加载器为 ExtClassLoader， ExtClassLoader 的父类加载器为 null，null 并不代表 ExtClassLoader 没有父类加载器，而是 BootstrapClassLoader 。

```
sun.misc.Launcher$AppClassLoader@18b4aac2
sun.misc.Launcher$ExtClassLoader@5fdef03a
null
null
复制代码
```

核心方法
----

查看类`ClassLoader`的 loadClass 方法

```
protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException
    {
        synchronized (getClassLoadingLock(name)) {
            // First, check if the class has already been loaded
            // 检查类是否已经加载
            Class<?> c = findLoadedClass(name);
            if (c == null) {
                long t0 = System.nanoTime();
                try {
                	// 父加载器不为空，调用父加载器loadClass()方法处理
                    if (parent != null) {
                    	// 让上一层加载器进行加载
                        c = parent.loadClass(name, false);
                    } else {
                    	// 父加载器为空，使用启动类加载器 BootstrapClassLoader 加载
                        c = findBootstrapClassOrNull(name);
                    }
                } catch (ClassNotFoundException e) {
               		 // 抛出异常说明父类加载器无法完成加载请求
                    // ClassNotFoundException thrown if class not found
                    // from the non-null parent class loader
                }

                if (c == null) {
                    // If still not found, then invoke findClass in order
                    // to find the class.
                    long t1 = System.nanoTime();
                    // 调用此类加载器所实现的findClass方法进行加载
                    c = findClass(name);

                    // this is the defining class loader; record the stats
                    sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                    sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                    sun.misc.PerfCounter.getFindClasses().increment();
                }
            }
            if (resolve) {
            	// resolveClass方法是当字节码加载到内存后进行链接操作，对文件格式和字节码验证，并为 static 字段分配空间并初始化，符号引用转为直接引用，访问控制，方法覆盖等
                resolveClass(c);
            }
            return c;
        }
    }
复制代码
```

JVM 类加载机制的三种方式
==============

全盘负责
----

> 当一个类加载器负责加载某个 Class 时，该 Class 所依赖的和引用的其他 Class 也将由该类加载器负责载入，除非显示使用另外一个类加载器来载入

注意：

> 系统类加载器 AppClassLoader 加载入口类（含有 main 方法的类）时，会把 main 方法所依赖的类及引用的类也载入。只是调用了`ClassLoader.loadClass(name)`方法，并没有真正定义类。真正加载 class 字节码文件生成 Class 对象由双亲委派机制完成。

父类委托、双亲委派
---------

> 父类委托即双亲委派，双亲委派模型是描述类加载器之间的层次关系。它要求除了顶层的启动类加载器外，其余的类加载器都应当有自己的父类加载器。父子关系一般不会以继承的关系实现，而是以组合关系来复用父加载器的代码。

> 双亲委派模型是指：子类加载器如果没有加载过该目标类，就先委托父类加载器加载该目标类，只有在父类加载器找不到字节码文件的情况下才从自己的类路径中查找并装载目标类。

**双亲委派模型的好处**

> 保证 Java 程序的稳定运行，避免类的重复加载：JVM 区分不同类的方式不仅仅根据类名，相同的类文件被不同的类加载器加载产生的是两个不同的类

> 保证 Java 核心 API 不被篡改：如果没有使用双亲委派模型，而是每个类加载器加载自己的话就会出现一些问题，如编写一个称为 java.lang.Object 类，程序运行时，系统就会出现多个不同的 Object 类。反之使用双亲委派模型：无论使用哪个类加载器加载，最终都会委派给最顶端的启动类加载器加载，从而使得不同加载器加载的 Object 类都是同一个。

双亲委派机制加载 Class 的具体过程：

```
1. ClassLoader先判断该Class是否已加载，如果已加载，则返回Class对象，如果没有则委托给父类加载器

2. 父类加载器判断是否加载过该Class，如果已加载，则返回Class对象，如果没有则委托给祖父类加载器

3. 依此类推，直到始祖类加载器（引用类加载器）

4. 始祖类加载器判断是否加载过该Class，如果已加载，则返回Class对象
   如果没有则尝试从其对应的类路径下寻找class字节码文件并载入
   如果载入成功，则返回Class对象；如果载入失败，则委托给始祖类加载器的子类加载器

5. 始祖类加载器的子类加载器尝试从其对应的类路径下寻找class字节码文件并载入
   如果载入成功，则返回Class对象；如果载入失败，则委托给始祖类加载器的孙类加载器
   
6. 依此类推，直到源ClassLoader

7. 源ClassLoader尝试从其对应的类路径下寻找class字节码文件并载入
   如果载入成功，则返回Class对象；如果载入失败，源ClassLoader不会再委托其子类加载器，而是抛出异常
复制代码
```

注意：

> 双亲委派机制是 Java 推荐的机制，并不是强制的机制。可以继承 java.lang.ClassLoader 类，实现自己的类加载器。如果想保持双亲委派模型，应该重写`findClass(name)`方法；如果想破坏双亲委派模型，可以重写`loadClass(name)`方法。

缓存机制
----

> 缓存机制将会保证所有加载过的 Class 都将在内存中缓存，当程序中需要使用某个 Class 时，类加载器先从内存的缓存区寻找该 Class，只有缓存区不存在，系统才会读取该类对应的二进制数据，并将其转换成 Class 对象，存入缓存区。

> 对于一个类加载器实例来说，相同全名的类只加载一次，即 loadClass 方法不会被重复调用。因此，这就是为什么修改 Class 后，必须重启 JVM，程序的修改才会生效的原因。

> JDK8 使用的是直接内存，所以会用到直接内存进行缓存。因此，类变量为什么只会被初始化一次的原因。

打破双亲委派
======

> 在加载类的时候，会一级一级向上委托，判断是否已经加载，从自定义类加载器 --> 应用类加载器 --> 扩展类加载器 --> 启动类加载器，如果到最后都没有加载这个类，则回去加载自己的类。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1cac4de8aca2495da229fc76ab3d1a85~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

双亲委派模型并不是强制模型，而且会带来一些些的问题。例如：`java.sql.Driver`类，JDK 只能提供一个规范接口，而不能提供实现。提供实现的是实际的数据库提供商，提供商的库不可能放 JDK 目录里。

重写 loadclass 方法
---------------

自定义类加载，重写 loadclass 方法，即可破坏双亲委派机制

> 因为双亲委派的机制都是通过这个方法实现的，这个方法可以指定类通过什么类加载器来进行加载，所有如果改写加载规则，相当于打破双亲委派机制

```
import cn.ybzy.demo.Test;

import java.io.*;

public class MyClassLoader extends ClassLoader {

    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        byte[] classData;
        try {
            classData = loadClassData(name);
        } catch (IOException e) {
            throw new RuntimeException(e);
        }
        if (classData == null) {
            throw new ClassNotFoundException();
        } else {
            return defineClass(name, classData, 0, classData.length);
        }
    }

    private byte[] loadClassData(String className) throws IOException {
        String replace = className.replace('.', File.separatorChar);
        String path = ClassLoader.getSystemResource("").getPath() + replace + ".class";
        InputStream inputStream = null;
        ByteArrayOutputStream byteArrayOutputStream = null;
        try {
            inputStream = new FileInputStream(path);
            byteArrayOutputStream = new ByteArrayOutputStream();
            int bufferSize = 1024;
            byte[] buffer = new byte[bufferSize];
            int length = 0;
            while ((length = inputStream.read(buffer)) != -1) {
                byteArrayOutputStream.write(buffer, 0, length);
            }
            return byteArrayOutputStream.toByteArray();
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            if (byteArrayOutputStream != null) {
                byteArrayOutputStream.close();
            }
            if (inputStream != null) {
                inputStream.close();
            }
        }

        return null;
    }

    @Override
    protected Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException {
        synchronized (getClassLoadingLock(name)) {
            // First, check if the class has already been loaded
            Class<?> c = findLoadedClass(name);
            if (c == null) {
                long t0 = System.nanoTime();
                try {
                    // 修改classloader的原双亲委派逻辑，从而打破双亲委派
                    if (name.startsWith("cn.ybzy.demo")) {
                        c = findClass(name);
                    } else {
                        c = this.getParent().loadClass(name);
                    }
                } catch (ClassNotFoundException e) {
                    // ClassNotFoundException thrown if class not found
                    // from the non-null parent class loader
                }

                if (c == null) {
                    // If still not found, then invoke findClass in order
                    // to find the class.
                    long t1 = System.nanoTime();
                    c = findClass(name);

                    // this is the defining class loader; record the stats
                    sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                    sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                    sun.misc.PerfCounter.getFindClasses().increment();
                }
            }
            if (resolve) {
                resolveClass(c);
            }
            return c;
        }
    }
}
复制代码
```

```
public static void main(String[] args) throws ClassNotFoundException {
        MyClassLoader classLoader = new MyClassLoader();
        Class<?> aClass = classLoader.loadClass(Test.class.getName());
        System.out.println(aClass.getClassLoader());
    }
复制代码
```

```
cn.ybzy.demo.MyClassLoader@2f410acf
复制代码
```

自定义类加载器
=======

> 自定义类加载器的核心在于对字节码文件的获取，如果是加密的字节码则需要在类中对文件进行解密。

准备字节码文件
-------

创建 Test 类，同时进行`javac Test.class`编译成字节码文件，放到目录下：`D:\Temp\cn\ybzy\demo`

```
package cn.ybzy.demo;

public class Test {
    public static void main(String[] args) {
        System.out.println("Test...");
    }
}
复制代码
```

创建自定义类加载器
---------

```
import java.io.*;

public class MyClassLoader extends ClassLoader {
    private String root;

    protected Class<?> findClass(String name) throws ClassNotFoundException {
        byte[] classData;
        try {
            classData = loadClassData(name);
        } catch (IOException e) {
            throw new RuntimeException(e);
        }
        if (classData == null) {
            throw new ClassNotFoundException();
        } else {
            return defineClass(name, classData, 0, classData.length);
        }
    }

    private byte[] loadClassData(String className) throws IOException {
        String fileName = root + File.separatorChar + className.replace('.', File.separatorChar) + ".class";
        InputStream inputStream = null;
        ByteArrayOutputStream byteArrayOutputStream = null;
        try {
            inputStream = new FileInputStream(fileName);
            byteArrayOutputStream = new ByteArrayOutputStream();
            int bufferSize = 1024;
            byte[] buffer = new byte[bufferSize];
            int length = 0;
            while ((length = inputStream.read(buffer)) != -1) {
                byteArrayOutputStream.write(buffer, 0, length);
            }
            return byteArrayOutputStream.toByteArray();
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            if (byteArrayOutputStream != null) {
                byteArrayOutputStream.close();
            }
            if (inputStream != null) {
                inputStream.close();
            }
        }

        return null;
    }

    public String getRoot() {
        return root;
    }

    public void setRoot(String root) {
        this.root = root;
    }
}
复制代码
```

执行测试
----

启动 main 方法，执行测试

```
public static void main(String[] args) {
        MyClassLoader classLoader = new MyClassLoader();
        classLoader.setRoot("D:\\Temp");
        Class<?> testClass = null;
        try {
            testClass = classLoader.loadClass("cn.ybzy.demo.Test");
            Object object = testClass.newInstance();
            System.out.println(object.getClass().getClassLoader());
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        } catch (InstantiationException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        }
    }
复制代码
```

```
cn.ybzy.demo.MyClassLoader@5679c6c6
复制代码
```

将 Test 类放到项目类路径下，由于双亲委托机制的存在，会直接导致该类由 AppClassLoader 加载，而不会通过自定义类加载器来加载 ![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/665c853008e0499a80068f5e4da9e13b~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

```
sun.misc.Launcher$AppClassLoader@18b4aac2
复制代码
```

注意事项
----

```
1、这里传递文件名需要是类的全限定性名称，因为defineClass方法是按这种方式/格式进行处理
   因此，若没有全限定名，需要将类的全路径加载进去

2、不要重写loadClass方法，因为这样容易破坏双亲委托模式

3、Test类本身可以被AppClassLoader类加载，因此不能把Test.class放在类路径下
   否则，由于双亲委托机制的存在，会直接导致该类由AppClassLoader加载，而不会通过自定义类加载器来加载
复制代码
```