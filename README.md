# Gradle book

## Dependency-manager

首先要介绍的一个家伙DefaultDefaultDependencyHandler, 他是继承自 DependencyHandler 接口, 在DefaultProject中通过getDependencies()里获得代码如下:

/// in gradle script 
dependencies {
    implementation project(':java')
    api ...
}
///

/// in DefaultProject
public void dependencies(Closure configureClosure) {
    ConfigureUtil.configure(configureClosure, getDependencies());
}
///

DefaultDefaultDependencyHandler 是用来配置dependency, 在详细介绍之前，需要提到一个关键的角色:DefaultConfigurationContainer, DefaultConfigurationContainer 继承自 NamedDomainObjectContainer<Configuration>.
简单解释一下，NamedDomainObjectContainer<Configuration> 会对应一个delegate去创建任意名字的Configuration, 这个名字就是用户来定义, 比如:
///
  implementation project(':xx')
  api project(':xx')
///
这里的 implementation 和 api 就是Configuration的名字.



