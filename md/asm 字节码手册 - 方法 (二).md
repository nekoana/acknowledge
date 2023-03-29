> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7214015651827859511#heading-7)

基本概念
----

这一小节 内容会比较枯燥，也比较难。 个人建议的就是 有空读一下 **揭秘 java 虚拟机** 这本书的相关章节，然后再来看这一小节，理解起来会更容易一些。

另外这一小节最佳的学习方法还是自己下个 asm bytecode viewer 的插件，然后自己多写代码 然后比对着翻译后的字节码指令 多体会。

### 常见的字节码指令

字节码指令由一个标识该指令的操作码和固定数目的参数组成：

*   操作码是一个无符号字节值——即字节代码名，由注记符号标识
*   参数是静态值，确定了精确的指令行为，紧跟在操作码之后 字节码指令大致可以分为两类，一类指令用于局部变量和操作数栈之间传递值。另一类用于对操作数栈的值 进行弹出和计算，并压入栈中。

常见的局部变量操作指令有：

*   ILOAD：用于加载 boolean、int、byte、short 和 char 类型的局部变量到操作数栈
*   FLOAD：用于加载 float 类型局部变量到操作数栈
*   LLOAD：用于加载 lang 类型局部变量到操作数栈，需要加载两个槽 slot
*   DLOAD：用于加载 double 类型局部变量到操作数栈，需要加载两个槽 slot
*   ALOAD：用于加载非基础类型的局部变量到操作数栈，比如对象之类的

常见的操作数栈指令有：

*   ISTORE：从操作数栈弹出 boolean、int、byte、short 和 char 类型的局部变量，并将它存储在由其索引 i 指定的局部变量中
*   FSTORE：从操作数栈弹出 float 类型的局部变量，并将它存储在由其索引 i 指定的局部变量中
*   LSTORE：从操作数栈弹出 long 类型的局部变量，并将它存储在由其索引 i 指定的局部变量中
*   DSTORE：从操作数栈弹出 double 类型的局部变量，并将它存储在由其索引 i 指定的局部变量中
*   ASTORE：用于弹出非基础类型的局部变量，并将它存储在由其索引 i 指定的局部变量中

通过上面可以看到，每种对应的数据类型都对应不同的 `XLOAD` 或 `XSTORE`。这是为了保证不会执行 非法的转换。

*   将一个值存储在局部变量表中，在以不同的类型加载它是非法操作，比如存入 ISTORE 类型，使用 FLOAD 加载
*   如果向一个局部变量表中的位置存储一个值，而这个值不同于原来的存储类型，这种操作是合法的

### 局部变量表

局部变量表是一组变量值的存储空间，用于存放方法参数和方法内部定义的局部变量。**在 .java 编译成 .class 文件时，已经确定了所需要分配的局部变量表和操作数栈的大小。**

### 变量槽

**局部变量表的容量单位是变量槽 (Variable Slot)。每个变量槽最多可以存储 32 为长度的空间**，所以对于 byte、char、boolean、short、int、float、reference 是占用一个变量槽， **对于 64 位长度的 long 和 double 占用连个变量槽**。 使用局部变量表时，通过索引定位对应数据的位置，索引值的范围是从 0 开始至局部变量表最大的变量槽数量。 如果访问的是 32 位数据类型的变量，索引 N 就代表了使用第 N 个变量槽，如果访问的是 64 位数据类型的变量，则说明会同时使用第 N 和 N+1 两个变量槽。 对于两个相邻的共同存放一个 64 位数据的两个变量槽，虚拟机不允许采用任何方式单独访问其中的某一个，如果遇到进行这种操作的字节码，Java 虚拟机就会在类加载的校验阶段中抛出异常。

当一个方法被调用时，会使用局部变量表来完成参数值到参数变量列表的传递过程。**如果执行的是对象实例的成员方法（不是 static 修饰的方法），那么局部变量表中第 0 位索引的变量槽默认就是该对象实例的引用 (this)，在方法中可以通过关键字 this 来访问到这个隐含的参数**。 其余参数则按照参数表顺序排列，**参数表分配完毕后，再根据方法体内部定义的局部变量顺序和作用域分配其余的变量槽。** 为了尽可能节省栈帧所耗的内存空间，局部变量表中的变量槽是可以重用的，当方法体中定义的局部变量超出其作用域时，该局部变量对应的变量槽就可以交给其他变量来重用。

### 虚拟机重要概念

JVM 中的解释器会逐条解释执行字节码指令。执行过程中，JVM 会维护一个栈，用于存储局部变量、操作数和返回地址。在执行每个指令时，JVM 从栈中取出操作数，执行相应的操作，然后将结果压回栈中。例如，加法指令会从栈中弹出两个数值，相加后将结果压回栈中。

我们可以看一个简单的例子

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8dc3cda9461847a4b862af9b002c4292~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

可以看下 字节码：

很简单的，取第一个局部变量，取第二个局部变量， 然后 调用 add 指令，最后 return 返回值

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3d0f3acb04694778b559b692c611c3c8~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

MAXSTACK 和 MAXLOCALS 是 Java 字节码文件中的两个属性，用于定义方法的最大操作数栈和最大局部变量表大小。

**MAXSTACK 表示该方法运行时所需的最大操作数栈的深度，即在方法执行过程中所需要使用的最大的栈空间。** 这个值是在编译期间计算出来的，因为 Java 字节码是一种基于栈的语言，因此在方法运行时，JVM 会按照该值来为方法分配栈空间。

**MAXLOCALS 则表示该方法运行时所需的最大局部变量表的大小，即该方法所使用的局部变量的数量**。这个值也是在编译期间计算出来的，因为在方法执行过程中，需要为局部变量分配内存空间。

这里有的人会觉得奇怪，为什么局部变量表的大小是 3， 我这里明明只有 2 个局部变量 分别是参数 x 和参数 y 啊

因为这个方法不是一个静态方法，他是一个类的对象方法，对于这种方法来说，会有一个隐藏的参数 也就是 他自己 就是 this， 所以这里是局部变量表的大小是 3

这里要谨记 **所有类的成员方法 都有一个隐藏的入参 参数 this**

### databean 的 字节码分析

看一下 get 方法

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b3065df7caf5427aaf80603e1dce9e35~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

看下字节码：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/554da9ab1a1846df9155d6367312c1ab~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

第一条指令 ALOAD 0 ，这个其实就是 将 this 压入操作数栈。 上一个小节我们提到过 类的成员方法 都有一个隐藏的入参 是 this ，位置是在第一个位置 也就是 0 位置上

第二条指令 就是 从栈中弹出这个值， 并将这个对象的 name 字段 压入栈中，

最后一条指令 就是从栈中弹出这个值了

再看下 set 方法

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cae28994678b4153a95c2a11287eeccc~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

第一条指令 前面讲过了 不说了

第二条指令就是 把函数的参数 这个局部变量 压入操作数栈

第三条指令 弹出这 2 个值，并将 int 值 存储在 name 字段中

最后一条指令 是 return，对于 void 方法来说，这里都是隐藏的一条 retuan 指令

再看下这个 databean 的构造方法：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8a1ea25b82254353a9025661bc986bcd~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

第一条指令 不说了 第二条指令 其实就是 调用 Object 的构造器

INVOKESPECIAL 指令主要用于两种情况：

1.  **调用超类的构造方法**：在一个子类的构造方法中，如果需要调用超类的构造方法完成对象初始化，可以使用 INVOKESPECIAL 指令来调用超类的构造方法。
2.  **调用私有方法**：在 Java 中，私有方法不能被子类重写或继承，但是它们可以在同一类中被其他方法调用。这种情况下，可以使用 INVOKESPECIAL 指令来调用私有方法。

总之，INVOKESPECIAL 指令是 Java 字节码中的一种重要指令，它主要用于调用超类构造器或私有方法。

### 一个稍微复杂点的 set 方法

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/52d30bec602a43148860e05754d0595f~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

在 Java 字节码中，**IFLE 是一种条件分支指令，它用于在栈顶值小于或等于 0 时跳转到指定的目标指令**。

这个就很好理解了吧， 这里是先取的 局部变量表中 index 为 1 的 局部变量 也就是函数参数的 age

对他进行压栈操作，然后 IFLE 指令 对他进行判断，小于等于 0 的时候

这里要注意的就是 在 java 中 抛出一个异常的固定指令规范 是 **先 new 再 dup 再 invokespecial**

### 异常处理器

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fa33e9a2606f4f238b4e03465c6bfcc6~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

**TRYCATCHBLOCK 指令需要四个参数**，分别是：

1.  第一个参数是一个指向 try 块的起始位置的偏移量（从字节码方法的开头开始计算）。
2.  第二个参数是一个指向 try 块的结束位置的偏移量。
3.  第三个参数是一个指向 catch 块的起始位置的偏移量。
4.  第四个参数是一个指向 catch 块异常处理程序代码的异常类型。

接口与组件
-----

上一篇文章中 可以利用 visitor 来遍历一个类的方法和字段，同样的，对于方法来说，我们也可以用 visitor 来遍历这个方法

上一小节我们有了 acccode 的扩展函数，这会我们添加一个 **opcode 的扩展函数 **

```
fun Int.opCode2String(): String {
    mapOpcodes.forEach {
        if (it.value == this) {
            return it.key
        }
    }
    return ""
}
复制代码
```

还是要走 ClassReader 只不过这次我们在 ClassVisitor 的 method 回调方法中，传入了 我们自定义的 MethodVisitor

```
val classReader = ClassReader("TestMethod")
val classVisitor = object : ClassVisitor(Opcodes.ASM7) {
    override fun visitMethod(
        access: Int,
        name: String?,
        descriptor: String?,
        signature: String?,
        exceptions: Array<out String>?
    ): MethodVisitor {
        // 最关键就是这里要返回一个自定义的 MethodVisitor
        val mv = super.visitMethod(access, name, descriptor, signature, exceptions)
        return MethodPrintVisitor(Opcodes.ASM7, mv)
    }
}
classReader.accept(classVisitor, ClassReader.SKIP_DEBUG)
复制代码
```

**因为每个方法的字节码都不同，所以这些回调函数的顺序也是不同的，但基本上都是遵循 visitCode 开始 和 visitMaxs 结束**

这 2 个回调之间的 那些回调函数的顺序以及出现次数 就完全取决于 你函数内部的实现了

```
package com.andoter.asm_example.part3.vivo

import com.andoter.asm_example.part2.vivo.Utils.opCode2String
import org.objectweb.asm.*


/**
 * MethodVisit 首先会访问注释和属性信息，然后才是方法的字节码，这些访问顺序在 visitCode 和 visitMaxs 调用之间。所以
 * 这两个方法可以用于检测字节码在访问序列中的开始和结束。ASM 中提供了三个基于 MethodVisitor API 的组件，用于生成和转化方法。
 *
 * - ClassReader 类分析已编译方法的内容，其在 accept 方法的参数中传递了 ClassVisitor，ClassReader 类将针对这一 ClassVisitor 返回的
 *  MethodVisitor 对象调用响应的方法。
 *  - ClassWriter 的 visitMethod 方法返回 MethodVisitor 接口的一个实现，它直接以二进制形式生成已编译方法
 *  - MethodVisitor类将它接收到的所有方法调用委托给另一个MethodVisitor方法。可以将它看作一个事件筛选器。
 *
 *  MethodVisitor 回调方法有：
 *  - visitParameter：访问方法一个参数
 *  - visitAnnotationDefualt：访问注解接口方法的默认值
 *  - visitAnnotaion：访问方法的一个注解
 *  - visitTypeAnnotation：访问方法签名上的一个类型的注解
 *  - visitAnnotableParameterCount：访问注解参数数量，就是访问方法参数有注解参数个数
 *  - visitParameterAnnotation：访问参数的注解，返回一个 AnnotationVisitor 可以访问该注解值
 *  - visitAttribute：访问方法的属性
 *  - 重要----visitCode：开始访问方法代码，此处可以添加方法运行前拦截器 ，这个回调可以认为是方法访问的开始
 *  - visitFrame：访问方法局部变量的当前状态以及操作栈成员信息
 *  - visitIntInsn：访问数值类型指令,当 int 取值-1~5采用 ICONST 指令，取值 -128~127 采用 BIPUSH 指令，取值 -32768~32767 采用 SIPUSH 指令，取值 -2147483648~2147483647 采用 ldc 指令。
 *  - 重要---- visitVarInsn：访问本地变量类型指令
 *  - visitTypeInsn：访问类型指令，类型指令会把类的内部名称当成参数 Type
 *  - 重要-----visitFieldInsn：域操作指令，用来加载或者存储对象的 Field
 *  - 重要---- visitMethodInsn：访问方法操作指令
 *  - visitDynamicInsn：访问动态类型指令
 *  - 重要----visitJumpInsn：访问比较跳转指令
 *  - visitLabelInsn：访问 label，当会在调用该方法后访问该label标记一个指令
 *  - visitLdcInsn：访问 LDC 指令，也就是访问常量池索引
 *  - visitLineNumber：访问行号描述
 *  - 重要-----visitMaxs：访问操作数栈最大值和本地变量表最大值 可以认为是方法访问的结束
 *  - visitLocalVariable：访问本地变量描述
 */
class MethodPrintVisitor(api: Int, methodVisitor: MethodVisitor?) : MethodVisitor(api, methodVisitor) {

    override fun visitMultiANewArrayInsn(descriptor: String?, numDimensions: Int) {
        super.visitMultiANewArrayInsn(descriptor, numDimensions)
        println("visitMultiANewArrayInsn, descriptor = $descriptor, numDimensions = $numDimensions")
    }

    override fun visitFrame(
        type: Int,
        numLocal: Int,
        local: Array<out Any>?,
        numStack: Int,
        stack: Array<out Any>?
    ) {
        super.visitFrame(type, numLocal, local, numStack, stack)
        println("visitFrame, type = $type, numLocal = $numLocal, local.size = $(local.size), numStack = $numStack")
    }

    /**
     *  visitVarInsn：访问本地变量类型指令
     */
    override fun visitVarInsn(opcode: Int, `var`: Int) {
        super.visitVarInsn(opcode, `var`)
        println("visitVarInsn, opcode = ${opcode.opCode2String()}, var = $`var`")
    }

    override fun visitTryCatchBlock(start: Label?, end: Label?, handler: Label?, type: String?) {
        super.visitTryCatchBlock(start, end, handler, type)
        println("visitTryCatchBlock")
    }

    override fun visitLookupSwitchInsn(dflt: Label?, keys: IntArray?, labels: Array<out Label>?) {
        super.visitLookupSwitchInsn(dflt, keys, labels)
        println("visitLookupSwitchInsn")
    }

    /**
     * visitJumpInsn：访问比较跳转指令
     */
    override fun visitJumpInsn(opcode: Int, label: Label?) {
        super.visitJumpInsn(opcode, label)
        println("visitJumpInsn, opcode = ${opcode.opCode2String()}")
    }

    override fun visitLdcInsn(value: Any?) {
        super.visitLdcInsn(value)
        println("visitLdcInsn, value = $value")
    }

    override fun visitAnnotableParameterCount(parameterCount: Int, visible: Boolean) {
        super.visitAnnotableParameterCount(parameterCount, visible)
    }

    override fun visitIntInsn(opcode: Int, operand: Int) {
        super.visitIntInsn(opcode, operand)
        println("visitIntInsn, opcode = ${opcode.opCode2String()}, operand = $operand")
    }

    override fun visitTypeInsn(opcode: Int, type: String?) {
        super.visitTypeInsn(opcode, type)
        println("visitTypeInsn, opcode = ${opcode.opCode2String()}, type = $type")
    }

    override fun visitAnnotationDefault(): AnnotationVisitor? {
        println("visitAnnotationDefault")
        return super.visitAnnotationDefault()
    }

    override fun visitAnnotation(descriptor: String?, visible: Boolean): AnnotationVisitor? {
        println("visitAnnotation")
        return super.visitAnnotation(descriptor, visible)
    }

    override fun visitTypeAnnotation(
        typeRef: Int,
        typePath: TypePath?,
        descriptor: String?,
        visible: Boolean
    ): AnnotationVisitor? {
        println("visitTypeAnnotation")
        return super.visitTypeAnnotation(typeRef, typePath, descriptor, visible)
    }

    /**
     * 方法访问的结束
     */
    override fun visitMaxs(maxStack: Int, maxLocals: Int) {
        super.visitMaxs(maxStack, maxLocals)
        println("visitMaxs")
    }

    override fun visitInvokeDynamicInsn(
        name: String?,
        descriptor: String?,
        bootstrapMethodHandle: Handle?,
        vararg bootstrapMethodArguments: Any?
    ) {
        println("visitInvokeDynamicInsn, name = $name, descriptor = $descriptor, bootstrapMethodHandle = ${bootstrapMethodHandle?.name}")
        super.visitInvokeDynamicInsn(
            name,
            descriptor,
            bootstrapMethodHandle,
            *bootstrapMethodArguments
        )
    }

    /**
     * 当ASM解析字节码时遇到一个标签时，它将调用MethodVisitor的visitLabel方法，并传递标签对象作为参数
      */
    override fun visitLabel(label: Label?) {
        super.visitLabel(label)
        println("visitLabel ")
    }

    override fun visitTryCatchAnnotation(
        typeRef: Int,
        typePath: TypePath?,
        descriptor: String?,
        visible: Boolean
    ): AnnotationVisitor? {
        return super.visitTryCatchAnnotation(typeRef, typePath, descriptor, visible)
    }

    override fun visitMethodInsn(opcode: Int, owner: String?, name: String?, descriptor: String?) {
        super.visitMethodInsn(opcode, owner, name, descriptor)
        println("visitMethodInsn, opcode = ${opcode.opCode2String()}, owner = $owner, name = $name, descriptor = $descriptor")
    }

    /**
     *  visitMethodInsn：访问方法操作指令
     */
    override fun visitMethodInsn(
        opcode: Int,
        owner: String?,
        name: String?,
        descriptor: String?,
        isInterface: Boolean
    ) {
        super.visitMethodInsn(opcode, owner, name, descriptor, isInterface)
        println("visitMethodInsn, opcode = ${opcode.opCode2String()}, owner = $owner, name = $name, descriptor = $descriptor")
    }

    /**
     * 当ASM在解析字节码时遇到一个不需要操作数的指令时（例如RETURN或NOP指令），、
     * 它将调用MethodVisitor的visitInsn方法，并传递该指令的操作码（opcode）作为参数。
     */
    override fun visitInsn(opcode: Int) {
        super.visitInsn(opcode)
        println("visitInsn, opcode = ${opcode.opCode2String()}")
    }

    override fun visitInsnAnnotation(
        typeRef: Int,
        typePath: TypePath?,
        descriptor: String?,
        visible: Boolean
    ): AnnotationVisitor? {
        println("visitInsnAnnotation")
        return super.visitInsnAnnotation(typeRef, typePath, descriptor, visible)
    }

    override fun visitParameterAnnotation(
        parameter: Int,
        descriptor: String?,
        visible: Boolean
    ): AnnotationVisitor? {
        println("visitParameterAnnotation")
        return super.visitParameterAnnotation(parameter, descriptor, visible)
    }

    override fun visitIincInsn(`var`: Int, increment: Int) {
        super.visitIincInsn(`var`, increment)

        println("visitIincInsn")
    }

    override fun visitLineNumber(line: Int, start: Label?) {
        super.visitLineNumber(line, start)
        println("visitLineNumber")
    }

    override fun visitLocalVariableAnnotation(
        typeRef: Int,
        typePath: TypePath?,
        start: Array<out Label>?,
        end: Array<out Label>?,
        index: IntArray?,
        descriptor: String?,
        visible: Boolean
    ): AnnotationVisitor? {
        println("visitLocalVariableAnnotation")
        return super.visitLocalVariableAnnotation(
            typeRef,
            typePath,
            start,
            end,
            index,
            descriptor,
            visible
        )
    }

    override fun visitTableSwitchInsn(min: Int, max: Int, dflt: Label?, vararg labels: Label?) {
        super.visitTableSwitchInsn(min, max, dflt, *labels)
        println("visitTableSwitchInsn")
    }

    override fun visitEnd() {
        super.visitEnd()
        println("visitEnd")
    }

    override fun visitLocalVariable(
        name: String?,
        descriptor: String?,
        signature: String?,
        start: Label?,
        end: Label?,
        index: Int
    ) {
        super.visitLocalVariable(name, descriptor, signature, start, end, index)
        println(
            "visitLocalVariable, name = $name, descriptor = $descriptor, " +
                    "signature = $signature, start = $start + end = $end, index = $index"
        )
    }

    override fun visitParameter(name: String?, access: Int) {
        super.visitParameter(name, access)
        println("visitParameter, name = $name, access = ${access.opCode2String()}")
    }

    override fun visitAttribute(attribute: Attribute?) {
        super.visitAttribute(attribute)
        println("visitAttribute")
    }

    /**
     * visitFieldInsn：域操作指令，用来加载或者存储对象的 Field
     */
    override fun visitFieldInsn(opcode: Int, owner: String?, name: String?, descriptor: String?) {
        super.visitFieldInsn(opcode, owner, name, descriptor)
        println("visitFieldInsn, opcode = ${opcode.opCode2String()}, owner = $owner, name = $name, descriptor = $descriptor")
    }

    /**
     *  开始访问方法代码，此处可以添加方法运行前拦截器 ，
     *  这个回调可以认为是方法访问的开始
     */
    override fun visitCode() {
        super.visitCode()
        println("visitCode")
    }
}
复制代码
```

这里给一个小例子 稍微体会一下 methodvisitor

比如说这个构造函数：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/93e92a8d62b240d2845ccc990d928d25~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

在 visit 的回调里面 就变成了：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c43e6c8c7cd147479c0f97facc682e1c~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

### 修改方法的指令 --- 无状态转换

这一小节，我们来尝试修改一下方法的指令 体会一下如何在字节码中修改一个方法

看一个简单的方法：

```
public class AddTimerTest {
    public void addTimer() throws InterruptedException {
        Thread.sleep(1000);
    }
}
复制代码
```

看一下他的字节码：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/99140ccd5b4a42ad8b2806414c9bb7e3~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

**LDC 指令是 Java 虚拟机（JVM）中的一种指令，用于将常量（Constant）推送到栈顶**

这个方法的指令特别简单，不解释了

假设此时我们想通过字节码的方法 把这个方法修改成：

```
public class AddTimerTest2 {
    private static long time = 0L;

    public void addTimer() throws InterruptedException {
        time -= System.currentTimeMillis();
        Thread.sleep(1000);
        time += System.currentTimeMillis();
    }
}
复制代码
```

看下他的字节码

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a191880712bd4f2e8fe2b1fde119cd0e~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

**LSUB 和 LADD 指令是 Java 虚拟机（JVM）中的两种指令，分别用于执行长整型（Long）的减法和加法操作。**

可以观察一下 这两个方法的字节码指令 区别就在于 开头和结尾 增加了 4 条指令，我们要做的就是 通过字节码修改的方法 来修改这个方法

```
fun main() {
    val classReader = ClassReader("com.andoter.asm_example.part3.vivo.AddTimerTest")
    // 第二个参数很重要，可以省略我们计算max的操作，绝大多数场景我们用这个参数就可以了
    val classWriter = ClassWriter(classReader, ClassWriter.COMPUTE_MAXS)
    classReader.accept(object : ClassVisitor(Opcodes.ASM7, classWriter) {
        var owner: String? = ""
        override fun visit(
            version: Int,
            access: Int,
            name: String?,
            signature: String?,
            superName: String?,
            interfaces: Array<out String>?
        ) {
            super.visit(version, access, name, signature, superName, interfaces)
            owner = name
        }

        // 不要忘记 新增一个static的变量
        override fun visitEnd() {
            val fieldVisitor =
                cv.visitField(Opcodes.ACC_PUBLIC + Opcodes.ACC_STATIC, "time", "J", null, null)
            fieldVisitor?.visitEnd()
            super.visitEnd()
        }

        override fun visitMethod(
            access: Int,
            name: String?,
            descriptor: String?,
            signature: String?,
            exceptions: Array<out String>?
        ): MethodVisitor {
            var methodVisitor = cv.visitMethod(access, name, descriptor, signature, exceptions)
            // 不是构造方法才需要添加方法修改逻辑
            if (methodVisitor != null && name != "<init>") {
                methodVisitor = object :MethodVisitor(Opcodes.ASM7, methodVisitor){
                    override fun visitCode() {
                        mv.visitCode()
                        // 在方法的开始处 添加字节码
                        mv.visitFieldInsn(Opcodes.GETSTATIC, owner, "time", "J")
                        mv.visitMethodInsn(
                            Opcodes.INVOKESTATIC,
                            "java/lang/System",
                            "currentTimeMillis",
                            "()J"
                        )
                        mv.visitInsn(Opcodes.LSUB)
                        mv.visitFieldInsn(Opcodes.PUTSTATIC, owner, "time", "J")

                    }

                    override fun visitInsn(opcode: Int) {
                        // 我们要在return语句之前 添加这段代码
                        if (opcode >= Opcodes.IRETURN && opcode <= Opcodes.RETURN || opcode == Opcodes.ATHROW) {
                            mv.visitFieldInsn(Opcodes.GETSTATIC, owner, "time", "J")
                            mv.visitMethodInsn(Opcodes.INVOKESTATIC, "java/lang/System", "currentTimeMillis", "()J")
                            mv.visitInsn(Opcodes.LADD)
                            mv.visitFieldInsn(Opcodes.PUTSTATIC, owner, "time", "J")
                        }
                        mv.visitInsn(opcode)
                    }
                }
            }
            return methodVisitor
        }

    }, ClassReader.SKIP_DEBUG)

    val bytes = classWriter.toByteArray()

    // 实际写文件
    File("asm_example/files2/AddTimerTest.class").sink().buffer().apply {
        write(bytes)
        flush()
        close()
    }
}
复制代码
```

我们当然也可以 实际运行一下这个类 看一下效果：

```
val myClassloader = MyClassloader()
val loadedClass = myClassloader.loadClassFromFile("/asm_example/files2/AddTimerTest.class", "com.andoter.asm_example.part3.xxx.AddTimerTest")
val method=loadedClass.getMethod("addTimer")
// 获取 time 字段
val field : Field? = loadedClass.getDeclaredField("time")
// 触发方法
method.invoke(loadedClass.newInstance())
// 打印 time 耗时
println("time = ${field?.getLong(loadedClass.newInstance())}")
复制代码
```

看一下执行结果 生效的

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1d51034ac1374627b97c6ea9d71e16d2~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)