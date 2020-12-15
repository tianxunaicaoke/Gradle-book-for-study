# Gradle 进阶 第五篇

虽千万人，吾往矣

## Plugin 源码

接着上一节所讲的微内核架构，系统的 Plugin 管理简单的实现方式就是通过在系统内部实现一个注册表，用来获取 Plugin，并且得到 Plugin 的可用性。
下面来从源码展开了解一下 Gradle plugin 的管理。在源码的110多个模块中，plugin 管理先关的有三个，分别是 gradle.plugin-development、gradle.plugins、以及 gradle.plugin-user。
其中 gradle.plugin-development 是关于支持开发 plugin 的模块，
在后面“如何设计 Plugin”中将会详细讲解。这里举个简单例子：JavaGradlePluginPlugin，它就是我们平常写 plugin 时需要 apply 的 plugin，摘个官网的例子：
~~~
plugins {
    id 'java-gradle-plugin'
}

gradlePlugin {
    plugins {
        simplePlugin {
            id = 'org.samples.greeting'
            implementationClass = 'org.gradle.GreetingPlugin'
        }
    }
}
~~~
gradle.plugins 模块中定义了大部分的内置 Plugins，这里就一句带过，后文会有一系列专门的文章来介绍一些特别的 Plugin。
本文的重点是系统如何管理 plugin，也就是模块 gradle.plugin-user的主要功能。

## plugin 查找
之前的一篇文章中讲解了有关 Gradle 脚本的编译过程，其中有一个细节省略未说，就是在编译每一个 .gradle 文件的时候都会分为两部分，如果脚本文件里有 buildscript{} 代码块，就会先编译 buildscript{} 里的代码、或者 apply xxx ，或者plugins{} 为一个 class，编译完成之后直接运行，接着编译剩下的代码为一个 class，再运行。
举个简单的例子：
~~~
// case 1
plugins {
    id 'com.android.application'
}
======= 分割线 ======
android {
    compileSdkVersion 30
    buildToolsVersion "30.0.2"

    defaultConfig {
      ...
    }
}
// case 2
buildscript {
    repositories {
        google()
        jcenter()
    }
    dependencies {
        classpath "com.android.tools.build:gradle:4.1.0"
    }
}
======= 分割线 ======
allprojects {
    repositories {
        google()
        jcenter()
    }
}

task clean(type: Delete) {
    delete rootProject.buildDir
}
~~~
在分割线之上的会先编译，并且运行，之后编译分割线之下的部分，再运行。其中 plugins {} 的代码块就是一个 PluginRequest。Gradle 会根据这个 PluginRequest 去查找需要加载的 plugin。这里需要注意一点，case 2 中的 classpath "com.android.tools.build:gradle:4.1.0"，


Gradle 用来管理插件的类是 DefaultPluginManager，具体实现如下：
~~~
public class DefaultPluginManager implements PluginManagerInternal {
...
    @Override
    public void apply(String pluginId) {
        PluginImplementation<?> plugin = pluginRegistry.lookup(DefaultPluginId.unvalidated(pluginId));
        if (plugin == null) {
            throw new UnknownPluginException("Plugin with id '" + pluginId + "' not found.");
        }
        doApply(plugin);
    }
...
}
~~~