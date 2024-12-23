# 【Android】BuildConfig类找不到或无法生成最最终解决方案

## 缘起

由于新版本的Gradle插件进行了更新，导致新的Android项目中BuildConfig类不会自动生成。

> 从 Android Gradle 插件 7.0.0 开始，出于性能优化的考虑，默认情况下不再自动生成 BuildConfig 类。这是为了加快构建速度，特别是在大型项目中。所以如果我们需要主动生成`BuildConfig`类，有几种方法可以重新启用它。

闲话少说，直接上解决办法。

## 解决

### 显式启用BuildConfig

在gradle.properties文件中加一句（根目录的）：

```
android.buildFeatures.buildConfig=true
```

![image-20241221231744148](https://gitee.com/ForeverHamburger/picgo_imgs1/raw/master/202412212322461.png)



或者：

在模块级build.gradle文件中添加配置：

```
android {
    buildFeatures {
        buildConfig  true
    }
}
```

![image-20241221231805087](https://gitee.com/ForeverHamburger/picgo_imgs1/raw/master/202412212322076.png)

### 添加自定义字段

例如：

```
buildConfigField("boolean","isModule",String.valueOf(cfg.isModule))
```

![image-20241221231842350](https://gitee.com/ForeverHamburger/picgo_imgs1/raw/master/202412212322668.png)

## 如解

但是，笔者在尝试的过程中，居然试了上面俩方法发现都不行！！！

而且项目中甚至没有出现build文件夹！！！

如果发现模块中缺失了build文件夹，可以在命令行中输入：

```
./gradlew build
```

然后就可以正常用啦

## 结语

本文参考：

> [新版本 Android Studio 没有BuildConfig ？_android buildconfig-CSDN博客](https://blog.csdn.net/u013762572/article/details/140484589)
