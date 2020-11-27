# gradle 是什么呢

本文是针对使用过Gradle的一些年轻人哈，看了之后，耗子尾汁。

gradle 的源代码地址 https://github.com/gradle/gradle 我们可以看到gradle的源码里（基于 gradle 大版本的 version 6）java 占比44% groovy 占比46%, 源码里面大部分的核心代码核心模块都是java 语言编写,test 代码主要是由groovy语言编写。
<img src="language.PNG"/>

往往高端的代码都以一种朴素的呈现方式，以gradle-6.3-all 为例, 解压之后目录结构如下：
***bin
*****gradle(unix and linux 启动脚本)
*****gradle.bat(windows 启动脚本)
***docs
***init.d(自定义的init.gradle 位置)
***lib(编译好的jar包)
***src(源码)
**LICENSE
**NOTICE
**README

启动Gradle就是从bin 目录下的启动脚本文件触发到lib目录下的jar里的某个java的main函数。以gradle.bat 为例, 其中最终调用的位置如下：入口类是org.gradle.launcher.GradleMain。

"%JAVA_EXE%" %DEFAULT_JVM_OPTS% %JAVA_OPTS% %GRADLE_OPTS% "-Dorg.gradle.appname=%APP_BASE_NAME%" -classpath "%CLASSPATH%" org.gradle.launcher.GradleMain %CMD_LINE_ARGS%

如果我们这个时候继续跟着这个GradleMain深入阅读的话，快放弃这个牛(愚)逼(蠢)的决定。因为代码的细节对于入门Gradle的猿来说就像马老师的打五鞭一样，没有一定的内力，一般是看不懂。

那我们继续换个方向来再去open Gradle 吧。".gradle" 文件是一个很好的入手点：
~~~groovy
plugins {
 id 'groovy'
 id 'java-library'
}

group 'com.example'
version '1.1-SNAPSHOT'

allprojects {
    repositories {
        mavenCentral()
    }
}
...
dependencies {
    compile 'org.codehaus.groovy:groovy-all:2.3.11'
  ...
}
~~~

 看到这个文件很熟悉但是肯定有很多疑问，比如 plugins{xxx} 干什么了，谁来编译它或者解释它给机器呢等等，为了能解答这个问题，需要好好了解一下groovy。我们从很多地方都能知道".gradle" 文件其实就是groovy的的脚本文件。在这里我简单的以java程序员可以理解的方式解释一下plugins{}, 在gradle代码中有一个地方有一个java函数它的名字是plugins, 它的参数是一个闭包：
~~~
  public void plugins(Closure<?> closure){

  }
~~~
 什么是闭包请找代驾：http://groovy-lang.org/closures.html 

我要强调一点哈，Groovy 是基于JVM, 所以呢， 我们的这个脚本文件也是会像java文件一样被编译成.class 文件， 然后被加到jvm虚拟机里运行。
所以我们就可以从网络上继续获取groovy 脚本的运行原理, 以及groovy 作为DSL的支柱: 元对象协议（Meta Object Protocol）简称MOP, 基于这个协议就有了运行时和编译时的两种 metaprogramming, 细节和干货都在这个链接里面 http://groovy-lang.org/metaprogramming.html。

如果上面的链接你都看完了, 那你的英语应该过了六级。回正题，我们要了解".gradle"文件的运行。我先把脚本文件编译出来的class 文件show 出来。
~~~
....
public class build_84d5gswk7zgm2rs9c1flrc0wd extends ProjectScript{
....
    public Object run()
    {
        CallSite acallsite[] = $getCallSiteArray();
       
         public final class _run_closure1 extends Closure
            implements ...
         {
             ....
            public Object doCall(Object it)
            {
             ....
            }
         }
    }

     private static void $createCallSiteArray_1(String as[)
    {
        as[0] = "google";
        as[1] = "jcenter";
        as[2] = "mavenLocal";
    }

}
~~~

