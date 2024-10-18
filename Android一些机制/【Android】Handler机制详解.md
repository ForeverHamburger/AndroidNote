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







## SP: Handler内存泄漏的原因？

内部类持有外部类的引用。

### 1. **Handler持有外部类的隐式引用**

在Android中，通常会在一个`Activity`或`Fragment`中创建`Handler`，而非静态的`Handler`类会隐式地持有它所在的外部类（通常是`Activity`或者`Fragment`）的引用。这意味着如果`Handler`的消息或任务没有处理完，那么`Handler`对象会继续存在，并且它所持有的`Activity`的引用也不会被释放，导致`Activity`的内存无法及时回收，这就是**内存泄漏**。

```java
public class MyActivity extends AppCompatActivity {
    // 非静态 Handler
    private Handler handler = new Handler() {
        @Override
        public void handleMessage(Message msg) {
            // 处理消息
        }
    };
}
```

上面的代码中，`Handler`会持有`MyActivity`的引用，如果`Handler`在`MyActivity`销毁之后还有未处理的消息，`MyActivity`将无法被垃圾回收器回收，造成内存泄漏。

### 2. 如何解决Handler内存泄漏

#### **使用静态内部类 + 弱引用**

将`Handler`定义为静态内部类，并通过**弱引用**持有外部类的引用。这样可以避免`Handler`直接持有外部类的引用，防止内存泄漏。

```java
public class MyActivity extends AppCompatActivity {
    // 使用静态内部类
    private static class MyHandler extends Handler {
        private final WeakReference<MyActivity> mActivity;

        public MyHandler(MyActivity activity) {
            mActivity = new WeakReference<>(activity);
        }

        @Override
        public void handleMessage(Message msg) {
            MyActivity activity = mActivity.get();
            if (activity != null) {
                // 处理消息
            }
        }
    }

    private MyHandler handler = new MyHandler(this);
}
```

#### **在`onDestroy()`中移除所有未处理的消息和任务**

确保在`Activity`或`Fragment`销毁时，清除所有未处理的消息和任务，避免`Handler`继续持有外部类引用。

```java
@Override
protected void onDestroy() {
    super.onDestroy();
    handler.removeCallbacksAndMessages(null); // 移除所有的消息和任务
}
```

> ## 复习：**为何静态内部类加弱引用可以防止内存泄漏**
>
> #### **静态内部类与外部类的引用关系**
>
> - **普通内部类**：普通的内部类会隐式地持有外部类的引用。当你在一个`Activity`或`Fragment`中定义一个`Handler`，该`Handler`是其内部类的一部分，编译器会自动将外部类的实例引用传递给内部类对象。这意味着即使外部类（如`Activity`）销毁了，只要内部类（如`Handler`）依然存在，它就会持有对外部类的引用，导致外部类无法被垃圾回收，从而造成内存泄漏。
> - **静态内部类**：静态内部类不持有外部类的引用。因为静态类不依赖于其外部类的实例，它不包含对外部类的隐式引用。因此，静态内部类不会因为保留外部类的引用而导致内存泄漏。
>
> #### **弱引用（WeakReference）**
>
> Java中的`WeakReference`允许你持有一个对象的“弱引用”，这意味着如果垃圾回收器（GC）检测到一个只有弱引用的对象，并且没有其他强引用存在，GC就会回收该对象。这样即使静态内部类中的`Handler`持有外部类的弱引用，也不会阻止外部类（如`Activity`）的回收。
>
> 当我们将静态内部类与弱引用结合使用时，即使`Handler`有一个指向`Activity`的弱引用，在`Activity`被销毁时，GC仍然可以回收`Activity`对象，不会造成内存泄漏。

| **特性**         | **强引用（Strong Reference）**     | **弱引用（Weak Reference）**                      |
| ---------------- | ---------------------------------- | ------------------------------------------------- |
| **垃圾回收**     | 永远不会被GC回收，除非主动断开引用 | 如果只有弱引用指向该对象，GC会在合适时回收它      |
| **常见使用场景** | 一般用途的对象引用（常规变量引用） | 缓存、避免内存泄漏的场景（如`Handler`）           |
| **访问方式**     | 直接访问对象                       | 通过`WeakReference.get()`访问对象，可能返回`null` |
| **内存管理**     | 对象始终占用内存                   | 当对象没有强引用时，可以被回收，释放内存          |
| **适合场景**     | 任何需要确保对象存活的场景         | 需要允许GC回收、或者防止内存泄漏的场景            |



## SP: Handler阻塞机制为什么不会导致ANR

### ANR是什么？

> ANR是“Application Not Responding”的缩写，中文意思是“应用程序无响应”。在Android系统中，当一个应用程序在特定时间内没有响应用户的输入事件（如按键、触摸屏操作）或者没有在规定的时间内完成某些操作（如广播处理、服务操作），系统就会认为该应用程序发生了ANR，进而弹出一个对话框提示用户是否要关闭该应用程序。
>
> ANR主要发生在以下几种情况：
>
> 1. 主线程（UI线程）被长时间占用：如果主线程执行耗时的操作，如网络请求、大计算量的任务等，导致无法及时处理用户的交互事件，就可能触发ANR。
> 2. 广播接收器处理时间过长：如果一个广播接收器在执行`onReceive()`方法时耗时过长（通常超过10秒），系统也会认为发生了ANR。
> 3. 服务操作超时：如果服务（Service）中的操作耗时过长，特别是在前台服务（Foreground Service）中，也可能导致ANR。

## SP: 享元模式与Handler

在Android的**Handler机制**中。享元模式的主要目标是共享对象以减少内存开销，适用于大量相似对象的场景。Handler机制中的`Message`对象使用了享元模式。

- **享元模式**通过共享相同的对象来避免重复创建类似对象，减少内存消耗。
- 享元模式将对象分为**内部状态**（可以共享的部分）和**外部状态**（因使用情景不同而变化的部分）。
- 享元模式通常用于优化大量对象的内存使用，比如池化技术（Object Pool）。

### **Message对象的复用：享元模式的体现**

在Android中，`Message`对象本质上是一个轻量级的消息载体，用于在不同线程之间传递数据或标识操作类型。由于`Message`对象在应用中可能被频繁创建和销毁，直接频繁创建新的`Message`实例会造成不必要的内存开销。因此，Android对`Message`对象的创建进行了优化，通过享元模式来**复用**`Message`对象。

### **Message.obtain()方法**

`Message.obtain()`方法用于获取`Message`对象。在调用`obtain()`时，系统不会总是创建新的`Message`对象，而是会尝试从`Message`的**对象池**（即享元池）中获取一个空闲的`Message`实例。如果池中有可用的`Message`，则返回该实例；如果没有可用的实例，则创建一个新的`Message`。

