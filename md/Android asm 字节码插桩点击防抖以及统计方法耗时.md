> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7187664156995584059)

1、目标
====

使用 asm 字节码插桩的方式，实现给点击事件加上**防抖**和**统计方法耗时**的功能

2、api 介绍
========

### 1、Transform API

Transform API 是 AGP1.5 就引入的特性，Android 在构建过程中回将 Class 转成 Dex，此 API 就是提供了在此过程中插入自定逻辑字节码的功能，我们可以使用此 API 做一些功能，比如无痕埋点，耗时统计等功能。不过此 API 在 AGP7.0 已经被废弃，8.0 会被移除，取而代之的是 Transform Action

### 2、Transform Action

Transform Action 是有 Gradle 提供的，直接使用 Transform Action 会有点麻烦，AGP 为我们封装了一层 AsmClassVisitorFactory，我们一般可以使使用 AsmClassVisitorFactory，这样代码量会减少，而且性能还有提升。简单使用的话，整体流程跟 Transform API 差不多。

然后我们知道 ASM 有两套 API，core api 和 tree api([blog.51cto.com/lsieun/4088…](https://link.juejin.cn?target=https%3A%2F%2Fblog.51cto.com%2Flsieun%2F4088765 "https://blog.51cto.com/lsieun/4088765"))，具体区别可以看下链接，tree api 使用会更方便一些，实现一些功能会更简单，不过性能上会比 core api 差一些。因为 Transform API 废弃了，所以接下来都是以 Transform Action 为例子。

3、实现方案
======

我们使用 plugin 的方式编写插桩代码，然后将它 publish 到本地，然后在对应工程引用这个 plugin

1、新建 plugin
-----------

这个网上有很多资料，可自行查找，就是配置 **resources** 目录，新建 **.properties** 文件，在 build.gradle 中配置 **publishing{}** 即可。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9b49ef9a851d474cbd484145617f8a72~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

moudle 的 build.gradle 文件中添加一下

```
group "com.trans.test.plugin"
version "1.0.0"

publishing{ //当前项目可以发布到本地文件夹中
    repositories {
        maven {
            url= '../repo' //定义本地maven仓库的地址
        }
    }

    publications {
        PublishAndroidAssetLibrary(MavenPublication) {
            groupId group
            artifactId artifactId
            version version
        }
    }
}
复制代码
```

**当修改 plugin 的代码后，记得 publish 一下，以更新下本地库**

使用的 module 在 build.gradle 引用一下

```
apply plugin: com.example.transformaction.AsmPlugin
复制代码
```

**根目录的 setting.grdle 中导入本地路径**

```
maven { url('./repo') }
复制代码
```

2、编写插桩代码，这里叙述一下大概的逻辑
--------------------

#### 1、过滤所有需要的方法

1、正常的点击 setOnclickListener(), 页面实现 OnclickListener 接口，重写 onClick() 方法

2、匿名内部类 setOnclickListener()

3、xml 点击事件

4、ButterKnife 点击事件

#### 2、对方法进行 hook 插桩

基本逻辑就是，我们用 kotlin 实现一个 “防抖” 的功能，然后将这个功能的调用代码以字节码的方式插入到需要 hook 的方法中（具体实现下面会说明）

4、具体实现步骤
========

1、先实现一个 Plugin
--------------

```
class AsmPlugin : Plugin<Project> {
    override fun apply(project: Project) {
        println("我是插件")
        val androidComponents = project.extensions.getByType(AndroidComponentsExtension::class.java)
        androidComponents.onVariants { variant ->
            variant.instrumentation.transformClassesWith(
                MyTestTransform::class.java,
                InstrumentationScope.PROJECT) {params->
                params.config.set(ViewDoubleClickConfig())
            }
            variant.instrumentation.setAsmFramesComputationMode(
                FramesComputationMode.COMPUTE_FRAMES_FOR_INSTRUMENTED_METHODS
            )
        }
        println("插件插入完成")
    }
}
复制代码
```

从代码可以看出，（Transform API 中我们使用 AppExtension）Transform Action 使用 **AndroidComponentsExtension** 来获取组件，然后一次插入我们班自定义的 MyTestTransform 来插入我们的字节码。

2、实现 MyTestTransform
--------------------

```
interface DoubleClickParameters : InstrumentationParameters {
    @get:Input
    val config: Property<ViewDoubleClickConfig>
}


abstract class MyTestTransform: AsmClassVisitorFactory<DoubleClickParameters> {
    override fun createClassVisitor(classContext: ClassContext, nextClassVisitor: ClassVisitor): ClassVisitor {
        return TreeTestVisitor(
           nextClassVisitor = nextClassVisitor,
           config = parameters.get().config.get()
       )
        // return CoreClassVisitor(nextClassVisitor)

    }

    override fun isInstrumentable(classData: ClassData): Boolean {
        return true
    }
}
复制代码
```

DoubleClickParameters 是用来传参的，TreeTestVisitor 使用了传参的方式，所以使用 AsmClassVisitorFactory 泛型用了 DoubleClickParameters。以上代码可以看出 MyTestTransform 内部 createClassVisitor 需要返回一个 ClassVisitor，我们用两种实现方式（core api 和 tree api ）来演示下。

3、TreeTestVisitor（tree api）
---------------------------

```
class TreeTestVisitor(
    private val nextClassVisitor: ClassVisitor,
    private val config: ViewDoubleClickConfig
) : ClassNode(Opcodes.ASM5) {

    private val extraHookPoints = listOf(
        ViewDoubleClickHookPoint(
            interfaceName = "android/view/View$OnClickListener",
            methodName = "onClick",
            nameWithDesc = "onClick(Landroid/view/View;)V"
        ),
        ViewDoubleClickHookPoint(
            interfaceName = "com/chad/library/adapter/base/listener/OnItemClickListener",
            methodName = "onItemClick",
            nameWithDesc = "onItemClick(Lcom/chad/library/adapter/base/BaseQuickAdapter;Landroid/view/View;I)V"
        ),
        ViewDoubleClickHookPoint(
            interfaceName = "com/chad/library/adapter/base/listener/OnItemChildClickListener",
            methodName = "onItemChildClick",
            nameWithDesc = "onItemChildClick(Lcom/chad/library/adapter/base/BaseQuickAdapter;Landroid/view/View;I)V",
        )
)
    
    override fun visitEnd(){
         // 这里就是遍历methods 对方法一一进行Visitor
         // 这里我们要做的就是取出所有的onClick时事件然后一一插入对应的字节码
         // 点击事件有几种情况
         // 1、正常的点击setOnclickListener(),页面实现OnclickListener接口，重写onClick()方法
         // 2、匿名内部类setOnclickListener()
         // 3、xml点击事件
         // 4、ButterKnife点击事件

        // 以上其实可以分为三类 
        // 1、使用注解，判断注解  hasAnnotation（）
        // 2、通过MethodNode的interfaces数组，来判断是实现onClickLitener接口（包括列表的onItemClickLisener等）
        // 3、lambda表达式的方式
        
        super.visitEnd()
        val shouldHookMethodList = mutableSetOf<MethodNode>()
        methods.forEach { methodNode ->

            //使用了 ViewAnnotationOnClick 自定义注解的情况
            methodNode.hasAnnotation("Lcom/example/transformsaction/annotation/ViewAnnotationOnClick;") -> {
                shouldHookMethodList.add(methodNode)
            }

            //使用了 Butterknife 注解的情况
            methodNode.hasAnnotation("Lbutterknife/OnClick;") -> {
                shouldHookMethodList.add(methodNode)
            }

            //使用了匿名内部类的情况
            methodNode.isHookPoint() -> {
                shouldHookMethodList.add(methodNode)
            }

            //判断方法内部是否有需要处理的 lambda 表达式
            val dynamicInsnNodes = methodNode.filterLambda {
                val nodeName = it.name
                val nodeDesc = it.desc
                val find = extraHookMethodList.find { point ->
                    nodeName == point.methodName && nodeDesc.endsWith(point.interfaceSignSuffix)
                }
                find != null
            }
            dynamicInsnNodes.forEach {
                val handle = it.bsmArgs[1] as? Handle
                if (handle != null) {
                    //找到 lambda 指向的目标方法
                    val nameWithDesc = handle.name + handle.desc
                    val method = methods.find { it.nameWithDesc == nameWithDesc }!!
                    shouldHookMethodList.add(method)
                }
            }    
        }

        shouldHookMethodList.forEach {
            hookMethod(modeNode = it)
        }
        accept(nextClassVisitor)
    }

    // MethodNode的拓展方法，判断注解
    // 举个栗子：我们新定义了一个注解 ViewAnnotationOnClick
    // annotationDesc就对应 ViewAnnotationOnClick的全限定路径，
    // 即："Lcom/example/transformsaction/annotation/ViewAnnotationOnClick;"
    fun MethodNode.hasAnnotation(annotationDesc: String): Boolean {
    	return visibleAnnotations?.find { it.desc == annotationDesc } != null
	}

    // MethodNode的拓展方法，匿名内部类
    private fun MethodNode.isHookPoint(): Boolean {
        val myInterfaces = interfaces
        if (myInterfaces.isNullOrEmpty()) {
            return false
        }
        extraHookMethodList.forEach {
            if (myInterfaces.contains(it.interfaceName) && this.nameWithDesc == it.nameWithDesc) {
                return true
            }
        }
        return false
    }

    // MethodNode的拓展方法，lambda表达式
    fun MethodNode.filterLambda(filter: (InvokeDynamicInsnNode) -> Boolean): List<InvokeDynamicInsnNode> {
        val mInstructions = instructions ?: return emptyList()
        val dynamicList = mutableListOf<InvokeDynamicInsnNode>()
        mInstructions.forEach { instruction ->
            if (instruction is InvokeDynamicInsnNode) {
                if (filter(instruction)) {
                    dynamicList.add(instruction)
                }
            }
        }
        return dynamicList
	}

    // 给过滤后的方法插入字节码
    private fun hookMethod(modeNode: MethodNode) {
        // 取出描述
        val argumentTypes = Type.getArgumentTypes(modeNode.desc)
        // 得出对应描述类型在该方法参数中的位置 
        //（主要是新建ViewDoubleClickCheck。onClick(view:View)有个入参，要取被hook函数的参数传入hook方法中）
        val viewArgumentIndex = argumentTypes?.indexOfFirst {
            it.descriptor == ViewDescriptor
        } ?: -1
        if (viewArgumentIndex >= 0) {
            val instructions = modeNode.instructions
            if (instructions != null && instructions.size() > 0) {

                // 插入防抖的字节码
                val listCheck = InsnList()
                // 得出入参要取被hook函数的位置
                val index =  getVisitPosition(
                    argumentTypes,
                    viewArgumentIndex,
                    modeNode.isStatic
                )

                //参数
                listCheck.add(
                    VarInsnNode(
                        Opcodes.ALOAD, index
                    )
                )
                // 插入ViewDoubleClickCheck的调用函数的字节码
                listCheck.add(
                    MethodInsnNode(
                        Opcodes.INVOKESTATIC,
                        "com/example/transformsaction/view/ViewDoubleClickCheck",
                        "onClick",
                        "(Landroid/view/View;)Z"
                    )
                )
                // 因为是插入的字节码为判断语句，不满足的需要return
                val labelNode = LabelNode()
                listCheck.add(JumpInsnNode(Opcodes.IFNE, labelNode))
                listCheck.add(InsnNode(Opcodes.RETURN))
                listCheck.add(labelNode)

                //将新建的字节码插入instructions中
                instructions.insert(listCheck)



                // 目的是在方法末尾插入字节码
                for( node in instructions){
                    //判断是不是方法结尾的AbstractInsnNode
                    if(node.opcode == Opcodes.ARETURN || node.opcode == Opcodes.RETURN){
                        System.out.println("找到了")

                        // 创建字节码容器
                        val listEnd = InsnList()

                        // 字节码方法参数
                        listEnd.add(
                            VarInsnNode(
                                Opcodes.ALOAD, index
                            )
                        )
                        // 插入ToastClick.endClick()
                        listEnd.add(
                            MethodInsnNode(
                                Opcodes.INVOKESTATIC,
                                "com/example/transformsaction/view/ToastClick",
                                "endClick",
                                "()V"
                            )
                        )

                        // 将字节码插入到结尾node之前，使用insertBefore
                        instructions.insertBefore(node,listEnd)
                    }

                }


                // 在方法开始插入字节码
                val list = InsnList()

                list.add(
                    VarInsnNode(
                        Opcodes.ALOAD, index
                    )
                )
                // 插入ToastClick.startClick()
                list.add(
                    MethodInsnNode(
                        Opcodes.INVOKESTATIC,
                        "com/example/transformsaction/view/ToastClick",
                        "startClick",
                       "()V"
                    )
                )
                instructions.insert(list)
            }
        }
    }
    
}
复制代码
```

#### **InsnList**

插入字节码的时候是对过滤后的 shouldHookMethodList 一一进行字节码插入，也就是调用 InsnList 的 insert 方法，简单说下 InsnList，InsnList 提供许多插入字节码的方法：

```
add(final AbstractInsnNode insnNode)
末尾插入一个AbstractInsnNode

add(final InsnList insnList)
末尾插入一组InsnList

insert(final AbstractInsnNode insnNode)
头部插入一个AbstractInsnNode

insert(final InsnList insnList)
头部插入一组InsnList

insert(final AbstractInsnNode previousInsn, final AbstractInsnNode insnNode)
在previousInsn后插入一个AbstractInsnNode

insert(final AbstractInsnNode previousInsn, final InsnList insnList)
在previousInsn后插入一组InsnList

insertBefore(final AbstractInsnNode nextInsn, final AbstractInsnNode insnNode)
在previousInsn前插入一个AbstractInsnNode

insertBefore(final AbstractInsnNode nextInsn, final InsnList insnList)
在previousInsn前插入一组InsnList
复制代码
```

以上只是部分方法，有兴趣的可以去看下源码。

4、CoreClassVisitor (core api)
-----------------------------

```
class CoreClassVisitor(nextVisitor: ClassVisitor) : ClassVisitor(Opcodes.ASM7, nextVisitor) {

    override fun visitMethod(
        access: Int,
        name: String,
        desc: String,
        signature: String?,
        exceptions: Array<out String>?
    ): MethodVisitor {
        val methodVisitor = super.visitMethod(access, name, desc, signature, exceptions)
        return MyClickVisitor(Opcodes.ASM7,methodVisitor,access,name,desc)
    }

}
复制代码
```

就是重写一下 visitMethod 函数，然后将具体的逻辑传入到 MyClickVisitor 去实现，**这里只演示在方法插入防抖的实现**

```
class MyClickVisitor(api: Int, methodVisitor: MethodVisitor?, access: Int, name: String?,
                     val descriptor: String?
) : AdviceAdapter(api, methodVisitor,
    access,
    name, descriptor
) {

    // 注解缓存
    var visibleAnnotations: ArrayList<AnnotationNode>? = null


    // 获取注解 参考了tree api
    override fun visitAnnotation(descriptor: String?, visible: Boolean): AnnotationVisitor? {
        val annotation = AnnotationNode(descriptor)
        if(null == visibleAnnotations){
            visibleAnnotations = ArrayList()
        }
        if (visible) {
            println("添加注解:"+ descriptor)
            visibleAnnotations?.add(annotation)
        }
        return annotation
    }
     
    override fun onMethodEnter() {
        val viewArgumentIndex = argumentTypes?.indexOfFirst {
            it.descriptor == ViewDescriptor
        } ?: -1

        println("打印注解列表长度"+visibleAnnotations?.size)
        // 
        if (matchMethod(name, descriptor) || matchExitMethod()) {
            println("拦截一个")
            mv.visitVarInsn(ALOAD, getVisitPosition(
                argumentTypes,
                viewArgumentIndex,
                access and Opcodes.ACC_STATIC != 0
            ))
            mv.visitMethodInsn(
                INVOKESTATIC,
                "com/example/transformsaction/view/ViewDoubleClickCheck",
                "onClick",
                "(Landroid/view/View;)Z",
                false
            )
            val label0 = Label()
            mv.visitJumpInsn(IFNE, label0)
            mv.visitInsn(RETURN)
            mv.visitLabel(label0)
        }
        super.onMethodEnter()
    }

    private fun matchMethod(name: String, desc: String?): Boolean {
        println("拦截判断$name  $desc")
        return  (name == "onClick" && desc == "(Landroid/view/View;)V")
                || (name == "onItemClick" && desc == "(Lcom/chad/library/adapter/base/BaseQuickAdapter;Landroid/view/View;I)V")
                || (name == "onItemChildClick" && desc == "(Lcom/chad/library/adapter/base/BaseQuickAdapter;Landroid/view/View;I)V")
    }

    private fun matchExitMethod(): Boolean {
        return (hasCheckViewAnnotation() || hasButterKnifeOnClickAnnotation())
    }

    private fun hasCheckViewAnnotation(): Boolean {
        return hasAnnotation("Lcom/example/transformsaction/annotation/ViewAnnotationOnClick;")
    }

    private fun hasButterKnifeOnClickAnnotation(): Boolean {
        return hasAnnotation("Lbutterknife/OnClick;")
    }

    fun hasAnnotation(annotationDesc: String): Boolean {
        var value = visibleAnnotations?.find { it.desc == annotationDesc } != null
        println("判断注解:"+ value)
        return value
    }
}
复制代码
```

实现逻辑基本跟 **TreeTestVisitor** 差不多，无非就是一个是用 InsnList，另一个使用 MethodVisitor 的 api 进行插入。有一点就是 lambda 表达式的 hook, 我没想到好的方法，参考了 tree api，重写 visitInvokeDynamicInsn 方法，调用时就创建一个 InvokeDynamicInsnNode，保存到缓存列表里, 然后通过判断保存的列表是否包含对应的接口以及方法，就可以判断对应的 lambda 是否是目标方法：

```
override fun visitInvokeDynamicInsn(
  name: String?,
  descriptor: String?,
  bootstrapMethodHandle: Handle?,
  vararg bootstrapMethodArguments: Any?
) {
  println("添加lambda:"+ descriptor+"  "+name)
  instructions.add(
      InvokeDynamicInsnNode(
          name, descriptor?.split(")")?.get(1) ?: descriptor, bootstrapMethodHandle, *bootstrapMethodArguments
      )
  )
}
复制代码
```

```
fun filterLambda(filter: (InvokeDynamicInsnNode) -> Boolean): List<InvokeDynamicInsnNode> {
    val mInstructions = instructions
    val dynamicList = mutableListOf<InvokeDynamicInsnNode>()
    mInstructions.forEach { instruction ->
        if (instruction is InvokeDynamicInsnNode) {
            if (filter(instruction)) {
                dynamicList.add(instruction)
            }
        }
    }
    return dynamicList
}
复制代码
```

但是获取到是在 **onMethodEnter** 之后，没找到像 InsnList 一样在各个位置插入字节码的 api，后续再优化吧。

对比一下插入之前和之后的代码吧：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3347f10b52bb4df4a4cb2dd481e07aa7~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/49fdd0b878d142ccbc7f795465095549~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

完美！！！

5、总结
====

基本逻辑是如上述所示，实现字节码插桩，主要考虑两个个问题：

### 1、字节码插桩位置在哪？

就是怎么去过滤对应的方法，使用 tree api 可以通过 MethodNode 内部的变量来过滤，即 visibleAnnotations（注解），interfaces（实现的接口），instructions（可用于判断 lambda 表达式对应的概关键信息，以点击事件为例，lambdab 表达式方法最终会被转成一个静态方法，方法名类似于 “onCreatelambda0”，关键信息会被放在 instructions 中，所以可以通过 instructions 判断。

### 2、怎么把字节码插入进去 ？

就是对 asm API 的调用了，这个慢慢学习吧

6、鸣谢
====

本文是参考了 [juejin.cn/post/704232…](https://link.juejin.cn?target=https%3A%2F%2Flinks.jianshu.com%2Fgo%3Fto%3Dhttps%253A%252F%252Fjuejin.cn%252Fpost%252F7042328862872567838%2523heading-3 "https://links.jianshu.com/go?to=https%3A%2F%2Fjuejin.cn%2Fpost%2F7042328862872567838%23heading-3") ASM 字节码插桩：实现双击防抖，感谢大佬，原文有源码，可自行下载。

我看过后，想对自己理的解做一下梳理，所以才有此文。自己的代码地址 [gitee.com/wlr123/tran…](https://link.juejin.cn?target=https%3A%2F%2Fgitee.com%2Fwlr123%2Ftransforms-action "https://gitee.com/wlr123/transforms-action")