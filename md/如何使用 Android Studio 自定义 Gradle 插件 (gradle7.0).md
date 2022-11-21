> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7166445120714178597)

Android Stuido 自定义 Gradle7.0 插件
===============================

> 本文包括以下三个部分:
> 
> 1.  使用 android stuido 定义一个自定义 gradle 插件
> 2.  发布插件到本地
> 3.  使用插件

如何使用 Android stuido 创建一个插件
--------------------------

### 1、新建 Android 工程

### 2、创建 Android Library

这里我们创建了一个名为 plugin 的 module

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f1915902c5844bfa8c3bc4ef2b4b9329~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

此时 plugin module 好只是一个 android library，要让它成为 gradle 插件模块还需要做些改造。

### 修改 build.gradle 文件

```
plugins {
    id 'groovy'
}
dependencies {
    //gradle sdk
    implementation gradleApi()
    //groovy sdk
    implementation localGroovy()
}
复制代码
```

修改 build 文件后编译

### 配置 plugin 的目录结构

1.  删除不必要的文件和目录只保留 build.gradle 和 src/main
2.  在 src/main 目录下新建 groovy 目录，插件代码保存在此处
3.  在 src/main 目录下新建 src/main/resources/META-INF/gradle-plugins 目录，插件声明在此处
4.  创建插件代码，在 groovy 目录下创建自定义包，在包中创建一个 MyPlugin.java 重命名文件为 MyPlugin.groovy 简单输出一句话。

```
package com.custom.plugin

import org.gradle.api.Plugin
import org.gradle.api.Project;


class MyPlugin implements Plugin<Project> {

    @Override
    void apply(Project project) {
        println "hello, this is my custom plugin!"
    }
}
复制代码
```

6.  声明插件 在刚刚创建的 gradle-plugins 文件目录下创建插件声明文件 custom-gradle-plugin.properties 文件中需要描述插件的类路径。

```
implementation-class=com.custom.plugin.MyPlugin
复制代码
```

> **注意这里的文件名将是你在引用此插件时的插件名称**。

好了到这里插件就创建完成了， 我们来看下插件的目录结构。

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/aba99fbd88af4b1abb97547bd5473d5c~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

接下来我们将学习如何发布插件和使用插件

如何发布插件到本地
---------

### 配置 gradle 文件

需要配置发布信息，这里只发布到本地。如何发布到远端自行 google, 这里发布到了项目目录下 repo 目录中。

```
plugins {
    id 'groovy'
    id 'maven-publish'
}
dependencies {
    //gradle sdk
    implementation gradleApi()
    //groovy sdk
    implementation localGroovy()
}

publishing {
    // 定义发布什么
    publications {
        plugin(MavenPublication) {
            groupId = "com.custom.plugin"
            artifactId = 'Myplugin'
            version = '1.0.0'
            from components.java
        }
    }
    repositories {
        maven {
            name = 'repo'
            url = "../repo"
        }
    }
}
复制代码
```

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c84da03421e944aea5c60f743ce9b869~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/185923c8923c4ce3ae9024e23b2bf358~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

插件就发布到本地仓库了，我们来看下项目结构

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1caee3dd01f24e31a078118593afd056~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?) 到这里插件就算发布完成了。接下来我们继续学习如何使用插件。

如何使用插件
------

### 引入本地仓库

在项目的 setting.gradle 文件中添加本地仓库

```
maven {
    allowInsecureProtocol(true)
    url uri('./repo')
}
复制代码
```

最终如下

```
pluginManagement {
    repositories {
        gradlePluginPortal()
        google()
        mavenCentral()
        //添加本地仓库
        maven {
            allowInsecureProtocol(true)
            url uri('./repo')
        }
    }
}
dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
    repositories {
        google()
        mavenCentral()
        //添加本地仓库
        maven {
            allowInsecureProtocol(true)
            url uri('./repo')
        }
    }
}
rootProject.name = "GradleCustomPlugin"
include ':app'
include ':plugin'
复制代码
```

### 引用插件

在项目的 buid.gadle 文件中添加

```
buildscript {
    dependencies {
        classpath('com.custom.plugin:Myplugin:1.0.0')
    }
}
复制代码
```

最终如下

```
buildscript {
    dependencies {
        classpath('com.custom.plugin:Myplugin:1.0.0')
    }
}

// Top-level build file where you can add configuration options common to all sub-projects/modules.
plugins {
    id 'com.android.application' version '7.3.0' apply false
    id 'com.android.library' version '7.3.0' apply false
}
复制代码
```

### 在主模块下使用插件

在 app 模块下的 build.gradle 文件中添加插件

```
plugins {
    id 'com.android.application'
    // 添加插件
    id 'custom-gradle-plugin'
}
复制代码
```

检查是否成功
------

编译引用插件的模块。会看到我们在 MyPlugin.groovy 中打印的提示 hello, this is my custom plugin!

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b08ae8d425b34248acf5e791f6536a1a~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)