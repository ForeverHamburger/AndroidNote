# Kotlin

## 问题合集

### Java是解释型语言还是编译型语言

> 编译型语言的特点是编译器会将我们编写的源代码一次性地编译成计算机可识别的二进制文件，然后计算机直接执行，像C和C++都属于编译型语言。
>
> 解释型语言则完全不一样，它有一个解释器，在程序运行时，解释器会一行行地读取我们编写的源代码，然后实时地将这些源代码解释成计算机可识别的二进制数据后再执行，因此解释型语言通常效率会差一些，像Python和JavaScript都属于解释型语言。

**1. 编译阶段：从源码到字节码**

**编译型特征**：Java 源代码（`.java` 文件）通过 **javac 编译器**编译为 **字节码**（`.class` 文件）。

```
javac Main.java → 生成 Main.class（平台无关的字节码）
```

字节码是介于源代码和机器码之间的中间表示形式，类似汇编语言，但面向 JVM 而非物理硬件。

------

**2. 执行阶段：JVM 的混合执行模式**

**解释执行**：JVM 的 **字节码解释器** 逐行读取字节码并解释执行，此时表现出 **解释型语言** 的特征。

**即时编译（JIT）**：JVM 的 **JIT 编译器**（如 HotSpot）会监控运行时的热点代码（高频执行的代码块），将其 **动态编译为本地机器码**，后续直接执行机器码。

## 变量和函数

### 变量

val（value的简写）用来声明一个不可变的变量，这种变量在初始赋值之后就再也不能重新赋值，对应J应Java中的final变量。

var（variable的简写）用来声明一个可变的变量，这种变量在初始赋值之后仍然可以再被重新赋值，对应J应Java中的非final变量。

| Java 数据类型 | 包装类      | 大小（字节）                | Kotlin 数据类型 | 说明                                                         |
| ------------- | ----------- | --------------------------- | --------------- | ------------------------------------------------------------ |
| `byte`        | `Byte`      | 1                           | `Byte`          | 表示 8 位有符号整数，范围是 -128 到 127                      |
| `short`       | `Short`     | 2                           | `Short`         | 表示 16 位有符号整数，范围是 -32768 到 32767                 |
| `int`         | `Integer`   | 4                           | `Int`           | 表示 32 位有符号整数，范围是 -2147483648 到 2147483647       |
| `long`        | `Long`      | 8                           | `Long`          | 表示 64 位有符号整数，范围是 -9223372036854775808 到 9223372036854775807，Kotlin 中长整型字面量需加 `L` 后缀，如 `123L` |
| `float`       | `Float`     | 4                           | `Float`         | 表示 32 位单精度浮点数，Kotlin 中浮点数字面量需加 `f` 或 `F` 后缀，如 `1.23f` |
| `double`      | `Double`    | 8                           | `Double`        | 表示 64 位双精度浮点数，Kotlin 中默认浮点数字面量就是 `Double` 类型，如 `1.23` |
| `char`        | `Character` | 2                           | `Char`          | 表示 16 位 Unicode 字符，Kotlin 中字符用单引号括起来，如 `'A'` |
| `boolean`     | `Boolean`   | 未明确定义，通常认为是 1 位 | `Boolean`       | 表示布尔值，只有 `true` 和 `false` 两个值                    |

> 那么我们应该什么时候使用val，什么时候使用var呢？
>
> 永远优先使用val来声明一个变量，而当val没有办法满足你的需求时再使用var。这样设计出来的程序会更加健壮，也更加符合高质量的编码规范。

### 函数

```kotlin
fun methodName(param1: Int, param2: Int): Int {
	return 0 
}
```

### 函数语法糖

当一个函数中只有一行代码时，Kotlin允许我们不必编写函数体，可以直接将唯一的一行代码写在函数定义的尾部，中间用等号连接即可。比如我们刚才编写的largerNumber()函数就只有一行代码，于是可以将代码简化成如下形式：

```kotlin
fun largerNumber(num1: Int, num2: Int): Int = max(num1, num2)
```

使用这种语法，return关键字也可以省略了，等号足以表达返回值的意思。另外，还记得Kotlin出色的类型推导机制吗？在这里它也可以发挥重要的作用。由于max()函数返回的是一个Int值，而我们在largerNumber()函数的尾部又使用等号连接了max()函数，因此Kotlin可以推导出largerNumber()函数返回的必然也是一个Int值，这样就不用再显式地声明返回值类型了，代码可以进一步简化成如下形式：

```kotlin
fun largerNumber(num1: Int, num2: Int) = max(num1, num2)
```

## 程序逻辑控制

```kotlin
fun largerNumber(num1: Int, num2: Int): Int {
    var value = 0 if (num1 > num2) {
        value = num1 
    } else {
        value = num2 
    }
    return value 
}
```

Kotlin中的if语句相比于J于Java有一个额外的功能，它是可以有返回值的，返回值就是if语句每一个条件中最后一行代码的返回值。因此，上述代码就可以简化成如下形式：

```kotlin
fun largerNumber(num1: Int, num2: Int): Int {
    val value = if (num1 > num2) {
        num1 
    } else { 
        num2 
    } 
    return value 
}
```

value其实也是一个多余的变量，我们可以直接将if语句返回，这样代码将会变得更加精简：

```kotlin
fun largerNumber(num1: Int, num2: Int): Int {
    return if (num1 > num2) {
        num1 
    } else { 
        num2 
    } 
}
```

