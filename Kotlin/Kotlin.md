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

val（value的简写）用来声明一个**不可变**的变量，这种变量在初始赋值之后就再也不能重新赋值，对应J应Java中的final变量。

var（variable的简写）用来声明一个**可变**的变量，这种变量在初始赋值之后仍然可以再被重新赋值，对应J应Java中的非final变量。

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

### if条件语句

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

### when条件语句

例如定义一个函数，接收一个学生姓名参数，再用if条件语句找到对应的分数返回。

```kotlin
fun getScore(name: String) = if (name == "Tom") { 
    86 
} else if (name == "Jim") {
    77 
} else if (name == "Jack") {
    95 
} else if (name == "Lily") { 
    100 
} else {
    0 
}
```

用when语句的话，就可以改写成这样：

```kotlin
fun getScore(name: String) = when (name) {
    "Tom" -> 86 
    "Jim" -> 77 
    "Jack" -> 95 
    "Lily" -> 100 
    else -> 0 
}
```

when语句允许传入一个任意类型的参数，然后可以在when的结构体中定义一系列的条件，格式是：

```
匹配值 -> { 执行逻辑 }
```

当执行逻辑只有一行代码时，{ }可以省略。

除了精确匹配之外，when语句还允许进行类型匹配。

```kotlin
fun checkNumber(num: Number) { 
    when (num) { 
        is Int -> println("number is Int") 
        is Double -> println("number is Double") 
        else -> println("number not support") 
    } 
}
```

is关键字就是类型匹配的核心，相当于Java中的instanceof关键字。

由于checkNumber()函数接收一个Number类型的参数，这是Kotlin内置的一个抽象类，像Int、Long、Float、Double等与数字相关的类都是它的子类，所以这里就可以使用类型匹配来判断传入的参数到底属于什么类型，如果是Int型或Double型，就将该类型打印出来，否则就打印不支持该参数的类型。

when语句还有一种不带参数的用法，虽然这种用法可能不太常用，但有的时候却能发挥很强的扩展性。

例如：

```kotlin
fun getScore(name: String) = when { 
	name == "Tom" -> 86 
    name == "Jim" -> 77 
    name == "Jack" -> 95
    name == "Lily" -> 100 
    else -> 0 
}
```

> Kotlin中判断字符串或对象是否相等可以直接使用==关键字，而不用像J像Java那样调用equals()方法。

适用场景：

> 假设所有名字以Tom开头的人，他的分数都是86分，这种场景如果用带参数的when语句来写就无法实现，而使用不带参数的when语句就可以这样写。

```kotlin
fun getScore(name: String) = when { 
    name.startsWith("Tom") -> 86 
    name == "Jim" -> 77 
    name == "Jack" -> 95 
    name == "Lily" -> 100 
    else -> 0 
}
```

### 循环语句

区间的概念：

```kotlin
val range = 0..10
```

有了区间之后，我们就可以通过for-in循环来遍历这个区间，比如在main()函数中编写如下代码：

```kotlin
fun main() {
    for (i in 0..10) {
        println(i) 
    }
}
```

Kotlin中可以使用until关键字来创建一个左闭右开的区间，如下所示：

```kotlin
val range = 0 until 10
```

上述代码表示创建了一个0到10的左闭右开区间，它的数学表达方式是[0, 10)。修改main()函数中的代码，使用until替代..关键字。

默认情况下，for-in循环每次执行循环时会在区间范围内递增1，相当于Java for-i循环中i++的效果，而如果你想跳过其中的一些元素，可以使用step关键字：

```kotlin
fun main() { 
	for (i in 0 until 10 step 2) { 
		println(i) 
	} 
}
```

上述代码表示在遍历[0, 10)这个区间的时候，每次执行循环都会在区间范围内递增2，相当于for-i循环中i = i + 2的效果。

如果你想创建一个降序的区间，可以使用downTo关键字，用法如下：

```kotlin
fun main() { 
    for (i in 10 downTo 1) { 
        println(i) 
    } 
}
```

