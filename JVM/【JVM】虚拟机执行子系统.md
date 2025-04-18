# 【JVM】类文件结构

代码编译的结果从本地机器码转变为字节码，是存储格式发展的一小步，却是编程语言发展的一大步。

Java 虚拟机不与包括 Java 语言在内的任何程序语言绑定，它只与“Class 文件”这种特定的二进制文件格式所关联，Class 文件中包含了 Java 虚拟机指令集、符号表以及若干其他辅助信息。

# 【JVM】虚拟机类加载机制

![image-20250418010634077](https://gitee.com/ForeverHamburger/picgo_imgs1/raw/master/202504180106188.png)
