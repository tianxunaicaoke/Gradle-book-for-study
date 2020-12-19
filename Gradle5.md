# Gradle 进阶 第六篇

有志者，事竟成

## Gradle Project
在之前的一系列文章中，我们从一些方面了解了 Gradle，从这篇开始，我们会系统的把 Gradle 的重要组件作以讲解。
首先讲解 Project，我在前文有大量讲解关于".gradle"文件，Project 就是和"build.gradle"一一对应。多模块项目往往一个模块就对应一个 Project，之前的文章讲解了 ".gradle"文件的编译和解析，这篇文章就用一个栗子，细细品味一下 Gradle 的设计。

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

task clean(type: Delete) {
    delete rootProject.buildDir
}
~~~
以上栗子里的每一个代码块，对应下面一个小章节。

## plugins { ... } 解析
上一章讲的就是 plugin 的管理，就像我在栗子里加的分割线，在分割线之上的会先编译，并且运行。在运行的过程中会生成对于 android plugin 的请求，并查找，最后把 Plugin 对应的路径加入 Gradle 的相应的 ClassLoader 的 path 里，这样在代码里就可以真正的调用到 Plugin 对应的 apply 方法。
我在这里直接就展示出 android plugin 的 apply 方法。
~~~

~~~
## android { ... } 解析
## dependencies { ... } 解析
## task clean ... 解析