# 【MVP】MVP与内存泄漏

# MVP模式为什么会存在内存泄漏的隐患

由上可见，Presenter中持有View接口对象，这个接口对象实际为MainActivity.this，Modle中也同时拥有Presenter对象实例，当MainActivity要销毁时，Presenter中有Modle在获取数据，那么问题来了，这个Activity还能正常销毁吗？ 

**答案是不能！** 

当Modle在获取数据时，不做处理，它就一直持有Presenter对象，而Presenter对象又持有Activity对象，这条GC链不剪断，Activity就无法被完整回收。                                

换句话说：Presenter不销毁，Activity就无法正常被回收。

## 解决MVP的内存泄露

Presenter在Activity的onDestroy方法回调时执行资源释放操作，或者在Presenter引用View对象时使用更加容易回收的软引用，弱应用。

Activity

```java
@Override
    public void onDestroy() {
        super.onDestroy();
        mPresenter.destroy();
        mPresenter = null;
    }

```

Presenter

```java
public void destroy() {
    view = null;
    if(modle != null) {
        modle.cancleTasks();
        modle = null;
    }
}
```

Model

```java
public void cancleTasks() {
    // TODO 终止线程池ThreadPool.shutDown()，AsyncTask.cancle()，或者调用框架的取消任务api
}
```



> [（一）Java 中的引用类型、对象的可达性以及回收处理大家应该都知道 Java 中除了强引用类型外还有几个特殊的引用类型 - 掘金](https://juejin.cn/post/6844903974391250957)
>
> [内存泄漏（一）MVP模式中的内存泄漏以及解决方案_安卓mvp处理内存泄漏-CSDN博客](https://blog.csdn.net/zhangjin1120/article/details/120941499)