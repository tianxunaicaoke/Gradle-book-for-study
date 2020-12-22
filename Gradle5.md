# Gradle 进阶 第六篇

有志者，事竟成

## Gradle Project
在之前的一系列文章中，我们从一些方面了解了 Gradle，这一片用一个栗子把她们串联起来。从这篇之后，我们会系统的把 Gradle 的重要组件作以讲解。 Project，我在前文有大量讲解关于".gradle"文件，Project 就是和"build.gradle"一一对应。多模块项目往往一个模块就对应一个 Project，之前的文章讲解了 ".gradle"文件的编译和解析。
~~~
plugins {
    id 'com.android.application'
}
============ above will first compile and run ============
android {
    compileSdkVersion 29
    buildToolsVersion "30.0.2"

    defaultConfig {
        applicationId "com.example.gradledebug"
        minSdkVersion 23
        targetSdkVersion 29
        versionCode 1
        versionName "1.0"

        testInstrumentationRunner "androidx.test.runner.AndroidJUnitRunner"
    }

    buildTypes {
        release {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
        }
    }
    compileOptions {
        sourceCompatibility JavaVersion.VERSION_1_8
        targetCompatibility JavaVersion.VERSION_1_8
    }
}

dependencies {
    implementation 'androidx.appcompat:appcompat:1.2.0'
    implementation 'com.google.android.material:material:1.2.1'
    implementation 'androidx.constraintlayout:constraintlayout:2.0.4'
    testImplementation 'junit:junit:4.+'
    androidTestImplementation 'androidx.test.ext:junit:1.1.2'
    androidTestImplementation 'androidx.test.espresso:espresso-core:3.3.0'
}
~~~
以上栗子里的每一个代码块，对应下面一个小章节。

## plugins { ... } 解析
上一章讲的就是 plugin 的管理，就像我在栗子里加的分割线，在分割线之上的会先编译，并且运行。在运行的过程中会生成对于 android plugin 的请求，并查找，最后把 Plugin 对应的路径加入 Gradle 的相应的 ClassLoader 的 path 里，这样在代码里就可以真正的调用到 Plugin 对应的 apply 方法。
我在这里直接就展示出 android plugin 的 apply 方法。
~~~
 class AppPlugin extends BasePlugin{
    ...
    @Override
    public void apply(@NonNull Project project) {
        super.apply(project);
    }
    ...
 }

 abstract class BasePlugin<E extends BaseExtension2>
        implements Plugin<Project>, ToolingRegistryProvider {
        ...
     @Override
     public void apply(@NonNull Project project) {
        // We run by default in headless mode, so the JVM doesn't steal focus.
        System.setProperty("java.awt.headless", "true");

        this.project = project;
        this.projectOptions = new ProjectOptions(project);

        project.getPluginManager().apply(AndroidBasePlugin.class);

        checkPathForErrors();
        checkModulesForErrors();

        PluginInitializer.initialize(project);
        ProfilerInitializer.init(project, projectOptions);
        threadRecorder = ThreadRecorder.get();

        ProcessProfileWriter.getProject(project.getPath())
                .setAndroidPluginVersion(Version.ANDROID_GRADLE_PLUGIN_VERSION)
                .setAndroidPlugin(getAnalyticsPluginType())
                .setPluginGeneration(GradleBuildProject.PluginGeneration.FIRST)
                .setOptions(AnalyticsUtil.toProto(projectOptions));

        BuildableArtifactImpl.Companion.disableResolution();
        if (!projectOptions.get(BooleanOption.ENABLE_NEW_DSL_AND_API)) {
            TaskInputHelper.enableBypass();

            threadRecorder.record(
                    ExecutionType.BASE_PLUGIN_PROJECT_CONFIGURE,
                    project.getPath(),
                    null,
                    this::configureProject);

            threadRecorder.record(
                    ExecutionType.BASE_PLUGIN_PROJECT_BASE_EXTENSION_CREATION,
                    project.getPath(),
                    null,
                    this::configureExtension);

            threadRecorder.record(
                    ExecutionType.BASE_PLUGIN_PROJECT_TASKS_CREATION,
                    project.getPath(),
                    null,
                    this::createTasks);
        } else {
            // Apply the Java plugin
            project.getPlugins().apply(JavaBasePlugin.class);

            // create the delegate
            ProjectWrapper projectWrapper = new ProjectWrapper(project);
            PluginDelegate<E> delegate =
                    new PluginDelegate<>(
                            project.getPath(),
                            project.getObjects(),
                            project.getExtensions(),
                            project.getConfigurations(),
                            projectWrapper,
                            projectWrapper,
                            project.getLogger(),
                            projectOptions,
                            getTypedDelegate());

            delegate.prepareForEvaluation();

            // after evaluate callbacks
            project.afterEvaluate(
                    p ->
                            threadRecorder.record(
                                    ExecutionType.BASE_PLUGIN_CREATE_ANDROID_TASKS,
                                    p.getPath(),
                                    null,
                                    delegate::afterEvaluate));
        }
    }
        ...
        
 }
~~~
其中需要注意 1. 在 apply 方法里面还 apply 了 AndroidBasePlugin。2.configureProject，configureExtension，createTasks。3. 如果 ENABLE_NEW_DSL_AND_API 设置成 true 会 apply java plugin。没有找到太多关于 ENABLE_NEW_DSL_AND_API 的作用，但是通过源码推断，使用了 ENABLE_NEW_DSL_AND_API 为 true，就是用 kotlin 语言来完成 Android 相关的构建。这里的 android gradle plugin 的版本号是3.1.0。
## android { ... } 解析
android {} 在 Groovy 语法中表示: android(Closure closure)，closure 就是 {} 里的代码，在前面的文章里，已经讲解了 Gradel 脚本文件的函数调用。在 apply 了 com.android.application plugin 之后，会以"android" 为名字，把一个 BaseExtension 加入 Project 中的 Convention，所以当脚本运行到 android{} 时，其实就是运行了 android(closure)，就是用{}里的闭包来配置 BaseExtension。所以这里就直接接着前文的 configureExtension 开始解析。
~~~
    private Object configureExtension(String name, Object[] args){
        Closure closure = (Closure) args[0];
        Action<Object> action = ConfigureUtil.configureUsing(closure);
        return extensionsStorage.configureExtension(name, action);
    }
~~~
在这个代码片段里重点是 configureUsing，代码如下：
~~~
    /**
     * Creates an action that uses the given closure to configure objects of type T.
     */
    public static <T> Action<T> configureUsing(@Nullable final Closure configureClosure) {
        if (configureClosure == null) {
            return Actions.doNothing();
        }

        return new WrappedConfigureAction<T>(configureClosure);
    }
~~~
如同注解里所说的，这个函数只是为了创建一个用给定的 closure 去配置某个 object 的一个操作。其中 WrappedConfigureAction 代码如下：
~~~
        public static class WrappedConfigureAction<T> implements Action<T> {
        private final Closure configureClosure;

        WrappedConfigureAction(Closure configureClosure) {
            this.configureClosure = configureClosure;
        }

        @Override
        public void execute(T t) {
            configure(configureClosure, t);
        }

        public Closure getConfigureClosure() {
            return configureClosure;
        }
    }
~~~
在 execute 中调用 configure(configureClosure, t);
~~~
    public static <T> T configure(@Nullable Closure configureClosure, T target) {
        if (configureClosure == null) {
            return target;
        }

        if (target instanceof Configurable) {
            ((Configurable) target).configure(configureClosure);
        } else {
            configureTarget(configureClosure, target, new ConfigureDelegate(configureClosure, target));
        }

        return target;
    }
~~~
这里我分析一下 target 不是 Configurable 的情况：
~~~
        private static <T> void configureTarget(Closure configureClosure, T target, ConfigureDelegate closureDelegate) {
        ...

        Closure withNewOwner = configureClosure.rehydrate(target, closureDelegate, configureClosure.getThisObject());
        new ClosureBackedAction<T>(withNewOwner, Closure.OWNER_ONLY, false).execute(target);
    }
~~~
其中 rehydrate 是 Groovy 提供的函数，可以在 copy closure 的时候，替换 closure 的 delegate，this，owner。最终调用 ClosureBackedAction 的 excute 函数。
这里可以啰嗦一下关于 Groovy Closure 相关的知识点，讲 Closure 的话首先就要讲委托策略，要理解委托的概念，我们必须首先了解 this 在闭包中的含义。闭包实际上定义了3个不同的东西:
> 1. this 对应于定义闭包的封闭类
> 
> 2. owner 对应于定义闭包的封闭对象，该对象可以是类，也可以是闭包
> 
> 3. delegate 对应于第三方对象，在该对象中，当未定义消息的接收者时解析方法调用或属性。

同时闭包定义了多种可供选择的解析策略:

> 1. OWNER_FIRST 是默认策略。如果属性/方法存在于所有者上，那么它将在所有者上被调用。如果没有，则使用委托。
> 2. DELEGATE_FIRST 首先使用委托，然后才是所有者
> 3. OWNER_ONLY 只解析所有者上的属性/方法查找:委托将被忽略。
> 4. DELEGATE_ONLY只解决委托上的属性/方法查找:所有者将被忽略。
> 5. TO_SELF 供开发人员使用

所以闭包的执行就是按照解析策略来调用配置好的委托。
~~~
    public void execute(T delegate) {
        if (closure == null) {
            return;
        }

        try {
            if (configurableAware && delegate instanceof Configurable) {
                ((Configurable) delegate).configure(closure);
            } else {
                Closure copy = (Closure) closure.clone();
                copy.setResolveStrategy(resolveStrategy);
                copy.setDelegate(delegate);
                if (copy.getMaximumNumberOfParameters() == 0) {
                    copy.call();
                } else {
                    copy.call(delegate);
                }
            }
        } catch (groovy.lang.MissingMethodException e) {
            if (Objects.equal(e.getType(), closure.getClass()) && Objects.equal(e.getMethod(), "doCall")) {
                throw new InvalidActionClosureException(closure, delegate);
            }
            throw e;
        }
    }
~~~
可以看到最重点的逻辑在 copy.setDelegate(delegate); 简单说就是把需要配置的对象，设置成为 closure 的 delegate，closure 的执行就最终转换成为对需要配置的对象的配置。
~~~
{
    compileSdkVersion 29
    buildToolsVersion "30.0.2"

    defaultConfig {
        applicationId "com.example.gradledebug"
        minSdkVersion 23
        targetSdkVersion 29
        versionCode 1
        versionName "1.0"

        testInstrumentationRunner "androidx.test.runner.AndroidJUnitRunner"
    }

    buildTypes {
        release {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
        }
    }
    compileOptions {
        sourceCompatibility JavaVersion.VERSION_1_8
        targetCompatibility JavaVersion.VERSION_1_8
    }
}
~~~
回到栗子上来，就是把 AppExtension 设置成为上面 closure 的 delegate。
AppExtension 继承自 BaseExtension。
~~~
 public abstract class BaseExtension implements AndroidConfig {
       ...
       public void compileSdkVersion(int apiLevel) {
        compileSdkVersion("android-" + apiLevel);
       }
       ...
       public void buildToolsVersion(String version) {
        checkWritability();
        ...
        buildToolsRevision = Revision.parseRevision(version, Revision.Precision.MICRO);
      }
      ...
       public void defaultConfig(Action<DefaultConfig> action) {
        checkWritability();
        action.execute(defaultConfig);
     }
     ...

 }
~~~
可以看到 android {} 中的函数调用最终都映射到 BaseExtension 中。
## dependencies { ... } 解析
dependencies 这个代码块中配置了 Project 构建时所需要的依赖，其中包含了很多细节，是 Gradle 中非常重要，并且是 Gradle 支柱性的一个功能。后面会有一系列的文章来讲解，这里就简单的解释一下。 
~~~
dependencies {
    implementation 'androidx.appcompat:appcompat:1.2.0'
    implementation 'com.google.android.material:material:1.2.1'
    implementation 'androidx.constraintlayout:constraintlayout:2.0.4'
    testImplementation 'junit:junit:4.+'
    androidTestImplementation 'androidx.test.ext:junit:1.1.2'
    androidTestImplementation 'androidx.test.espresso:espresso-core:3.3.0'
}
~~~
其中 implementation 是一个 configuration，Gradle 会从配置好的仓库里根据声明的依赖去查找。声明的完整格式是分三段，GroupId:ArtifactId:Version。
整个依赖系统里面还会包括，依赖的解析，冲突的解决策略，仓库的管理，configuration 的管理等等很多，需要细细去讲解，这里就到此为止。
