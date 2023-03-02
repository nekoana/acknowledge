> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7189051824148742200)

最近有同学在做 APM 链路监控发现了一个诡异的类被加载的问题，没有被调用到的函数里面用到的类，居然触发了类加载，于是结合 JVM 的源码做了一下分析，过程如下：

现象描述
----

简化后有如下几个类，其中 `IParent` 是一个抽象类，`ChildImpl` 是 `IParent` 的子类，TestB 类有一个 test 方法，入参为 IParent，TestA 有两个静态方法，一个 foo，一个 test，其中 test 方法中新建了一个 ChildImpl 作为入参调用了 testB 的 test 方法。

```
public abstract class IParent {
}

public class ChildImpl extends IParent {
    static {
        System.out.println("init child impl");
    }
}

public class TestB {
    public void test(IParent obj) {
        System.out.println(obj);
    }
}

public class TestA {
    public static void test() {
        TestB testB = new TestB();
        testB.test(new ChildImpl());
    }

    public static void foo() {
        System.out.println("in hello foo");
    }
}
复制代码
```

接下来是在 main 方法中调用 TestA 的 foo 方法

```
package me.ya;

public class Main {
    public static void main(String[] args) {
        TestA.foo();
    }
}
复制代码
```

那么问题是:

> 只调用 TestA 的 foo 方法，不调用 test 方法，会触发 IParent 和 ChildImpl 的类加载吗？

从 idea 的代码提示也可以确认 TestA 的 test 方法是没有人调用的。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bdf70b49720549248f80ce5e78f55ad7~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

通过 jvm 启动参数 `-verbose:class` 查看类加载的情况：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1c8ea14ba0bf47ad8eb8aaeb8d5b6443~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

`IParent` 和 `ChildImpl` 这两个类居然被加载了。

如果我们把 testB 的 test 方法做一下修改，直接用 ChildImpl 作为函数参数，不存在多态继承的情况下，就不会对应类的加载。

```
public class MyChild {
}


public class TestB {
    public void test(IParent obj) {
        System.out.println(obj);
    }
    public void test_v2(ChildImpl obj) {
        System.out.println(obj);
    }
}

public class TestA {
    public static void test() {
        TestB testB = new TestB();
//        testB.test(new ChildImpl());
        testB.test_v2(new ChildImpl());
    }

    public static void foo() {
        System.out.println("in hello foo");
    }
}
复制代码
```

这个时候不会触发 `IParent` 和 `ChildImpl` 的类加载。看到这里，可能有同学已经猜到了，是因为多态导致了对应的问题出现。接下来我们从 JVM 源码的角度看一下这个过程。

JVM 源码调试分析
----------

通过简单的代码阅读，找到了一个比较理想的断点来分析这个问题，在函数`VerificationType::is_reference_assignable_from` 上打一个断点。

```
b VerificationType::is_reference_assignable_from
复制代码
```

通过 bt 查看此函数的调用堆栈如下：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/108716d8f19a4a02b4eb5a10e19ba088~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

`is_reference_assignable_from` 就是检查一个类型 to （当前类型）是否可以被 from 类型赋值，逻辑很清晰：

首先判断 from 是否为 null，如果 from 为 null，则是合法的，null 可以赋值给任意对象引用和数组类型，比如

```
Foo to = null; // 合法
复制代码
```

接下来判断 from 和 to 类型签名是否相同，如果是同一类型，则一定是合法的，比如：

```
Foo from = new Foo();
Foo to = from; // 合法
复制代码
```

如果 to 类型是对象引用类型，则判断 from 和 to 是否有继承关系。首先判断一个特殊情况，如果 to 是 `java.lang.Object` 这个上帝类型，则一定是成功的。

```
Foo from = new Foo();
Object to = from; // 合法
复制代码
```

如果 to 不是上帝类型，则需要加载 to 的对应类文件，以便获取更多的信息。

接下来判断 to 类型是不是接口类型，如果 to 是接口类型，则直接返回成功（why？）。

如果 to 类型不是接口类型，则需要把 from 类型的类文件也加载进来，判断 from 类型是不是 to 的子类型。

```
public class Bar extends Foo {
}
Bar from = new Bar();
Foo to = from; // 合法
复制代码
```

完整的代码如下：

```
bool VerificationType::is_reference_assignable_from(
    const VerificationType& from, ClassVerifier* context,
    bool from_field_is_protected, TRAPS) const {
  instanceKlassHandle klass = context->current_class();
  if (from.is_null()) { // 如果 from 为 null，则合法（null 可以复制给任意对象和数组类型）
    // null is assignable to any reference
    return true;
  } else if (is_null()) {
    return false;
  } else if (name() == from.name()) { // 如果 from 和 to 是同一类型，则也合法
    return true;
  } else if (is_object()) { // 对象引用类型
    // 如果 to 是一个对象类型，则需要检查 from 和 to 类型是否有继承关系
    if (name() == vmSymbols::java_lang_Object()) {
      // 如果 to 为 java.lang.Object 类型，则也是合法
      return true;
    }
    // 加载 to 类型的 class 文件以便获取更多的信息
    Klass* obj = SystemDictionary::resolve_or_fail(
        name(), Handle(THREAD, klass->class_loader()),
        Handle(THREAD, klass->protection_domain()), true, CHECK_false);
    KlassHandle this_class(THREAD, obj);

    // 如果是接口类型，则认为合法（至于为什么，英文注释里有）
    if (this_class->is_interface()) {
        // We treat interfaces as java.lang.Object, including
        // java.lang.Cloneable and java.io.Serializable
      return true;
    } else if (from.is_object()) { // 如果 from 是对象引用类型
      // 则需要加载 from 类型的类文件
      Klass* from_class = SystemDictionary::resolve_or_fail(
          from.name(), Handle(THREAD, klass->class_loader()),
          Handle(THREAD, klass->protection_domain()), true, CHECK_false);
          // 判断 from 是否是 to 的子类型
      bool result = InstanceKlass::cast(from_class)->is_subclass_of(this_class());
      return result;
    }
  } else if (is_array() && from.is_array()) { // 判断数组的情况
    VerificationType comp_this = get_component(context, CHECK_false);
    VerificationType comp_from = from.get_component(context, CHECK_false);
    if (!comp_this.is_bogus() && !comp_from.is_bogus()) {
      return comp_this.is_assignable_from(comp_from, context,
                                          from_field_is_protected, THREAD);
    }
  }
  return false;
}
复制代码
```

回到我们的例子中，通过 gdb 我们可以查看 from 和 to 类型。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b659403c7d484068b4c3049a01720682~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

可以看到 to 的类型为抽象类，`me/ya/IParent`，from 为实现类 `me/ya/ChildImpl`

接下来继续往下，通过判断 to 是一个对象类型，则加载 `me/ya/IParent` 类

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/09ba4487074d4cdcb7b94f604bb8c0f6~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

接下来加载 `me/ya/ChildImpl` 类来判断 from 和 to 是否有父子类关系。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/68c0945be91040a6a6d71ea3b9bb4d89~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

到这里就很清楚为什么函数没有被调用到，函数内用到的类竟然被加载了。

简单总结就是：TestB 类被加载的过程需要进行校验类文件的合法性，其中一项就是函数调用的参数赋值是否合法。涉及多态的情况，需要判断 from 是否是 to 的子类型（需要加载类文件）

继续尝试，如果 IParent 为接口类型
---------------------

按照前面的代码逻辑，如果 IParent 为接口类型，代码中会直接判断为合法，不会触发 ChildImpl 类型的加载。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1368a07896784b73ba4b6dc69926c67e~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

将 IParent 改为接口

```
public interface IParent {
}
public class ChildImpl implements IParent {
}
复制代码
```

再次运行会发现，确实没有加载 ChildImpl 类。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0204ef9d8c124d3d872e2d1de9ba343b~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

回到一开始的实验，如果不使用多态，直接用子类型 ChildImpl 作为函数参数类型时，不会触发 from 和 to 类型的加载

```
public void test_v2(ChildImpl obj) {
    System.out.println(obj);
}
复制代码
```

因为不存在多态，from 和 to 的类型一样不需要后续加载类文件来进一步判断，所以不会触发对应类型的加载。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/741e79241b424b2b80ffe84435bae075~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

后记
--

调试是一个好东西，可以帮我们理解很多冷知识。