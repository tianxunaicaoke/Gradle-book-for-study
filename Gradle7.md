# Gradle 进阶 第八篇

宁为玉碎，不为瓦全

## Gradle Project 下卷
上一章讲到 Gradle 的 ConfigurationContainer，ConfigurationContainer 里面包含了一些系列的 Configuration，而 Configuration 又继承了 FileCollection 接口。其实现类 DefaultConfiguration 中包括了对外发布的一个集合，以及构建依赖的一个集合。以供 project 管理依赖，并且对外提供构建好的 Artifact。
~~~
public class DefaultConfiguration extends AbstractFileCollection implements ConfigurationInternal, MutationValidator {
    ...
    private DefaultDependencySet allDependencies;
    ...
    private DefaultPublishArtifactSet allArtifacts;
}
~~~

## ArtifactHandler

## ExtensibleDynamicObject
ExtensibleDynamicObject 我在第二章讲过了，在动态调用系统中详细描述了，这个类的作用，有不清楚的可以回看一下，这里就不在赘述。

