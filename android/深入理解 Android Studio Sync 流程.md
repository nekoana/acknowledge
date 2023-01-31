> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7152769459541770254#heading-9)

1. 初识 Sync
==========

我们一般会把 Sync 理解为 Android Studio 的准备阶段，包括解析工程配置信息、下载远程依赖到本地、更新代码索引等准备工作，当修改 gradle build 文件后，需要重新 Sync 将 Gradle 构建配置信息同步到 IDE，进而使 IDE 的功能及时应用新的构建配置，这些功能包括项目的 Gradle Task 列表展示、依赖信息展示等等。Sync 是 Android Studio 中独有的概念，当通过 Gradle 命令行程序构建 Android 应用时，只会经历 Gradle 定义的 Initialization、Configuration 和 Execution 生命周期，根本没有 Sync 的概念。Android Studio 的 Sync 阶段涉及到 IDE、Gradle、Plugin 等多个角色，梳理清楚这些角色各自的作用和联系，才能清晰理解 Sync 的整体架构，下面分别来介绍这些角色：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cf385dae43a64b68a977247cdc08cab7~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

**IDE** **层面**：Android Studio 基于 IntelliJ IDEA 扩展而来，复用了 IntelliJ IDEA 强大的代码编辑器和开发者工具等 IDE 基础能力，在此之上 Android Studio 提供了更多提高 Android 构建效率的功能，如 Android 模拟器、代码模版等等。另外 Android Studio 也将自身专业的 Android 应用开发能力反哺给 IntelliJ IDEA，以 Android IDEA Plugin 的形式，使 IntelliJ IDEA 支持 Android 应用开发，二者互相赋能，相辅相成。

**Gradle** **层面**：Gradle 是一个灵活而强大的开源构建系统，它除了提供跨平台的可执行程序支持命令行执行 Gradle 构建外，还专门提供了 Gradle Tooling API 编程 SDK，供外部更方便、更紧密的将 Gradle 构建能力嵌入到 IDE 中，IntelliJ IDEA、Eclipse、VSCode 等 IDE 都采用了这种方式。在 Gradle 源码中也有专门服务于 IntelliJ IDEA、Eclipse 等 IDE 的代码模块，构建工具和 IDE 两个角色之间同样是互相赋能，强强联合。

**Plugin 层面**：Plugin 层面包括 Android IDEA Plugin 和 Android Gradle Plugin，Android IDEA Plugin 为 IntelliJ IDEA/Android Studio 拓展了 Android 应用开发能力；Android Gradle Plugin 为 Gradle 拓展了 Android 应用构建能力。谷歌通过这两个 Plugin 将现代成熟优秀的 IDE 开发能力和构建工具联合在一起为 Android 所用，相比于早期 Eclipse 加 ANT 构建的开发方式，大幅提升了 Android 应用开发效率和体验。

2. Sync 流程分析
============

了解了 Sync 阶段涉及到的角色以及它们之间的关系后，接下来深入 Android Studio 源码，从代码层面梳理清楚 Sync 的关键流程。

2.1 Android Studio 源码分析
-----------------------

### 2.1.1 功能入口及准备

在 Android Studio 中触发 Sync 操作后，会从最上层的入口类 GradleSyncInvoker 调用到实际负责解析 Gradle 构建信息的 GradleProjectResolver，调用链如下图所示：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fbe350ac6f3a47c4bcf5b9ce7df9ecd1~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

调用过程中涉及到的关键类：

*   **GradleSyncInvoker**：触发 Sync 的入口类，在 Android Studio 多处需要执行 Sync 的地方，都是通过调用此类的 requestProjectSync 方法来触发的
    
*   **ExternalSystemUtil**：GradleSyncInvoker、GradleSyncExecutor 等类是专门针对 Sync 功能的封装，Sync 是 Android Studio 中独有的操作，在 IntelliJ IDEA 中并没有 Sync 的概念，IntelliJ IDEA 通过 [Reload All Gradle Projects] 操作来触发解析工程的 Gradle 构建信息，直接从 ExternalSystemUtil 类开始执行
    
*   **GradleProjectResolver**：负责具体执行 Sync，其中 resolveProjectInfo 方法是具体执行 Sync 逻辑的地方，该方法的定义如下：
    
    ```
    public DataNode<ProjectData> resolveProjectInfo(
        @NotNull ExternalSystemTaskId id,
        @NotNull String projectPath,
        boolean isPreviewMode,
        @Nullable S settings,
        @NotNull ExternalSystemTaskNotificationListener listener)
      throws ExternalSystemException, IllegalArgumentException, IllegalStateException
    复制代码
    ```
    
*   **id**：本次 Sync 操作的唯一标识，后续可通过调用 GradleProjectResolver 中的 cancelTask 方法取消本次 Sync 任务
    
*   **projectPath**：工程绝对路径
    
*   **settings**：Sync 工程的配置参数，可设置 Java 版本等
    
*   **listener**：用于监听此次 Sync 的过程及结果
    
*   **isPreviewMode**：是否要启用预览模式，Android Studio 首次打开未知来源项目时，会让开发者选择项目的打开方式，如下图，若选择 [Stay in Safe Mode] 则会以 “预览模式” 运行项目，表示 IDE 仅可以浏览项目的源代码，不会执行或解析任何构建任务和脚本
    

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/04838fb584e04b43a8901c77b29f693b~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

进入 GradleProjectResolver 的 resolveProjectInfo 方法中后，首先会对预览模式进行处理，如下代码所示，如果是预览模式，则会简单构造出对应的工程数据结构后立马返回，不进行任何的解析行为：

```
if (isPreviewMode) {
  String projectName = new File(projectPath).getName();
  ProjectData projectData = new ProjectData(GradleConstants.SYSTEM_ID, projectName, projectPath, projectPath);
  DataNode<ProjectData> projectDataNode = new DataNode<>(ProjectKeys.PROJECT, projectData, null);
  ......
  return projectDataNode;
}
复制代码
```

使用 Android Studio/IntelliJ IDEA 打开工程时，除了指定工程所在根目录，还可以指定 Gradle 配置文件，这个逻辑在源码中也有所体现：

```
if (projectPathFile.isFile() && projectPath.endsWith(GradleConstants.EXTENSION) && projectPathFile.getParent() != null) {
  projectDir = projectPathFile.getParent();
  if (settings != null) {
    List<String> arguments = settings.getArguments();
    if (!arguments.contains("-b") && !arguments.contains("--build-file")) {
      settings.withArguments("-b", projectPath);
    }
  }
} else {
  projectDir = projectPath;
}
复制代码
```

可以看到如果打开的不是工程根目录而是配置文件，那就会把该配置文件所在的目录作为工程路径，并且会通过 --build-file 参数来指定配置文件，后续会传入到 Gradle 中。对工程初步处理后，接着通过 Gradle Tooling API 获取工程的 BuildEnvironment：

```
ModelBuilder<BuildEnvironment> modelBuilder = connection.model(BuildEnvironment.class);
复制代码
```

调用 Gradle Tooling API 时其内部会自动下载项目配置版本的 Gradle，也就是说，上面代码中获取 BuildEnvironment Model 的调用本身确保了 Gradle 的下载，即将 /grade/wrapper/gradle-wrapper.properties 里 distributionUrl 指定的 Gradle 下载到 GRADLE_HOME/wrapper/dists 下。

BuildEnvironment 是 Gradle 提供的 Gradle Model，Gradle Model 是一个非常重要的概念，当 IDE 通过 Gradle Tooling API 和 Gradle 交互时，传输的就是各种各样的 Model，比如 Gradle 自带的 GradleProject  _、_  BuildEnvironment 等，另外也可以在 Gradle Plugin 中通过 ToolingModelBuilderRegistry 注册自定义的 Model，如 Android Gradle Plugin 中注册了 AndroidProject Model，那就可以通过 Gradle Tooling API 获取 Android 项目特有的 AndroidProject Model，从而获取由 Android Gradle Plugin 提供的 Android 应用相关工程信息。

### 2.1.2 配置 BuildAction

继续分析 Android Studio Sync 源码，接下来构造了一个 ProjectImportAction，它实现了 Gradle Tooling API 中的 BuildAction 接口，BuildAction 定义如下：

```
/**
* An action that executes against a Gradle build and produces a result of type {  @code  T}.
 *  @param <T> The type of result produced by this action.
*/
public interface BuildAction<T> extends Serializable {

T execute(BuildController controller);
}
复制代码
```

BuildAction 是即将传入到 Gradle 构建进程中执行的行为，并且可将结果数据序列化返回给调用方。这个 BuildAction 至关重要，它是实际和 Gradle 通信的地方，其中实现了组织生成工程信息、下载依赖等功能，是 Sync 流程中的核心逻辑。BuildAction 再配合 Gradle Tooling API 中的 BuildActionExecuter，就可以将 BuildAction 交由 Gradle 触发执行了，在执行之前，需先通过 BuildActionExecuter 配置 JVM 参数、Gradle 命令行参数以及环境变量等构建信息：

```
private static void configureExecutionArgumentsAndVmOptions(@NotNull GradleExecutionSettings executionSettings,
                                                            @NotNull DefaultProjectResolverContext resolverCtx,
                                                            boolean isBuildSrcProject) {
  executionSettings.withArgument("-Didea.sync.active=true");
  if (resolverCtx.isResolveModulePerSourceSet()) {
    executionSettings.withArgument("-Didea.resolveSourceSetDependencies=true");
  }
  if (!isBuildSrcProject) {
    for (GradleBuildParticipant buildParticipant : executionSettings.getExecutionWorkspace().getBuildParticipants()) {
      executionSettings.withArguments(GradleConstants.INCLUDE_BUILD_CMD_OPTION, buildParticipant.getProjectPath());
    }
  }
  GradleImportCustomizer importCustomizer = GradleImportCustomizer.get();
  GradleProjectResolverUtil.createProjectResolvers(resolverCtx).forEachOrdered(extension -> {
    if (importCustomizer == null || importCustomizer.useExtraJvmArgs()) {
      // collect extra JVM arguments provided by gradle project resolver extensions
      ParametersList parametersList = new ParametersList();
      for (Pair<String, String> jvmArg : extension.getExtraJvmArgs()) {
        parametersList.addProperty(jvmArg.first, jvmArg.second);
      }
      executionSettings.withVmOptions(parametersList.getParameters());
    }
    // collect extra command-line arguments
    executionSettings.withArguments(extension.getExtraCommandLineArgs());
  });
}
复制代码
```

上述代码较多，我们先只关注于 GradleExecutionSettings 的 withArgument 和 withVmOptions 方法，它们分别负责收集 Gradle 命令行参数和 JVM 参数。从代码中可以看出，这些 Gradle 命令行参数及 JVM 参数大多数是从 extension 中收集而来，这里的 extension 是指 IntelliJ IDEA Plugin 中的扩展，和扩展点结合使用，可扩展 IntelliJ IDEA 平台特性或其他 IntelliJ IDEA Plugin 的功能特性，举个例子，Gradle IDEA Plugin 提供了工程解析的扩展点：

```
<extensionPoint 
   qualified 
   interface="org.jetbrains.plugins.gradle.service.project.GradleProjectResolverExtension"/>
复制代码
```

Android IDEA Plugin 中实现了此扩展点，提供了 Android 工程解析的拓展：

```
<extensions defaultExtensionNs="org.jetbrains.plugins.gradle">
  <projectResolve implementation=
  "com.android.tools.idea.gradle.project.sync.idea.AndroidGradleProjectResolver"
   order="first"/>
......
</extensions>
复制代码
```

接着来看 BuildAction 的参数配置逻辑，最终设置 JVM 参数的地方在 GradleExecutionHelper 的 prepare 方法中：

```
List<String> jvmArgs = settings.getJvmArguments();
BuildEnvironment buildEnvironment = getBuildEnvironment(connection, id, listener, (CancellationToken)null, settings);
if (!jvmArgs.isEmpty()) {
  // merge gradle args e.g. defined in gradle.properties
  Collection<String> merged;
  if (buildEnvironment != null) {
    // the BuildEnvironment jvm arguments of the main build should be used for the 'buildSrc' import
    // to avoid spawning of the second gradle daemon
    BuildIdentifier buildIdentifier = getBuildIdentifier(buildEnvironment);
    List<String> buildJvmArguments = buildIdentifier == null || "buildSrc".equals(buildIdentifier.getRootDir().getName())
                                     ? ContainerUtil.emptyList()
                                     : buildEnvironment.getJava().getJvmArguments();
    merged = mergeBuildJvmArguments(buildJvmArguments, jvmArgs);
  } else {
    merged = jvmArgs;
  }
  List<String> filteredArgs = ContainerUtil.mapNotNull(merged, s -> StringUtil.isEmpty(s) ? null : s);
  operation.setJvmArguments(ArrayUtilRt.toStringArray(filteredArgs));
}
复制代码
```

如上代码所示，JVM 参数的配置逻辑很简单：将之前从一系列 GradleProjectResolve 扩展中收集的、存放在 GradleExecutionSettings 中的 JVM 参数和 BuildEnvironment 中的 JVM 参数合并，然后调用 BuildActionExecuter 的 setJvmArguments 方法，将 JVM 参数设置给 BuildAction。Gradle 命令行参数同样是在 GradleExecutionHelper 的 prepare 方法中配置：

```
...
List<String> filteredArgs = new ArrayList<>();
if (!settings.getArguments().isEmpty()) {
  String loggableArgs = StringUtil.join(obfuscatePasswordParameters(settings.getArguments()), " ");
  LOG.info("Passing command-line args to Gradle Tooling API: " + loggableArgs);
  // filter nulls and empty strings
  filteredArgs.addAll(ContainerUtil.mapNotNull(settings.getArguments(), s -> StringUtil.isEmpty(s) ? null : s));
  ...
}
filteredArgs.add("-Didea.active=true");
filteredArgs.add("-Didea.version=" + getIdeaVersion());
operation.withArguments(ArrayUtilRt.toStringArray(filteredArgs));
复制代码
```

对于一个最简单的 Kotlin App Demo 工程，Gradle 命令行参数如下：

<table><thead><tr><th>来源</th><th>参数</th></tr></thead><tbody><tr><td>Android Studio 源码</td><td>--init-script /private/var/folders/_4/j3fdr4nd0x7cf17yvt20f5c00000gp/T/ijmapper.gradle -Didea.sync.active=true -Didea.resolveSourceSetDependencies=true -Porg.gradle.kotlin.dsl.provider.cid=676307056703202 -Pkotlin.mpp.enableIntransitiveMetadataConfiguration=true --init-script/private/var/folders/_4/j3fdr4nd0x7cf17yvt20f5c00000gp/T/ijinit3.gradle -Didea.active=true -Didea.version=2021.3</td></tr><tr><td>Android IDEA Plugin 扩展：com.android.tools.idea.gradle.project.sync.idea.AndroidGradleProjectResolver</td><td>--init-script /private/var/folders/_4/j3fdr4nd0x7cf17yvt20f5c00000gp/T/sync.studio.tooling4770.gradle -Djava.awt.headless=true --stacktrace-Pandroid.injected.build.model.only=true -Pandroid.injected.build.model.only.advanced=true -Pandroid.injected.invoked.from.ide=true -Pandroid.injected.build.model.only.versioned=3 -Pandroid.injected.studio.version=10.4.2 -Pandroid.injected.build.model.disable.src.download=true -Pidea.gradle.do.not.build.tasks=true</td></tr><tr><td>Kotlin IDEA Plugin 扩展：org.jetbrains.kotlin.idea.gradleJava.scripting.importing.KotlinDslScriptModelResolver</td><td>-Dorg.gradle.kotlin.dsl.provider.mode=classpath</td></tr><tr><td>Kotlin IDEA Plugin 扩展：org.jetbrains.kotlin.idea.gradleJava.scripting.importing.KotlinDslScriptModelResolver</td><td>-Porg.gradle.kotlin.dsl.provider.cid=676307056703202</td></tr><tr><td>Kotlin IDEA Plugin 扩展：org.jetbrains.kotlin.idea.gradleJava.configuration.KotlinMPPGradleProjectResolver</td><td>-Pkotlin.mpp.enableIntransitiveMetadataConfiguration=true</td></tr></tbody></table>

重点关注其中的 --init-script 命令行参数，它可以指定一个初始化脚本，初始化脚本会在项目构建脚本之前执行。初始化脚本是 Gradle 提供的一个非常灵活的机制，除了命令行配置，还可以将初始化脚本命名为 init.gradle 放置到 USER_HOME/.gradle/ 下进行配置。初始化脚本允许自定义所有项目的构建逻辑，比如定义特定机器上所有项目的 JDK 路径等环境信息。在上述 Kotlin App Demo 工程中，Android IDEA Plugin 扩展配置了一个初始化脚本，内容如下：

```
initscript {
    dependencies {
        classpath files([mapPath('/Users/bytedance/IDE/intellij-community/out/production/intellij.android.gradle-tooling'), mapPath('/Users/bytedance/IDE/intellij-community/out/production/intellij.android.gradle-tooling.impl'), mapPath('/Users/bytedance/.m2/repository/org/jetbrains/kotlin/kotlin-stdlib/1.5.10-release-945/kotlin-stdlib-1.5.10-release-945.jar')])
    }
}
allprojects {
    apply plugin: com.android.ide.gradle.model.builder.AndroidStudioToolingPlugin
}
复制代码
```

初始化脚本中对所有项目应用了 AndroidStudioToolingPlugin 插件，此插件中通过 ToolingModelBuilderRegistry 注册了 AdditionalClassifierArtifactsModel，这个 Gradle Model 中实现了下载依赖 sources 和 javadoc 的功能：

```
/**
* Model Builder for [AdditionalClassifierArtifactsModel].
*
* This model builder downloads sources and javadoc for components specifies in parameter, and returns model
* [AdditionalClassifierArtifactsModel], which contains the locations of downloaded jar files.
*/
class AdditionalClassifierArtifactsModelBuilder : ParameterizedToolingModelBuilder<AdditionalClassifierArtifactsModelParameter> {
...
复制代码
```

也就是说，Android IDEA Plugin 提供了依赖 sources 和 javadoc 的下载功能，当通过 Gradle Tooling API 获取 AdditionalClassifierArtifactsModel Gradle Model 的时候，会触发依赖的 sources 和 javadoc 下载。

分析完 BuildAction 的 JVM 参数和 Gradle 命令行参数配置流程后，最后来看 BuildAction 环境变量的配置，最终配置的地方在 GradleExecutionHelper 的 setupEnvironment 方法中：

```
GeneralCommandLine commandLine = new GeneralCommandLine();
commandLine.withEnvironment(settings.getEnv());
commandLine.withParentEnvironmentType(
  settings.isPassParentEnvs() ? GeneralCommandLine.ParentEnvironmentType.CONSOLE : GeneralCommandLine.ParentEnvironmentType.NONE);
Map<String, String> effectiveEnvironment = commandLine.getEffectiveEnvironment();
operation.setEnvironmentVariables(effectiveEnvironment);
复制代码
```

GeneralCommandLine 中包括当前 Java 进程所有的环境变量，其他环境变量和 JVM 参数类似都被收集在 GradleExecutionSettings 中，Android Studio 会先将其他环境变量与 GeneralCommandLine 中的环境变量合并，然后配置给 BuildAction. 对于不进行任何配置的默认 sync 行为，GradleExecutionSettings 中环境变量为空，全由 GeneralCommandLine 提供。

### 2.1.3 执行 BuildAction

分析完 BuildAction 的配置逻辑后，接着来看 BuildAction 中具体做了哪些事。BuildAction 中的行为不再处于 Android Studio IDE 进程了，而是在 Gradle 构建进程中执行。IDE 通过 Gradle Tooling API 与 Gradle 交互时，主要媒介是 Gradle Model，BuildAction 中也不例外。BuildAction 的具体执行逻辑见其实现类 ProjectImportAction 中的 execute 方法，我们只关注此方法中与 Gradle Model 相关的代码：

```
public AllModels execute(final BuildController controller) {
  ...
  fetchProjectBuildModels(wrappedController, isProjectsLoadedAction, myGradleBuild);
  addBuildModels(wrappedController, myAllModels, myGradleBuild, isProjectsLoadedAction);
  ...
}
复制代码
```

BuildAction 中调用 fetchProjectBuildModels 和 addBuildModels 方法获取 Gradle Model。先来分析 fetchProjectBuildModels 方法，该方法中进一步调用 getProjectModels 方法：

```
private List<Runnable> getProjectModels(@NotNull BuildController controller,
                                        @NotNull final AllModels allModels,
                                        @NotNull final BasicGradleProject project,
                                        boolean isProjectsLoadedAction) {
    ...
    Set<ProjectImportModelProvider> modelProviders = getModelProviders(isProjectsLoadedAction);
    for (ProjectImportModelProvider extension : modelProviders) {
      extension.populateProjectModels(controller, project, modelConsumer);
    }
    ...
}
复制代码
```

如上代码所示，通过 ProjectImportModelProvider 的 populateProjectModels 方法进一步去获取 Gradle Model。BuildAction 的 addBuildModels 方法与此十分相似：

```
private void addBuildModels(@NotNull final ToolingSerializerAdapter serializerAdapter,
                            @NotNull BuildController controller,
                            @NotNull final AllModels allModels,
                            @NotNull final GradleBuild buildModel,
                            boolean isProjectsLoadedAction) {
    Set<ProjectImportModelProvider> modelProviders = getModelProviders(isProjectsLoadedAction);
    for (ProjectImportModelProvider extension : modelProviders) {
      extension.populateBuildModels(controller, buildModel, modelConsumer);
    }
    ...
}
复制代码
```

可以看到同样是交给了 ProjectImportModelProvider 去获取 Gradle Model，不同的是，前者调用的 populateProjectModels，此处调用的是 populateBuildModels 方法，ProjectImportModelProvider 的作用就是生成 Gradle Model。ProjectImportModelProvider 同 JVM 参数和 Gradle 命令行参数一样，都是由一系列 Gradle IDEA Plugin 扩展提供，如下代码所示：

```
for (GradleProjectResolverExtension resolverExtension = tracedResolverChain;
     resolverExtension != null;
     resolverExtension = resolverExtension.getNext()) {
    ...
    ProjectImportModelProvider modelProvider = resolverExtension.getModelProvider();
    if (modelProvider != null) {
      projectImportAction.addProjectImportModelProvider(modelProvider);
    }
    ProjectImportModelProvider projectsLoadedModelProvider = resolverExtension.getProjectsLoadedModelProvider();
    if (projectsLoadedModelProvider != null) {
      projectImportAction.addProjectImportModelProvider(projectsLoadedModelProvider, true);
    }
}
复制代码
```

对于 Android 工程来说，重点关注 Android IDEA Plugin，它提供的 ProjectImportModelProvider 的实现类为 AndroidExtraModelProvider，该实现类的 populateProjectModels 方法如下：

```
override fun populateProjectModels(controller: BuildController,
                                   projectModel: Model,
                                   modelConsumer: ProjectImportModelProvider.ProjectModelConsumer) {
  controller.findModel(projectModel, GradlePluginModel::class.java)
    ?.also { pluginModel -> modelConsumer.consume(pluginModel, GradlePluginModel::class.java) }
controller.findModel(projectModel, KaptGradleModel::class.java)
    ?.also { model -> modelConsumer.consume(model, KaptGradleModel::class.java) }
}
复制代码
```

此方法中通过 Gradle Tooling API 获取了 GradlePluginModel 和 KaptGradleModel 两个 Gradle Model，GradlePluginModel 中提供了项目已应用的 Gradle Plugin 信息；KaptGradleModel 提供了 Kotlin 注解处理相关信息。接着来看 populateBuildModels 方法：

```
override fun populateBuildModels(
  controller: BuildController,
  buildModel: GradleBuild,
  consumer: ProjectImportModelProvider.BuildModelConsumer) {
  populateAndroidModels(controller, buildModel, consumer)
  populateProjectSyncIssues(controller, buildModel, consumer)
}
复制代码
```

其中分别调用了 populateAndroidModels 和 populateProjectSyncIssues 方法，先来看 populateProjectSyncIssues 方法，它会进一步通过 Gradle Tooling API 获取 ProjectSyncIssues Gradle Model，用于收集 Sync 过程中出现的问题。再来分析 populateAndroidModels 方法，它在整个 Sync 过程中至关重要，其中通过 Gradle Tooling API 获取了 Android 工程相关的 Gradle Model：

```
val androidModules: MutableList<AndroidModule> = mutableListOf()
  buildModel.projects.forEach { gradleProject ->
findParameterizedAndroidModel(controller, gradleProject, AndroidProject::class.java)?.also { androidProject ->
consumer.consumeProjectModel(gradleProject, androidProject, AndroidProject::class.java)
      val nativeAndroidProject = findParameterizedAndroidModel(controller, gradleProject, NativeAndroidProject::class.java)?.also {
consumer.consumeProjectModel(gradleProject, it, NativeAndroidProject::class.java)
      }
androidModules.add(AndroidModule(gradleProject, androidProject, nativeAndroidProject))
    }
} 
复制代码
```

如上代码所示，分别获取了由 Android Gradle Plugin 注册的两个 Gradle Model AndroidProject 和 NativeAndroidProject，AndroidProject 中包括 Android 应用的 BuildType、Flavors、Variant、Dependency 等关键信息；NativeAndroidProject 中包括 NDK、NativeToolchain 等 Android C/C++ 项目相关信息。获取了最关键的 AndroidProject 和 NativeAndroidProject 后，接着是对单 Variant Sync 的处理：

```
if (syncActionOptions.isSingleVariantSyncEnabled) {
    chooseSelectedVariants(controller, androidModules, syncActionOptions)
}
复制代码
```

怎么理解单 Variant Sync 呢？Variant 指 Android 应用的产物变体，比如最简单的 Debug 和 Release 版本应用包。Variant 对应于一套构建配置，与项目源码结构、依赖列表相关联。Android 应用可能有多个 Variant，如果在 Sync 时构造所有 Variant，整体耗时可能极长，所以 Android Studio Sync 默认只会构造一个 Variant，并支持 Variant 切换功能。如果启用了单 Variant Sync，前面获取 AndroidProject 时会传入过滤参数，告知 Android Gradle Plugin 构造 AndroidProject 时无需构造 Variant 信息。

2.2 Sync 流程梳理
-------------

### 2.2.1 Android Studio 视角

Android Studio Sync 流程的源码庞大而繁杂，本文着重分析了从触发 Sync 入口到 Gradle 构建结束的阶段，后续还有对 Gradle 数据的处理，以及语言能力模块基于 Gradle 数据进行代码索引的流程，而语言能力又是一个大型而复杂的模块，本文就不再继续展开。

上文源码分析部分将触发 Sync 入口到 Gradle 构建结束划分为 Sync 功能入口及准备、配置 BuildAction 以及执行 BuildAction，整体流程如下图所示：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/22ef0a3295ef446dae22b24a016ad21f~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

### 2.2.2 Gradle 视角

上面都是以 IDE 视角去分析 Android Studio Sync，整体流程较复杂，接下来以 Gradle 视角去梳理 Sync 流程，着重关注 Gradle 侧的行为：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/101ac0f140344d2099c1e990ff781c53~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

如上图，对于 Gradle 来说，Sync 流程中 Android Studio 会通过 Gradle Tooling API 从 Gradle 侧获取一系列所需的 Gradle Model，除 Gradle 自身外，Gradle Plugin 也可以提供自定义的 Gradle Model。另外 Sync 流程中 Gradle 会经历自身定义的生命周期，聚焦此视角梳理流程如下：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8459009127e1475cb057659051fae819~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

当通过 BuildAction 获取 Gradle Model 时会触发 Gradle 的构建行为，Gradle 构建会经历自身生命周期定义的 Initialization、Configuration 和 Execution 阶段。

3. 总结
=====

本文首先介绍了 Android Studio Sync 流程中各个角色的作用及联系，对 Sync 有一个较清晰的整体认识，然后从源码角度深入分析从触发 Sync 入口到 Gradle 构建结束的阶段，并详细解释了 Gradle Model、BuildAction 等关键概念，最后分别从 Android Studio 视角和 Gradle 视角对 Sync 流程进行了整体梳理。

通过对 Android Studio Sync 流程的深入分析，除了对 Sync 功能的实现原理深度掌握外，对其意义也有了更深的理解。Sync 是 Android Studio 定义的一个 IDE 准备阶段，在这个准备阶段中，需提前准备好关键的 IDE 功能，而这些功能要达到可用状态，需要获取其必需的数据。基于这个角度，对 Sync 流程的优化方向也有了一定启发：首先从产品层面出发考虑 Sync 阶段的定义，不是开发者真正必需的功能都可以考虑省略或延后准备；然后确认必需功能所需的最小数据集，不必需的数据都可以省略；最后针对必需数据，通过更高效的实现或缓存，找到最快的获取方式。

4. **参考链接**
===========

[mp.weixin.qq.com/s/cftj6Wueo…](https://link.juejin.cn?target=https%3A%2F%2Fmp.weixin.qq.com%2Fs%2Fcftj6WueoHlLh-So9REEXQ "https://mp.weixin.qq.com/s/cftj6WueoHlLh-So9REEXQ")

[developer.android.com/studio/intr…](https://link.juejin.cn?target=https%3A%2F%2Fdeveloper.android.com%2Fstudio%2Fintro "https://developer.android.com/studio/intro")

[android.googlesource.com/platform/to…](https://link.juejin.cn?target=https%3A%2F%2Fandroid.googlesource.com%2Fplatform%2Ftools%2Fbase%2F%2B%2Fstudio-master-dev "https://android.googlesource.com/platform/tools/base/+/studio-master-dev")

[plugins.jetbrains.com/developers](https://link.juejin.cn?target=https%3A%2F%2Fplugins.jetbrains.com%2Fdevelopers "https://plugins.jetbrains.com/developers")

5. 关于我们
=======

我们是字节跳动终端技术团队（Client Infrastructure）下的 Developer Tools 团队，负责打造公司范围内，面向不同业务场景的研发工具，提升移动应用研发效率。目前急需寻找 Android 移动研发工程师 / iOS 移动研发工程师 / 服务端研发工程师。了解更多信息请联系：[wangyinghao.ahab@bytedance.com](https://link.juejin.cn?target=mailto%3Awangyinghao.ahab%40bytedance.com "mailto:wangyinghao.ahab@bytedance.com")，邮件主题 简历 - 姓名 - 求职意向 - 期望城市 - 电话。