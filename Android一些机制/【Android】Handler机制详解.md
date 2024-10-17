# 【Android】Handler机制详解

Android的**Handler机制**是Android系统中处理异步消息的重要工具，常用于在主线程（UI线程）和后台线程之间进行通信。它的核心是消息传递机制，通过发送和处理消息来实现线程间的任务调度和交互。Handler机制基于消息队列和消息循环，帮助开发者避免在非UI线程直接操作UI组件，从而避免线程安全问题。

## 1. **Handler机制的核心组件**

Handler机制主要由以下四个核心组件组成：

- **Handler**：处理消息和Runnable的工具，用于发送消息（`Message`）或执行任务（`Runnable`）到消息队列中，并在特定的线程上处理这些消息。
- **Message**：消息对象，用于封装从一个线程发送到另一个线程的数据。
- **MessageQueue**：消息队列，用来存储通过Handler发送的消息或任务，队列中的消息按顺序被处理。
- **Looper**：消息循环器，它不断地从消息队列中取出消息并分发给对应的Handler来处理。每个线程最多只能有一个Looper。

## 2. Handler的工作流程

**初始化Looper和MessageQueue**：在主线程中，Looper和MessageQueue默认已经初始化并开始运行。在其他线程中，需要手动调用Looper.prepare()来初始化Looper，并调用Looper.loop()来开始消息循环。

**创建Handler**：创建一个Handler对象，并将其关联到当前线程的Looper和MessageQueue。

**发送消息**：通过Handler的sendMessage()或post()方法将消息或Runnable对象发送到MessageQueue。

**处理消息**：Looper从MessageQueue中取出消息，并调用Handler的handleMessage()方法或执行Runnable对象。



## Handler阻塞机制为什么不会导致ANR







### ANR是什么？

> ANR是“Application Not Responding”的缩写，中文意思是“应用程序无响应”。在Android系统中，当一个应用程序在特定时间内没有响应用户的输入事件（如按键、触摸屏操作）或者没有在规定的时间内完成某些操作（如广播处理、服务操作），系统就会认为该应用程序发生了ANR，进而弹出一个对话框提示用户是否要关闭该应用程序。
>
> ANR主要发生在以下几种情况：
>
> 1. 主线程（UI线程）被长时间占用：如果主线程执行耗时的操作，如网络请求、大计算量的任务等，导致无法及时处理用户的交互事件，就可能触发ANR。
> 2. 广播接收器处理时间过长：如果一个广播接收器在执行`onReceive()`方法时耗时过长（通常超过10秒），系统也会认为发生了ANR。
> 3. 服务操作超时：如果服务（Service）中的操作耗时过长，特别是在前台服务（Foreground Service）中，也可能导致ANR。

## 享元模式与Handler

在Android的**Handler机制**中。享元模式的主要目标是共享对象以减少内存开销，适用于大量相似对象的场景。Handler机制中的`Message`对象使用了享元模式。

- **享元模式**通过共享相同的对象来避免重复创建类似对象，减少内存消耗。
- 享元模式将对象分为**内部状态**（可以共享的部分）和**外部状态**（因使用情景不同而变化的部分）。
- 享元模式通常用于优化大量对象的内存使用，比如池化技术（Object Pool）。

### **Message对象的复用：享元模式的体现**

在Android中，`Message`对象本质上是一个轻量级的消息载体，用于在不同线程之间传递数据或标识操作类型。由于`Message`对象在应用中可能被频繁创建和销毁，直接频繁创建新的`Message`实例会造成不必要的内存开销。因此，Android对`Message`对象的创建进行了优化，通过享元模式来**复用**`Message`对象。

### **Message.obtain()方法**

`Message.obtain()`方法用于获取`Message`对象。在调用`obtain()`时，系统不会总是创建新的`Message`对象，而是会尝试从`Message`的**对象池**（即享元池）中获取一个空闲的`Message`实例。如果池中有可用的`Message`，则返回该实例；如果没有可用的实例，则创建一个新的`Message`。
