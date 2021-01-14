# Gradle 进阶 第九篇

学不可以已

## NamedDomainObjectContainer
这里一章会讨论 Gradle 中一个非常重要的概念 NamedDomainObjectContainer。它是一个命名对象领域容器，源码里的注释解释，NamedDomainObjectContainer 其实就是一个可以拥有创建单个 item 能力的 NamedDomainObjectSet。而 NamedDomainObjectSet 又是可以根据 item 名字来排列的一个集合，这个集合里的元素都是一个领域的对象，以不同的名字来区别，所以一个集合里的名字在一个领域里只能是惟一的。
我先展示一张类图来大概感觉一下什么是 NamedDomainObject 相关的一系列关系：

<img src="NamedDomain.png">

理解 NamedDomainObjectContainer 的字面意思比较容易，这里我就先用一个栗子来开始：

~~~
plugins {
    id 'java'
}

sourceSets {
  main {
    java {
      
    }
  }
  test {
    java{

    }
  }
  example {
    java{

    }
  }
}
~~~
在 sourceSets 中创建了 main，test，example 三个 sourceSet，实际上在 JavaPlugin 被 apply 的时候，已经创建好了 main 和 test 两个 sourceSet，这里只是会去覆盖这两个 sourceSet 的配置。如下：
但是 example 是我们通过名字来创建了一个新的 sourceSet 对象。
~~~
 -- JavaPlugin.java
  private void configureSourceSets(JavaPluginConvention pluginConvention, final BuildOutputCleanupRegistry buildOutputCleanupRegistry) {
        Project project = pluginConvention.getProject();
        SourceSetContainer sourceSets = pluginConvention.getSourceSets();

        SourceSet main = sourceSets.create(SourceSet.MAIN_SOURCE_SET_NAME);

        SourceSet test = sourceSets.create(SourceSet.TEST_SOURCE_SET_NAME);
        ...
    }
~~~
sourceSets 的函数调用代理到了 SourceSetContainer 接口，它的实现类是 DefaultSourceSetContainer:
~~~
public class DefaultSourceSetContainer extends AbstractValidatingNamedDomainObjectContainer<SourceSet> implements SourceSetContainer{
    ...
     protected SourceSet doCreate(String name) {
        DefaultSourceSet sourceSet = instantiator.newInstance(DefaultSourceSet.class, name, objectFactory);
        sourceSet.setClasses(instantiator.newInstance(DefaultSourceSetOutput.class, sourceSet.getDisplayName(), fileResolver, fileCollectionFactory));
        return sourceSet;
    }
    ...

}
~~~