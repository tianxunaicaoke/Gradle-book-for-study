# Gradle 进阶 第五篇

虽千万人，吾往矣

## Plugin 应用

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
一般在 Gradle 的使用中，apply 一个 plugin 都是通过在脚本文件中声明来完成的，就如同上面的代码片中所示，apply 了"java-gradle-plugin"。Gradle 也提供了通过 java 代码来 apply 一个 plugin，代码如下：
~~~
 //invoke side
 project.getPluginManager().apply(BasePlugin.class);
~~~

~~~
 ==========================
// in DefaultPluginManager
    public void apply(String pluginId) {
        PluginImplementation<?> plugin = pluginRegistry.lookup(DefaultPluginId.unvalidated(pluginId));
        if (plugin == null) {
            throw new UnknownPluginException("Plugin with id '" + pluginId + "' not found.");
        }
        doApply(plugin);
    }

~~~
除了和 Gradle 一起发布的内部 plugin 之外，我们也可自定义一些本地的 Plugin，或者使用一些远程的第三方 plugin。

> 1. 应用 buildSrc 目录下的 Plugin
> 2. 应用脚本里直接编写的 Plugin
> 3. 应用本地/远程仓库里的 Plugin

这里就不做详细的demo，下面分析源码的时候会有提及。


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
~~~

~~~
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
在分割线之上的会先编译，并且运行，之后编译分割线之下的部分，再运行。其中 plugins {} 的代码块就是用来声明所需要的 Plugin 的一个Request。Gradle 会根据这个 PluginRequest 去查找需要加载的 plugin。

这里需要注意一点，case 2 中的 classpath "com.android.tools.build:gradle:4.1.0"，已经将 android plugin 加入了 dependencies 中，Gradle 就会去指定的仓库中去下载相应的插件版本并且加入 ClassPath 的 scorp 中。所以对于"应用本地/远程仓库里的 Plugin"，都需要 buildscript 的 dependencies 中声明。

关于 Gradle 是如何通过脚本把 Plugin 加入 Runtime，我先上一幅类图，作以梳理：
<img src="PluginManager.png">

在 DefaultScriptPluginFactory 类中：
~~~
            // Pass 1, extract plugin requests and plugin repositories and execute buildscript {}, ignoring (i.e. not even compiling) anything else

            CompileOperation<?> initialOperation = compileOperationFactory.getPluginsBlockCompileOperation(initialPassScriptTarget);
            Class<? extends BasicScript> scriptType = initialPassScriptTarget.getScriptClass();
            ScriptRunner<? extends BasicScript, ?> initialRunner = compiler.compile(scriptType, initialOperation, baseScope, Actions.doNothing());
            initialRunner.run(target, services);

            PluginRequests initialPluginRequests = getInitialPluginRequests(initialRunner);
            PluginRequests mergedPluginRequests = autoAppliedPluginHandler.mergeWithAutoAppliedPlugins(initialPluginRequests, target);

            PluginManagerInternal pluginManager = topLevelScript ? initialPassScriptTarget.getPluginManager() : null;
            pluginRequestApplicator.applyPlugins(mergedPluginRequests, scriptHandler, pluginManager, targetScope);

~~~
其中 getInitialPluginRequests 就是获得脚本里的 Plugin 请求，plugins { id 'com.android.application' }，接着合并完所有的 Plugin 请求之后调用 applyPlugins 方法去解析 Plugin 请求。

~~~
    @Override
    public void applyPlugins(final PluginRequests requests, final ScriptHandlerInternal scriptHandler, @Nullable final PluginManagerInternal target, final ClassLoaderScope classLoaderScope) {
        if (target == null || requests.isEmpty()) {
            defineScriptHandlerClassScope(scriptHandler, classLoaderScope, Collections.emptyList());
            return;
        }

        final PluginResolver effectivePluginResolver = wrapInAlreadyInClasspathResolver(classLoaderScope);
        if (!requests.isEmpty()) {
            addPluginArtifactRepositories(scriptHandler.getRepositories());
        }
        List<Result> results = resolvePluginRequests(requests, effectivePluginResolver);

        // Could be different to ids in the requests as they may be unqualified
        final Map<Result, PluginId> legacyActualPluginIds = newLinkedHashMap();
        final Map<Result, PluginImplementation<?>> pluginImpls = newLinkedHashMap();
        final Map<Result, PluginImplementation<?>> pluginImplsFromOtherLoaders = newLinkedHashMap();

        if (!results.isEmpty()) {
            for (final Result result : results) {
                applyPlugin(result.request, result.found.getPluginId(), new Runnable() {
                    @Override
                    public void run() {
                        result.found.execute(new PluginResolveContext() {
                            @Override
                            public void addLegacy(PluginId pluginId, Object dependencyNotation) {
                                legacyActualPluginIds.put(result, pluginId);
                                scriptHandler.addScriptClassPathDependency(dependencyNotation);
                            }

                            @Override
                            public void add(PluginImplementation<?> plugin) {
                                pluginImpls.put(result, plugin);
                            }

                            @Override
                            public void addFromDifferentLoader(PluginImplementation<?> plugin) {
                                pluginImpls.put(result, plugin);
                                pluginImplsFromOtherLoaders.put(result, plugin);
                            }
                        });
                    }
                });
            }
        }

        defineScriptHandlerClassScope(scriptHandler, classLoaderScope, pluginImplsFromOtherLoaders.values());
        applyLegacyPlugins(target, legacyActualPluginIds);
        applyPlugins(target, pluginImpls);
    }
~~~
解释一下四个参数，第一个 PluginRequests 是通过读取 plugins{} 里的插件请求并且合并了自动添加的一些 core Plugin 的请求。第二个参数 ScriptHandlerInternal 的实现类是 DefaultScriptHandler，
~~~
public class DefaultScriptHandler implements ScriptHandler, ScriptHandlerInternal, DynamicObjectAware {
  ...
    @Override
    public void dependencies(Closure configureClosure) {
        ConfigureUtil.configure(configureClosure, getDependencies());
    }
  ...
}
~~~

其中的 dependencies 方法就是 buildscript {dependencies {}}的具体实现。第三个参数 PluginManagerInternal 的实现类，就是我在上文提到的 DefaultPluginManager，第四个参数 ClassLoaderScope Represents a particular node in the ClassLoader graph（这是源码里的注解，觉得的翻译的话影响理解）。

回归正题，applyPlugins 里第一步会通过 PluginResolver 去解析 PluginRequest，并且返回解析好的 Result，代码如下：
~~~
 private List<Result> resolvePluginRequests(PluginRequests requests, PluginResolver effectivePluginResolver) {
        return collect(requests, request -> {
            PluginRequestInternal configuredRequest = pluginResolutionStrategy.applyTo(request);
            return resolveToFoundResult(effectivePluginResolver, configuredRequest);
        });
    }
~~~
注意两点：1. PluginResolver 的实现有多个，它们串联在一起，将从头到尾搜索每一个 Resolver，直到找到插件。2. 每一个 PluginRequest 如果找到结果，就会封装成一个 Result 返回，在 Result 里有一个 PluginResolution。上一个代码片：
~~~
public class ClassPathPluginResolution implements PluginResolution {

   ...

    @Override
    public void execute(PluginResolveContext pluginResolveContext) {
        PluginRegistry pluginRegistry = new DefaultPluginRegistry(pluginInspector, parent);
        PluginImplementation<?> plugin = pluginRegistry.lookup(pluginId);
        if (plugin == null) {
            throw new UnknownPluginException("Plugin with id '" + pluginId + "' not found.");
        }
        pluginResolveContext.add(plugin);
    }
}
~~~
最终是通过  PluginImplementation<?> plugin = pluginRegistry.lookup(pluginId);  DefaultPluginRegistry 来查找到 PluginImplementation。
这个流程比较长，就先到这里。后续会添加。