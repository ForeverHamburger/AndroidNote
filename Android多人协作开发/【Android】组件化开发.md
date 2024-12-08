# 【Android】组件化开发

## 组件分层

在学习组件化开发之前，首先要理解组件分层的概念。在Android应用的开发中，组件化的目的是将应用拆分成多个独立的模块，便于管理、开发、调试和维护。下面是常见的组件分层结构：

![image-20241204133223315](https://gitee.com/ForeverHamburger/picgo_imgs1/raw/master/202412041332466.png)

> **主工程（App）**:
>
> 只依赖各业务组件，通过配置指定依赖哪些业务组件。
>
> 除了一些全局的配置和主Activity之外，不包含任何业务代码，是应用的入口。
>
> **业务组件层**:
>
> 各组件之间无直接关联，通过路由进行通信。
>
> 可直接依赖基础组件层，同时也能依赖公用的一些功能组件。
>
> 业务组件可以在library和application之间切换，但是最后打包时必须是library。
>
> **功能组件层**:
>
> 依赖基础组件层。
>
> 对一些公用的功能业务进行封装与实现。
>
> 包括支付组件、分享组件等，这些组件可以被业务组件层调用。
>
> **基础组件层**:
>
> 封装公用的基础组件（并且实现了路由）。
>
> 包括网络框架、图片加载框架等主流的第三方库。
>
> 各种第三方的SDK，如Retrofit（网络请求）、RxJava（响应式编程）、Glide（图片加载）等。

### 组件化开发问题点

> 1. 业务组件，如何实现单独运行调试？
> 2. 业务组件间 没有依赖，如何实现页面的跳转？
> 3. 业务组件间 没有依赖，如何实现组件间通信/方法调用？
> 4. 业务组件间 没有依赖，如何获取fragment实例？
> 5. 业务组件不能反向依赖壳工程，如何获取Application实例、如何获取Application onCreate()回调（用于任务初始化）？

> **如何实现业务组件的单独调试？** 由于业务组件是独立开发的，它不包含主App的代码，因此需要特别的配置来保证可以单独运行和调试。
>
> **如何实现组件间的跳转？** 业务组件之间通常没有直接的依赖关系，因此需要通过路由来实现页面跳转，如ARouter。
>
> **如何实现组件间的通信或方法调用？** 业务组件间不能直接依赖，因此需要借助中介模式、事件总线（EventBus）或其他跨组件通信机制。
>
> **如何获取Fragment实例？** 每个组件都有独立的Fragment实例，但业务组件间不直接依赖，因此需要使用FragmentManager等机制来实现Fragment的共享。
>
> **如何获取Application实例？** 由于业务组件不能反向依赖主工程，因此需要通过合适的方式在组件内部获取Application实例，通常是通过全局的Context或者使用依赖注入来实现。

## 组件化项目搭建

**application**插件：如果一个模块被声明为application，那么它会成为一个apk文件，是可以直接安装运行的项目。

**library**插件：被声明为library的模块，它会成为一个aar文件，不可以单独运行。

### 抽取gradle版本数据，统一配置文件

首先需要创建一个`config.gradle`文件，用于全局管理版本号、依赖项等。这样可以避免项目中不同模块间依赖版本不一致的问题。

![image-20241208213313322](https://gitee.com/ForeverHamburger/picgo_imgs1/raw/master/202412082133391.png)

```java
ext {
    // false: 集成模式
    // true ：组件模式
    isModule = false
    android = [
            compileSdkVersion: 34,
            minSdkVersion    : 28,
            targetSdkVersion : 34,
            versionCode      : 1,
            versionName      : "1.0",
            applicationId : "com.xuptggg.teaswhisper"
    ]

    applicationId = ["app"  : "com.xuptggg.teaswhisper"]

    dependencies = [
            glide : 'com.github.bumptech.glide:glide:4.16.0',
            eventbus : 'org.greenrobot:eventbus:3.3.1',
            gson : 'com.google.code.gson:gson:2.10.1'
    ]

    libARouter = 'com.alibaba:arouter-api:1.5.2'
    libARouterCompiler = 'com.alibaba:arouter-compiler:1.5.2'
    libEventBusCompiler = 'org.greenrobot:eventbus-annotation-processor:3.3.1'
}
```

然后再project的build.gradle中引入config.gradle文件：

<img src="https://gitee.com/ForeverHamburger/picgo_imgs1/raw/master/202412082146202.png" alt="image-20241208214648119" style="zoom:50%;" />

### 基础组件创建以及配置build.gradle文件

创建基础组件（如`libBase`模块），并在`build.gradle`中配置好依赖，确保基础组件依赖全局的第三方库。

#### 创建基础组件Module

接下来我们创建Base组件：

![image-20241208212712662](https://gitee.com/ForeverHamburger/picgo_imgs1/raw/master/202412082127745.png)

第一个选项是创建可以单独运行的模块，也就是业务模块，是一个application。

第二个是创建一个Library，一般是用于创建基础层，不需要单独运行。

![image-20241208215032140](https://gitee.com/ForeverHamburger/picgo_imgs1/raw/master/202412082150192.png)

在这里我们创建Module name时多加了一层，这样可以用一个专门的包去管理所有的模块。

就是这样的效果：

![image-20241208215158875](https://gitee.com/ForeverHamburger/picgo_imgs1/raw/master/202412082151932.png)

创建Package name时，也多加了一个module,这个也是为了放在冲突，因为默认的这种创建方式的包名和主App的命名方式一致，导包的过程就会出现冲突。当然这个module可以根据自己的情况命名。

#### 修改libbase的builde.gradle

```java
plugins {
    alias(libs.plugins.android.library)
}

def cfg = rootProject.ext

android {
    namespace cfg.applicationId.base
    compileSdk cfg.android.compileSdkVersion

    defaultConfig {
        minSdk cfg.android.minSdkVersion

        testInstrumentationRunner "androidx.test.runner.AndroidJUnitRunner"
        consumerProguardFiles "consumer-rules.pro"
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
    api cfg.dependencies.glide
    api cfg.dependencies.eventbus
    api cfg.dependencies.gson
    api cfg.libARouter


    implementation libs.appcompat
    implementation libs.material
    testImplementation libs.junit
    androidTestImplementation libs.ext.junit
    androidTestImplementation libs.espresso.core
}
```

基础组件依赖全局提供的第三方库和SDK。其他业务组件和功能组件只要以来基础组件就能使用。

通常需要添加依赖的时候，会在config.gradle文件中添加全局变量，然后在libBase(基础组件)中去依赖，这样都可以使用新添加的依赖，对依赖的包进行统一管理。



### 业务组件创建以及配置build.gradle文件

每个业务组件模块都将包含自己的业务逻辑。在创建业务组件时，需要根据项目的需求，添加适当的依赖关系和配置。

#### 创建业务组件Module

选中项目右击，New->Module。

![image-20241208221605001](https://gitee.com/ForeverHamburger/picgo_imgs1/raw/master/202412082216131.png)

**业务组件都选择Phone&Tablet。**

为了分层更加明显，在Module name中加入了moduleCore，业务组件都放入这个moduleCore文件夹下。在Package name中加入了module是为了避免命名冲突。

之后Next->Finish完成创建。

同样的方法再创建login业务组件。直接右击moduleCore文件夹创建新的Module。

![image-20241208222341729](https://gitee.com/ForeverHamburger/picgo_imgs1/raw/master/202412082223769.png)

![image-20241208222349669](https://gitee.com/ForeverHamburger/picgo_imgs1/raw/master/202412082223703.png)

#### 修改login的builde.gradle

业务组件的`build.gradle`文件示例如下：

![image-20241208222744943](https://gitee.com/ForeverHamburger/picgo_imgs1/raw/master/202412082227950.png)

```java
//plugins {
//    alias(libs.plugins.android.application)
//}
def cfg = rootProject.ext
if (isModule.booleanValue()) {
    apply plugin: 'com.android.application'
} else {
    apply plugin: 'com.android.library'
}
```

![image-20241208223006074](https://gitee.com/ForeverHamburger/picgo_imgs1/raw/master/202412082230127.png)

将这些配置都改成ext中定义的全局参数。如果业务组件在组件化开发过程中，需要作为application单独运行就需要指定applicationId。否则不需要。

![image-20241208223318513](https://gitee.com/ForeverHamburger/picgo_imgs1/raw/master/202412082233569.png)

业务组件需要依赖基础组件，他要用的第三方库的依赖通过基础组件去依赖。

implementation project(':文件夹:组件名')

### 功能组件创建以及配置build.gradle文件

功能组件也同样不会单独作为一个application去运行，他和基础组件一样作为library配置。

#### 创建业务组件Module

仍然与基础组件是一样的步骤，不多赘述。

#### 修改moduleUltrasonic的builde.gradle

![image-20241208223912928](https://gitee.com/ForeverHamburger/picgo_imgs1/raw/master/202412082239989.png)

别忘了点击同步。

![image-20241208224001829](https://gitee.com/ForeverHamburger/picgo_imgs1/raw/master/202412082240876.png)

功能组件和业务组件一样**通过依赖基础组件去使用第三方库**。

![image-20241208223945262](https://gitee.com/ForeverHamburger/picgo_imgs1/raw/master/202412082239364.png)

## app配置build.gradle文件

app是一个特殊的module。他作为应用的启动层，**一定是一个application**。

基本和刚才login组件的修改方法相似，plugins不用改动，因为App层不会拿来当作library。

![image-20241208224318009](https://gitee.com/ForeverHamburger/picgo_imgs1/raw/master/202412082243101.png)

需要注意的就是`dependencies`。同样的app也需要导入基础模块的依赖，并且app是关联其他的多个组件，这里还需要导入其他组件例如：login

因此在这里也需要判断是否是debug模式，如果是debug模式就不需要导入这些组件。

![image-20241208224446587](https://gitee.com/ForeverHamburger/picgo_imgs1/raw/master/202412082244623.png)

完成的修改格式如下：

```java
plugins {
    alias(libs.plugins.android.application)
}
def cfg = rootProject.ext


android {
    namespace 'com.xuptggg.teaswhisper'
    compileSdk cfg.android.compileSdkVersion

    defaultConfig {
        applicationId cfg.android.applicationId
        minSdk cfg.android.minSdkVersion
        targetSdk cfg.android.targetSdkVersion
        versionCode cfg.android.versionCode
        versionName cfg.android.versionName

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
    implementation project(':modulesBase:libBase')

    if (!isModule.booleanValue()) {
        implementation project(':modulesCore:login')
        implementation project(':modulesCore:guidepage')
        implementation project(':modulesCore:navigation')
    }

    testImplementation libs.junit
    androidTestImplementation libs.ext.junit
    androidTestImplementation libs.espresso.core
}
```

## 组件之间AndroidManifest合并问题

每个组件通常都会有自己的`AndroidManifest.xml`文件，这对于声明权限、`Application`类、`Activity`等是必须的。然而，在组件化开发时，多个组件的`AndroidManifest.xml`文件可能会发生冲突，特别是在`Application`和`Activity`的声明上。

#### 解决办法：

**创建不同的Manifest文件**：通过在每个组件下创建`release`目录，并将该目录中的`AndroidManifest.xml`文件指定为集成模式下的Manifest文件。这样，开发和集成时使用不同的配置。

**条件性加载Manifest文件**：通过Gradle配置，根据是否是组件化模式或集成模式来加载不同的Manifest文件。

![image-20241208225121074](https://gitee.com/ForeverHamburger/picgo_imgs1/raw/master/202412082251152.png)

我们在一个组件下面的main新建一个**release文件夹**表示**集成开发时**Androidmanifest.xml的路径。

<img src="https://gitee.com/ForeverHamburger/picgo_imgs1/raw/master/202412082251381.png" alt="image-20241208225146281" style="zoom:50%;" />

右击此文件夹，New-->Other-->Android Manifest File。

检查AndroidManifest中的内容。

因为我们通过New Module创建出来的业务组件自带的AndroidManifest.xml文件是该组件作为一个Application单独运行时使用的。我们创建的release文件夹下的AndroidManifest.xml文件是集成模式下的。它不需要app壳工程中指定的属性，否则会重复冲突。

![image-20241208225346380](https://gitee.com/ForeverHamburger/picgo_imgs1/raw/master/202412082253485.png)

在业务组件的build.gradle文件中指定AndroidManifest.xml的路径。

![image-20241208225556355](https://gitee.com/ForeverHamburger/picgo_imgs1/raw/master/202412082255463.png)

## 检验方法

当处于集成模式时，业务组件无法单独启动。

![image-20241208224819559](https://gitee.com/ForeverHamburger/picgo_imgs1/raw/master/202412082248610.png)

当处于测试模式时，可以单独启动。

![image-20241208225712138](https://gitee.com/ForeverHamburger/picgo_imgs1/raw/master/202412082257242.png)

## 结语

完全没写ARouter的一篇组件化教程，主要是防止笔者自己搭建的时候忘掉东西，就暂且记在这里啦。

如果有任何错误欢迎指正
