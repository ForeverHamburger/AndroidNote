# 【Android】这些自带的代码or文件是干啥的？？

## build.gradle 文件

```java
buildscript {
    repositories {
        mavenCentral()//这行依赖
    }
}
allprojects {
    repositories {
        mavenCentral()//这行依赖
    }
}
```

> `buildscript` 和 `allprojects` 块通常出现在使用 Gradle 构建系统的 Android 项目的 `build.gradle` 文件中。

- `buildscript` 代码块用于配置构建脚本本身所需的依赖项和插件。在 `buildscript` 块中添加 `mavenCentral()` 表示构建脚本将从 Maven Central 仓库获取依赖。
- `allprojects` 代码块用于配置项目中所有模块（包括主模块和任何子模块）的通用设置。在 `allprojects` 块中添加 `mavenCentral()` 表示所有模块都将从 Maven Central 仓库获取依赖。

因此，这两行代码放在一起，意味着构建脚本本身和项目中的所有模块都将能够从 Maven Central 仓库获取它们各自所需的依赖项。Maven Central 是一个广泛使用的 Maven 仓库，提供了大量的开源项目和库的依赖。这样的配置使得依赖管理更加集中和高效。
