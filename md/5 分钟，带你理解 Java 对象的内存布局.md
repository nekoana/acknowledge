> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7215528103467515964)

Java 对象
=======

前言
--

本文将教你如何计算 **Java 对象**在**内存**中的**大小**。

当你 **new** 一个对象时，如果你对它在内存中，到底什么样，究竟占多大内存感兴趣。本文可以快速的给您答案。  
文章中的例子，默认 **JVM** 为 **64 位**，无压缩。

内存布局
----

**Java 内存布局** = **对象头** + **实例数据** + **对齐填充**

对象内存大小
------

**普通 Java 对象** = **对象头（16 字节）** + **实例数据（各种变量累计大小）** + **对齐填充（补充至 8 的整数倍）**

**数组对象** = **对象头（12 字节）** + **实例数据（各种变量累计大小）** + **对齐填充（补充至 8 的整数倍）**

只看上面对于没接触过这方面的同学，肯定有难度，这里只是先给大家个印象。别急，等看完下面你就会知道具体的含义了。

图片讲解
----

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/47d2b38176fd4ff9b5f6863b5212218e~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)
-------------------------------------------------------------------------------------------------------------------------------------------

### 1. 对象头

**对象头**由，**markword**+ **类型指针** + **数组长度**组成。  
普通对象：**markword**+ **类型指针** = 8 字节 + 4 字节 = **12 字节**  
数组对象：**markword**+ **类型指针** + **数组长度** = 8 字节 + 4 字节 +4 字节 = **16 字节**  

#### 1.1 markword

在 64 位 JVM 中,**markword** 是一个 **8 字节**的数据。一字节 8 位，所以 **markword** 是一个 **64 位**的数据。  

在 32 位 JVM 中，结果不同，这里不做赘述。  

##### markword 图解

不同的位数，代表含义不同。这里引用 [Mark Word 详解](https://link.juejin.cn?target=https%3A%2F%2Fblog.csdn.net%2Fweixin_40816843%2Farticle%2Fdetails%2F120811181 "https://blog.csdn.net/weixin_40816843/article/details/120811181")文章中的图片。

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f3125ea869e44b4bb2519b143d64dc2b~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

##### 无锁对象的 markword

可能没接触过的同学看不懂该图，我再举个例子。我们都知道可以通过 **synchronized** 锁对象，来做同步操作。  

一个对象创建出来没有被锁过，它的 **markword** 应该是下图这样的。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4000f15d7c6b4dbab7afac8b29316632~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?) 其余锁的状态看法，可以参考上图。  
这里需要注意，hashCode 的值，只有在调用 hashcode() 方法后，才会改变值。

#### 1.2 类型指针

大小为 4 字节，指向方法区中，该对象的类型。

#### 1.3 数组长度

只有当创建的是**数组**时，才有该部分，大小为 **4 字节**，代表当前**数组的长度**。**非数组**时，**不存在**。  

即：**普通对象对象头** = **markword** + **类型指针**  
**数组对象** = **markword** + **类型指针** + **数组长度**  

### 2. 实例数据

一个对象的实例数据大小，等于它所有成员变量大小，以及继承类的变量的大小的和。  

如果你的对象**不继承**任何对象，只有一个 **int 型**变量。那么你的**实例数据**大小就为 **4 字节**。  

如果对象 A 的父类有一个 boolean 类型变量，对象 A 有一个 char 类型变量。  
A 对象的**实例数据**大小 = **boolean（1 字节**）+**char（2 字节**）  

无论父类的变量是 **private** 还是 **public** 修饰，都会算在子类的**实例数据**中。父类的父类也会算在实例数据中。

注意一点，**静态变量**存放在**方法区**，因此不占用对象内存。  

### 3. 对齐填充

一个对象大小必须为 8 的整数倍。如果对象头 + 实例数据 = 57 个字节。那么对齐填充就为 7 字节。对象头 + 实例数据 + 对齐填充 = 64 字节。正好 8 的整数倍。你可以理解为对齐填充就是凑数用的。

实例讲解
----

下面通过一个具体例子，以及运行结果，来印证下对象内存布局是否如上图所说。  
本文含有运行后的结果。大家可以直接通过代码和结果来加深印象，无需实机运行代码。

### 如何查看对象

引入一个三方库：**jol**，通过这个库，可以打印对象**内存布局**。

```
implementation group: 'org.openjdk.jol', name: 'jol-core', version: '0.17'
复制代码
```

### 辅助类 Student

```
/**
 * Author（作者）：jtl
 * Date（日期）：2023/3/26 19:20
 * Detail（详情）：学生类
 */
public class Student extends Person{
    String name = "ZhangSan";// 对象引用：4字节
    boolean isRealMan;//boolean：1字节
    int age = 10;//boolean：1字节
    char score ='A';//char：2字节
    long time = 20230326;//long：8字节
    Student deskMate;//对象引用：4字节

    public Student() {
    }

    static int grade;

    public boolean isRealMan() {
        return isRealMan;
    }

    public void setRealMan(boolean realMan) {
        isRealMan = realMan;
    }
}
复制代码
```

### Student 的父类 Person

该类为了印证，**创建 Student** 对象时，**Person 类**的所有变量（包括**私有对象**），都会占用 **Student 对象**的**实例数据**大小。

```
/**
 * Author（作者）：jtl
 * Date（日期）：2023/3/28 16:07
 * Detail（详情）：父类，Person类
 */
public class Person extends Object {
    private float time = 0L;
    String country = "China";
}
复制代码
```

### 实际运行代码

这里举了 5 种例子，通过下文的运行结果，大家可以进行对比，加深印象。

```
public class Main {
    public static void main(String[] args) throws InterruptedException {
        Student student = new Student();
        System.out.println("-----------------------------第一阶段 new 对象-------------------------------");
        System.out.println(ClassLayout.parseInstance(student).toPrintable());

        System.out.println("-----------------------------第二阶段 hashcode-------------------------------");
        student.hashCode();
        System.out.println(ClassLayout.parseInstance(student).toPrintable());



        synchronized (student){
            System.out.println("-----------------------------第三阶段 synchronized加锁-------------------------------");
            System.out.println("synchronized (student)：\n"+ClassLayout.parseInstance(student).toPrintable());
        }

        new Thread(new Runnable() {
            @Override
            public void run() {
                synchronized (student){
                    System.out.println("-----------------------------第四阶段 synchronized加锁-------------------------------");
                    System.out.println("new Thread:synchronized (student)：\n"+ClassLayout.parseInstance(student).toPrintable());
                }
            }
        }).start();

        synchronized (student) {
            System.out.println("-----------------------------第五阶段 Student[] 数组-------------------------------");
            System.out.println(ClassLayout.parseInstance(new Student[]{new Student(), new Student(), new Student()}).toPrintable());
        }
    }
}
复制代码
```

### 内存布局

这里可以看到的是，只有调用了 **hashcode 方法**后。**markword** 中才会修改对应位置的数据。 另外只有对象为**数组**时，才会有**数组长度 (array length)** 这个选项，大小为 **4 字节**。

```
> Task :Main.main()
-----------------------------第一阶段 new 对象-------------------------------
org.example.Student object internals:
OFF  SZ                  TYPE DESCRIPTION               VALUE
  0   8                       (object header: mark)     0x0000000000000001 (non-biasable; age: 0)
  8   4                       (object header: class)    0x01000be8
 12   4                 float Person.time               0.0
 16   4      java.lang.String Person.country            (object)
 20   4                   int Student.age               10
 24   8                  long Student.time              20230326
 32   2                  char Student.score             A
 34   1               boolean Student.isRealMan         false
 35   1                       (alignment/padding gap)   
 36   4      java.lang.String Student.name              (object)
 40   4   org.example.Student Student.deskMate          null
 44   4                       (object alignment gap)    
Instance size: 48 bytes
Space losses: 1 bytes internal + 4 bytes external = 5 bytes total

-----------------------------第二阶段 hashcode-------------------------------
org.example.Student object internals:
OFF  SZ                  TYPE DESCRIPTION               VALUE
  0   8                       (object header: mark)     0x00000073035e2701 (hash: 0x73035e27; age: 0)
  8   4                       (object header: class)    0x01000be8
 12   4                 float Person.time               0.0
 16   4      java.lang.String Person.country            (object)
 20   4                   int Student.age               10
 24   8                  long Student.time              20230326
 32   2                  char Student.score             A
 34   1               boolean Student.isRealMan         false
 35   1                       (alignment/padding gap)   
 36   4      java.lang.String Student.name              (object)
 40   4   org.example.Student Student.deskMate          null
 44   4                       (object alignment gap)    
Instance size: 48 bytes
Space losses: 1 bytes internal + 4 bytes external = 5 bytes total

-----------------------------第三阶段 synchronized加锁-------------------------------
synchronized (student)：
org.example.Student object internals:
OFF  SZ                  TYPE DESCRIPTION               VALUE
  0   8                       (object header: mark)     0x000070000dcafac0 (thin lock: 0x000070000dcafac0)
  8   4                       (object header: class)    0x01000be8
 12   4                 float Person.time               0.0
 16   4      java.lang.String Person.country            (object)
 20   4                   int Student.age               10
 24   8                  long Student.time              20230326
 32   2                  char Student.score             A
 34   1               boolean Student.isRealMan         false
 35   1                       (alignment/padding gap)   
 36   4      java.lang.String Student.name              (object)
 40   4   org.example.Student Student.deskMate          null
 44   4                       (object alignment gap)    
Instance size: 48 bytes
Space losses: 1 bytes internal + 4 bytes external = 5 bytes total

-----------------------------第五阶段 Student[] 数组-------------------------------
[Lorg.example.Student; object internals:
OFF  SZ                  TYPE DESCRIPTION               VALUE
  0   8                       (object header: mark)     0x0000000000000001 (non-biasable; age: 0)
  8   4                       (object header: class)    0x0101ed10
 12   4                       (array length)            3
 16  12   org.example.Student Student;.<elements>       N/A
 28   4                       (object alignment gap)    
Instance size: 32 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

-----------------------------第四阶段 synchronized加锁-------------------------------
new Thread:synchronized (student)：
org.example.Student object internals:
OFF  SZ                  TYPE DESCRIPTION               VALUE
  0   8                       (object header: mark)     0x000060000265c002 (fat lock: 0x000060000265c002)
  8   4                       (object header: class)    0x01000be8
 12   4                 float Person.time               0.0
 16   4      java.lang.String Person.country            (object)
 20   4                   int Student.age               10
 24   8                  long Student.time              20230326
 32   2                  char Student.score             A
 34   1               boolean Student.isRealMan         false
 35   1                       (alignment/padding gap)   
 36   4      java.lang.String Student.name              (object)
 40   4   org.example.Student Student.deskMate          null
 44   4                       (object alignment gap)    
Instance size: 48 bytes
Space losses: 1 bytes internal + 4 bytes external = 5 bytes total
复制代码
```

上面结果的看法如下图所示：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8b9ba0f0a4454e908e9a812112d969e8~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?) 另外，对**齐填充**大小如下图所示。最终**对象大小**，一定是 **8 的整数倍**。

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5f5c6368d9734f29a1bb79963c669274~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

至于 **(alignment/padding gap)** 什么情况下会出现。本人没有仔细研究，感兴趣的同学可以看下 **jol 的源码**。下图为源码中的具体位置。

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b7c9375573454f3ea829d0b681d755c4~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)
-------------------------------------------------------------------------------------------------------------------------------------------

结尾
--

至此，Java 对象内存布局就讲完了。不知道大家还记住多少呢。如果怕看了不久就忘了，可以**收藏**本文，**点个赞**支持下作者。文章中若有错误，可以及时指正。  

今年 Java 找工作异常艰难，希望大家都可以度过这个寒冬。加油！！！  

上文的示例代码：[Java 内存布局](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2F13046434521%2FJavaLayout.git "https://github.com/13046434521/JavaLayout.git") 感兴趣的同学，可以下载运行一下。