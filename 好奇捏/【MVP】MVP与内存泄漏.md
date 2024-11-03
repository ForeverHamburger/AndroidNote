# 【MVP】MVP与内存泄漏

在Android开发中，MVP（Model-View-Presenter）模式是一种常见的架构模式，它将应用的逻辑分为三个部分：Model（模型），View（视图），和Presenter（呈现器）。这种模式有助于分离关注点，提高代码的可维护性和可测试性。然而，如果不正确地管理这些组件之间的引用，MVP模式可能会导致内存泄漏，影响应用的性能和稳定性。

# MVP模式为什么会存在内存泄漏的隐患

在MVP模式中，Presenter扮演着协调者的角色，它持有View接口的引用（通常是Activity或Fragment的实例），并且与Model进行交互以获取数据。Model层负责处理业务逻辑和数据操作，它可能需要异步任务来从网络或数据库获取数据。问题在于，如果Model在执行这些异步任务时没有正确地管理生命周期，它可能会一直持有Presenter的引用，而Presenter又持有Activity的引用。这形成了一个引用链，阻止了垃圾回收（GC）的进行，导致Activity无法被正常销毁，从而产生内存泄漏。

考虑以下情况：当用户离开一个Activity时，该Activity应该被销毁以释放资源。但是，如果Model层正在执行一个长时间运行的任务，并且这个任务持有Presenter的引用，那么即使用户已经离开了Activity，这个Activity也无法被垃圾回收器回收。这是因为Model层的异步任务阻止了Presenter的销毁，而Presenter又阻止了Activity的销毁。              

换句话说：Presenter不销毁，Activity就无法正常被回收。

## 解决MVP的内存泄露

为了避免这种内存泄漏，我们需要确保在Activity销毁时，正确地释放Presenter和Model中的资源。以下是一些常见的解决方案：

**在Activity的onDestroy方法中释放资源**： 在Activity的生命周期结束时，我们应该释放Presenter的引用，并调用其销毁方法来清理资源。

```java
@Override
public void onDestroy() {
    super.onDestroy();
    if (mPresenter != null) {
        mPresenter.destroy();
    }
    mPresenter = null;
}
```

**在Presenter中清理资源**： Presenter应该提供一个方法来释放View和Model的引用，以及取消所有正在进行的异步任务。

```java
public void destroy() {
    if (view != null) {
        view = null;
    }
    if (modle != null) {
        modle.cancleTasks();
        modle = null;
    }
}
```

**在Model中取消异步任务**： Model应该提供一个方法来取消所有正在进行的异步任务，例如网络请求或数据库操作。

```java
public void cancleTasks() {
    // 终止线程池ThreadPool.shutDown()，AsyncTask.cancle()，或者调用框架的取消任务api
}
```

**使用弱引用**： 另一种方法是在Presenter中使用弱引用来持有View的引用。这样，当Activity被销毁时，它的引用会被垃圾回收器回收，即使Presenter仍然持有这个引用。

```java
private WeakReference<View> viewRef;

public void setView(View view) {
    this.viewRef = new WeakReference<>(view);
}

public View getView() {
    return viewRef.get();
}
```

**生命周期意识**： 确保Model层的异步任务能够感知到Activity的生命周期。如果任务在Activity销毁后完成，它不应该尝试更新UI或与Activity交互。

## 结语

参考：

> [（一）Java 中的引用类型、对象的可达性以及回收处理大家应该都知道 Java 中除了强引用类型外还有几个特殊的引用类型 - 掘金](https://juejin.cn/post/6844903974391250957)
>
> [内存泄漏（一）MVP模式中的内存泄漏以及解决方案_安卓mvp处理内存泄漏-CSDN博客](https://blog.csdn.net/zhangjin1120/article/details/120941499)