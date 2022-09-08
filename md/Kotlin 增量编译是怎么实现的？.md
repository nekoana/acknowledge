> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7137089121989689351#comment)

前言
--

编译运行是一个`Android`开发者每天都要做的工作，增量编译对于开发者也极其重要，高命中率的增量编译可以极大的提高开发者的开发效率与体验

之前写了一些文章介绍`Kotlin`增量编译的原理，以及`Kotlin 1.7`支持了跨模块增量编译

了解了这些基本原理之后，我们今天一起来看下`Kotlin`增量编译的源码，看看`Kotlin`增量编译到底是怎么实现的

前置知识
----

[Kotlin 快速编译背后的黑科技，了解一下~](https://juejin.cn/post/7118908219841314823 "https://juejin.cn/post/7118908219841314823")  
[Kotlin 1.7 新特性：支持跨模块增量编译](https://juejin.cn/post/7126698891692474375 "https://juejin.cn/post/7126698891692474375")  
[Transform 被废弃，TransformAction 了解一下~](https://juejin.cn/post/7131889789787176974 "https://juejin.cn/post/7131889789787176974")

主要是`Kotlin`增量编译的原理介绍，以及因为在源码中使用了`TransformAction`，也需要了解一下`TransformAction`的基本使用

增量编译流程
------

### 第一步：编译入口

如果我们要在项目中使用`Kotlin`，都必须要添加`org.jetbrains.kotlin.android`插件，这个插件是我们编译`Kotlin`的入口，它的代码在`kotlin-gradle-plugin`插件中

这个插件的实现类就是`KotlinAndroidPluginWrapper`，可以看出`KotlinAndroidPluginWrapper`就是个包装，里面主要就是创建并配置`KotlinAndroidPlugin`

### 第二步：配置`KotlinAndroidPlugin`

`KotlinAndroidPlugin`是插件真正的入口，在这里完成`compileKotlin Task`相关的配置工作

```
internal open class KotlinAndroidPlugin(
    private val registry: ToolingModelBuilderRegistry
) : Plugin<Project> {

    override fun apply(project: Project) {
        checkGradleCompatibility()

        project.dynamicallyApplyWhenAndroidPluginIsApplied() 
    }

    private fun preprocessVariant(
        variantData: BaseVariant,
        compilation: KotlinJvmAndroidCompilation,
        project: Project,
        rootKotlinOptions: KotlinJvmOptionsImpl,
        tasksProvider: KotlinTasksProvider
    ) {
        val configAction = KotlinCompileConfig(compilation)
        configAction.configureTask { task ->
            task.useModuleDetection.value(true).disallowChanges()
            // 将kotlin 编译结果存储在tmp/kotlin-classes/$variantDataName目录下，会作为java compiler的class-path输入
            task.destinationDirectory.set(project.layout.buildDirectory.dir("tmp/kotlin-classes/$variantDataName"))
        }
        tasksProvider.registerKotlinJVMTask(project, compilation.compileKotlinTaskName, compilation.kotlinOptions, configAction)
    }
}
复制代码
```

省略了一些代码，主要做了几件事：

1.  检查`KGP`与`Gradle`的版本兼容，如果不兼容则抛出异常，中止构建
2.  如果在`project`中已经添加了`android`插件，则开始配置`kotlin-android`插件
3.  通过`KotlinCompileConfig`来配置`KotlinCompile Task`，设置`destinationDirectory`作为`Kotlin`编译结果存储目录，后续会作为`java compiler`的`classpath`输入

### 第三步：配置`KotlinCompile`的输入输出

要实现增量编译，最重要的一点就是配置输入输出，当输入输出没有发生变化时，`Task`就可以被跳过，而`KotlinCompile`输入输出的配置，主要是在`KotlinCompileConfig`中完成的

```
configureTaskProvider { taskProvider ->
	// 是否开启classpathSnapthot
    val useClasspathSnapshot = propertiesProvider.useClasspathSnapshot
    val classpathConfiguration = if (useClasspathSnapshot) {
    	// 注册 Transform
        registerTransformsOnce(project)
        project.configurations.detachedConfiguration(
            project.dependencies.create(objectFactory.fileCollection().from(project.provider { taskProvider.get().libraries }))
        )
    } else null

    taskProvider.configure { task ->
    	// 配置输入属性
        task.classpathSnapshotProperties.useClasspathSnapshot.value(useClasspathSnapshot).disallowChanges()
        if (useClasspathSnapshot) {
        	// 通过TransformAction读取输入
            val classpathEntrySnapshotFiles = classpathConfiguration!!.incoming.artifactView {
                it.attributes.attribute(ARTIFACT_TYPE_ATTRIBUTE, CLASSPATH_ENTRY_SNAPSHOT_ARTIFACT_TYPE)
            }.files
            task.classpathSnapshotProperties.classpathSnapshot.from(classpathEntrySnapshotFiles).disallowChanges()
            task.classpathSnapshotProperties.classpathSnapshotDir.value(getClasspathSnapshotDir(task)).disallowChanges()
        } else {
            task.classpathSnapshotProperties.classpath.from(task.project.provider { task.libraries }).disallowChanges()
        }
    }
}
复制代码
```

可以看出，主要做了这么几件事

1.  判断是否开启了`classpathSnapthot`，这也是支持跨模块增量编译的开关，如果开启了就注册`Transform`
2.  通过`TransformAction`获取输入，并配置给`Task`相应的属性

下面我们着重来看下`TransformAction`在这里做了什么工作?

### 第四步：跨模块增量编译支持

```
private fun registerTransformsOnce(project: Project) {
    val buildMetricsReporterService = BuildMetricsReporterService.registerIfAbsent(project)
    project.dependencies.registerTransform(ClasspathEntrySnapshotTransform::class.java) {
        it.from.attribute(ARTIFACT_TYPE_ATTRIBUTE, JAR_ARTIFACT_TYPE)
        it.to.attribute(ARTIFACT_TYPE_ATTRIBUTE, CLASSPATH_ENTRY_SNAPSHOT_ARTIFACT_TYPE)
    }
    project.dependencies.registerTransform(ClasspathEntrySnapshotTransform::class.java) {
        it.from.attribute(ARTIFACT_TYPE_ATTRIBUTE, DIRECTORY_ARTIFACT_TYPE)
        it.to.attribute(ARTIFACT_TYPE_ATTRIBUTE, CLASSPATH_ENTRY_SNAPSHOT_ARTIFACT_TYPE)
    }
}
复制代码
```

了解了前置知识中的`TransformAction`，可以看出这就是注册了只变换`ArtifactType`的变换，主要涉及`JAR_ARTIFACT_TYPE`和`DIRECTORY_ARTIFACT_TYPE`转换为`CLASSPATH_ENTRY_SNAPSHOT_ARTIFACT_TYPE`

也就是说依赖的`jar`和类目录都会转换为`CLASSPATH_ENTRY_SNAPSHOT_ARTIFACT_TYPE`类型，也就可以获取我们依赖的所有`classpath`的`abi`了

接下来我们看下`ClasspathEntrySnapshotTransform`的实现

#### `ClasspathEntrySnapshotTransform`实现

```
abstract class ClasspathEntrySnapshotTransform : TransformAction<ClasspathEntrySnapshotTransform.Parameters> {
    @get:Classpath
    @get:InputArtifact
    abstract val inputArtifact: Provider<FileSystemLocation>

    override fun transform(outputs: TransformOutputs) {
        val classpathEntryInputDirOrJar = inputArtifact.get().asFile
        val snapshotOutputFile = outputs.file(classpathEntryInputDirOrJar.name.replace('.', '_') + "-snapshot.bin")

        val granularity = getClassSnapshotGranularity(classpathEntryInputDirOrJar, parameters.gradleUserHomeDir.get().asFile)

		 val snapshot = ClasspathEntrySnapshotter.snapshot(classpathEntryInputDirOrJar, granularity, metrics)
         ClasspathEntrySnapshotExternalizer.saveToFile(snapshotOutputFile, snapshot)
        
    }

    /**
    * 如果是anroid.jar或者aar依赖，粒度为class, 否则为class_member_level 
    /
    private fun getClassSnapshotGranularity(classpathEntryDirOrJar: File, gradleUserHomeDir: File): ClassSnapshotGranularity {
        return if (
            classpathEntryDirOrJar.startsWith(gradleUserHomeDir) ||
            classpathEntryDirOrJar.name == "android.jar"
        ) CLASS_LEVEL
        else CLASS_MEMBER_LEVEL
    }
}
复制代码
```

关于自定义`TransformAction`，其实跟`Task`一样，也主要看 3 个部分，输入，输出，执行方法体

1.  `ClasspathEntrySnapshotTransform`的输入就是模块依赖的`jar`或者文件目录
2.  输出则是以`-snapshot.bin`结尾的文件
3.  方法体只做了一件事，通过`ClasspathEntrySnapshotter`计算出`claspath`的快照并保存，如果是`aar`依赖，计算的粒度为`class`，如果是项目内的类，计算的粒度是`class_member_level`

`ClasspathEntrySnapshotter`内部是如何计算`classpath`快照的我们这就不看了，我们简单看下下面这样一个类计算的快照是怎样的

```
class MyTest {
    fun startTest(text: String) {
        println(text)
        test1(1)
    }

    private fun test1(index: Int) {
        println("here test126$index")
    }
}
复制代码
```

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/69442ee62a8d4111815345560c959e20~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

`MyTest`类计算出来的快照如图所示，主要`classId`,`classAbiHash`,`classHeaderStrings`等内容

可以看出`private`函数的声明也是`abi`的一部分，当`public`或者`private`的函数声明发生变化时，`classAbiHash`都会发生变化，而只修改函数体时，`snapshot`不会发生任何变化。

### 第五步：`KotlinCompile Task`执行编译

在配置完成之后，接下来我们就来看下`KotlinCompile`是怎么执行编译的

```
abstract class KotlinCompile @Inject constructor(
    override val kotlinOptions: KotlinJvmOptions,
    workerExecutor: WorkerExecutor,
    private val objectFactory: ObjectFactory
) : AbstractKotlinCompile<K2JVMCompilerArguments>(objectFactory {

	// classpathSnapshot入参
    @get:Nested
    abstract val classpathSnapshotProperties: ClasspathSnapshotProperties

    abstract class ClasspathSnapshotProperties {
        @get:Classpath
        @get:Incremental
        @get:Optional // Set if useClasspathSnapshot == true
        abstract val classpathSnapshot: ConfigurableFileCollection
    }

    // 增量编译参数
    override val incrementalProps: List<FileCollection>
        get() = listOf(
            sources,
            javaSources,
            classpathSnapshotProperties.classpathSnapshot
        )

    override fun callCompilerAsync(inputChanges: InputChanges) {
    	// 获取增量编译环境变量
        val icEnv = if (isIncrementalCompilationEnabled()) {
            IncrementalCompilationEnvironment(
                changedFiles = getChangedFiles(inputChanges, incrementalProps),
                classpathChanges = getClasspathChanges(inputChanges),
            )
        } else null
        val environment = GradleCompilerEnvironment(incrementalCompilationEnvironment = icEnv)
        compilerRunner.runJvmCompilerAsync(
            (kotlinSources + scriptSources).toList(),
            commonSourceSet.toList(),
            javaSources.files,
            environment,
        )
    }

    // 查找改动了的input
    protected fun getChangedFiles(
        inputChanges: InputChanges,
        incrementalProps: List<FileCollection>
    ) = if (!inputChanges.isIncremental) {
        ChangedFiles.Unknown()
    } else {
        incrementalProps
            .fold(mutableListOf<File>() to mutableListOf<File>()) { (modified, removed), prop ->
                inputChanges.getFileChanges(prop).forEach {
                    when (it.changeType) {
                        ChangeType.ADDED, ChangeType.MODIFIED -> modified.add(it.file)
                        ChangeType.REMOVED -> removed.add(it.file)
                        else -> Unit
                    }
                }
                modified to removed
            }
            .run {
                ChangedFiles.Known(first, second)
            }
    }

    // 查找改变了的classpath
    private fun getClasspathChanges(inputChanges: InputChanges): ClasspathChanges = when {
        !classpathSnapshotProperties.useClasspathSnapshot.get() -> ClasspathSnapshotDisabled
        else -> {
            when {
                !inputChanges.isIncremental -> NotAvailableForNonIncrementalRun(classpathSnapshotFiles)
                inputChanges.getFileChanges(classpathSnapshotProperties.classpathSnapshot).none() -> NoChanges(classpathSnapshotFiles)
                !classpathSnapshotFiles.shrunkPreviousClasspathSnapshotFile.exists() -> {
                    NotAvailableDueToMissingClasspathSnapshot(classpathSnapshotFiles)
                }
                else -> ToBeComputedByIncrementalCompiler(classpathSnapshotFiles)
            }
        }
    }
}
复制代码
```

对于`KotlinCompile`，我们也可以从入参，出参，`TaskAction`的角度来分析

1.  `classpathSnapshotProperties`是个包装类型的输入，内部包括`@Classpath`类型的输入，使用`@Classpath`输入时，如果输入文件名发生变化而内容没有发生变化时，不会触发`Task`重新运行，这对`classpath`来说非常重要
2.  `incrementalProps`是组件后的增量编译输入参数，包括`kotlin`输入，`java`输入，`classpath`输入等
3.  `CompileKotlin`的`TaskAction`，它最后会执行到`callCompilerAsync`方法，在其中通过`getChangedFiles`与`getClasspathChanges`获取改变了的输入与`classpath`
4.  `getClasspathChanges`方法通过`inputChanges`获取一个已经改变与删除的文件的`Pair`
5.  `getClasspathChanges`则根据增量编译是否开启，是否有文件发生更改，历史`snapshotFile`是否存在，返回不同的`ClassPathChanges`密封类

在增量编译参数拼装完成后，接下来就是跟着逻辑走，最后会走到`GradleKotlinCompilerWork` 的 `compileWithDaemmonOrFailbackImpl`

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a20c41dfc3f94bc5993d8a0e3d20d203~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

```
private fun compileWithDaemonOrFallbackImpl(messageCollector: MessageCollector): ExitCode {
  val executionStrategy = kotlinCompilerExecutionStrategy()
  if (executionStrategy == DAEMON_EXECUTION_STRATEGY) {
    val daemonExitCode = compileWithDaemon(messageCollector)
    if (daemonExitCode != null) {
      return daemonExitCode
    }
  }
  val isGradleDaemonUsed = System.getProperty("org.gradle.daemon")?.let(String::toBoolean)
  return if (executionStrategy == IN_PROCESS_EXECUTION_STRATEGY || isGradleDaemonUsed == false) {
    compileInProcess(messageCollector)
   } else {
    compileOutOfProcess()
   }
}
复制代码
```

可以看出，`kotlin`编译有三种策略，分别是

1.  守护进程编译：`Kotlin`编译的默认模式，只有这种模式才支持增量编译，可以在多个`Gradle daemon`进程间共享
2.  进程内编译：`Gradle daemon`进程内编译
3.  进程外编译：每次编译都是在不同的进程

`compileWithDaemon` 会调用到 `Kotlin Compile` 里执行真正的编译逻辑：

```
val exitCode = try {
  val res = if (isIncremental) {
    incrementalCompilationWithDaemon(daemon, sessionId, targetPlatform, bufferingMessageCollector)
  } else {
    nonIncrementalCompilationWithDaemon(daemon, sessionId, targetPlatform, bufferingMessageCollector)
  }
} catch (e: Throwable) {
    null
}
复制代码
```

到这里会执行 `org.jetbrains.kotlin.daemon.CompileServiceImpl` 的 `compile` 方法，这样就终于调到了`Kotlin`编译器内部

### 第六步：`Kotlin` 编译器计算出需重编译的文件

经过这么多步骤，终于走到`Kotlin`编译器内部了，下面我们来看下`Kotlin`编译器的增量编译逻辑

```
protected inline fun <ServicesFacadeT, JpsServicesFacadeT, CompilationResultsT> compileImpl(){
	//...
	CompilerMode.INCREMENTAL_COMPILER -> {
	    when (targetPlatform) {
	        CompileService.TargetPlatform.JVM -> withIC(k2PlatformArgs) {
	            doCompile(sessionId, daemonReporter, tracer = null) { _, _ ->
	                execIncrementalCompiler(
	                    k2PlatformArgs as K2JVMCompilerArguments,
	                    gradleIncrementalArgs,
	                    //...
	                )
	            }
        }	
}
复制代码
```

如上代码，会判断输入的编译参数，如果是增量编译并且是`JVM`平台的话，就会执行`execIncrementalCompiler`方法，最后会调用到`sourcesToCompile`方法

```
private fun sourcesToCompile(
    caches: CacheManager,
    changedFiles: ChangedFiles,
    args: Args,
    messageCollector: MessageCollector,
    dependenciesAbiSnapshots: Map<String, AbiSnapshot>
): CompilationMode =
    when (changedFiles) {
        is ChangedFiles.Known -> calculateSourcesToCompile(caches, changedFiles, args, messageCollector, dependenciesAbiSnapshots)
        is ChangedFiles.Unknown -> CompilationMode.Rebuild(BuildAttribute.UNKNOWN_CHANGES_IN_GRADLE_INPUTS)
        is ChangedFiles.Dependencies -> error("Unexpected ChangedFiles type (ChangedFiles.Dependencies)")
    }

private fun calculateSourcesToCompileImpl(
        caches: IncrementalJvmCachesManager,
        changedFiles: ChangedFiles.Known,
        args: K2JVMCompilerArguments,
        abiSnapshots: Map<String, AbiSnapshot> = HashMap(),
        withAbiSnapshot: Boolean
    ): CompilationMode {
      	val dirtyFiles = DirtyFilesContainer(caches, reporter, kotlinSourceFilesExtensions)
      	// 初始化dirtyFiles
        initDirtyFiles(dirtyFiles, changedFiles)

    	// 计算变化的classpath
        val classpathChanges = when (classpathChanges) {
            is NoChanges -> ChangesEither.Known(emptySet(), emptySet())
            //  classpathSnapshot可用时
            is ToBeComputedByIncrementalCompiler -> reporter.measure(BuildTime.COMPUTE_CLASSPATH_CHANGES) {
                computeClasspathChanges(
                    classpathChanges.classpathSnapshotFiles,
                    caches.lookupCache,
                    storeCurrentClasspathSnapshotForReuse,
                    ClasspathSnapshotBuildReporter(reporter)
                ).toChangesEither()
            }
            is NotAvailableDueToMissingClasspathSnapshot -> ChangesEither.Unknown(BuildAttribute.CLASSPATH_SNAPSHOT_NOT_FOUND)
            is NotAvailableForNonIncrementalRun -> ChangesEither.Unknown(BuildAttribute.UNKNOWN_CHANGES_IN_GRADLE_INPUTS)
            // classpathSnapshot不可用时
            is ClasspathSnapshotDisabled -> reporter.measure(BuildTime.IC_ANALYZE_CHANGES_IN_DEPENDENCIES) {
                val lastBuildInfo = BuildInfo.read(lastBuildInfoFile)   
                getClasspathChanges(
                    args.classpathAsList, changedFiles, lastBuildInfo, modulesApiHistory, reporter, abiSnapshots, withAbiSnapshot,
                    caches.platformCache, scopes
                )
            }
            is NotAvailableForJSCompiler -> error("Unexpected type for this code path: ${classpathChanges.javaClass.name}.")
        }
        // 将结果添加到dirtyFiles
        val unused = when (classpathChanges) {
            is ChangesEither.Unknown -> {
                return CompilationMode.Rebuild(classpathChanges.reason)
            }
            is ChangesEither.Known -> {
                dirtyFiles.addByDirtySymbols(classpathChanges.lookupSymbols)
                dirtyClasspathChanges = classpathChanges.fqNames
                dirtyFiles.addByDirtyClasses(classpathChanges.fqNames)
            }
        }

        // ...
        return CompilationMode.Incremental(dirtyFiles)
    }    
复制代码
```

`calculateSourcesToCompileImpl`的目的就是计算`Kotlin`编译器应该重新编译哪些代码，主要分为以下几个步骤

1.  初始化`dirtyFiles`，并将`changedFiles`加入`dirtyFiles`，因为`changedFiles`需要重新编译
2.  `classpathSnapshot`可用时，通过传入的`snapshot.bin`文件，与`Project`目录下的`shrunk-classpath-snapshot.bin`进行比较得出变化的`classpath`，以及受影响的类。在比较结束时，也会更新当前目录的`shrunk-classpath-snapshot.bin`，供下次比较使用
3.  当`classpathSnapshot`不可用时，通过`getClasspathChanges`方法来判断`classpath`变化，这里面实际上是通过`last-build.bin`与`build-history.bin`来判断的，同时每次编译完成也会更新`build-history.bin`
4.  将受`classpath`变化影响的类也加入`dirtyFiles`
5.  返回`dirtyFiles`供`Kotlin`编译器真正开始编译

在这一步，`Kotlin`编译器利用输入的各种参数进行分析，将需要重新编译的文件加入`dirtyFiles`，供下一步使用

### 第七步：`Kotlin`编译器真正开始编译

```
private fun compileImpl(): ExitCode {
    // ...
    var compilationMode = sourcesToCompile(caches, changedFiles, args, messageCollector, classpathAbiSnapshot)
    when (compilationMode) {
        is CompilationMode.Incremental -> {
            // ...
            compileIncrementally(args, caches, allSourceFiles, compilationMode, messageCollector, withAbiSnapshot)
        }
        is CompilationMode.Rebuild -> rebuildReason = compilationMode.reason
    }
    // ...
}

protected open fun compileIncrementally(): ExitCode {
   while (dirtySources.any() || runWithNoDirtyKotlinSources(caches)) {
        // ...
        val (sourcesToCompile, removedKotlinSources) = dirtySources.partition(File::exists)
        // 真正进行编译
        val compiledSources = runCompiler(
            sourcesToCompile, args, caches, services, messageCollectorAdapter,
            allKotlinSources, compilationMode is CompilationMode.Incremental
        )
        // ...
    }    

    if (exitCode == ExitCode.OK) {
        // 写入`last-build.bin`
        BuildInfo.write(currentBuildInfo, lastBuildInfoFile)
    }

    val dirtyData = DirtyData(buildDirtyLookupSymbols, buildDirtyFqNames)
    // 写入`build-history.bin`
    processChangesAfterBuild(compilationMode, currentBuildInfo, dirtyData)

    return exitCode
}
复制代码
```

这段代码主要做了这么几件事：

1.  通过`sourcesToCompile`计算出发生改变的文件后，如果可以增量编译，则进入到`compileIncrementally`
2.  从`dirtySouces`中找出需要重新编译的文件，交给`runCompiler`方法进行真正的编译
3.  在编译结束之后，写入`last-build.bin`与`build-history.bin`文件，供下次编译时对比使用

到这里，增量编译的流程也就基本完成了。

总结
--

本文较为详细地介绍了`Kotin`是怎么一步步从编译入口到真正开始增量编译的，了解`Kotlin`增量编译原理可以帮助你定位为什么`Kotlin`增量编译有时会失效，也可以了解如何写出更容易命中增量编译的代码，希望对你有所帮助。

关于`Kotlin`增量编译还有更多的细节，本文也只是介绍了主要的流程，感兴趣的同学可直接查看`KGP`和`Kotlin`编译器的源码

### 参考资料

[深入研究 Android 编译流程 - Kotlin 是如何编译的](https://juejin.cn/post/7087832824501239821 "https://juejin.cn/post/7087832824501239821")