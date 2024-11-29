# 【Android】View事件分发

> 本文参考：
>
> [【Android】图解View事件分发机制_android view 事件分发-CSDN博客](https://blog.csdn.net/weixin_73871834/article/details/136888568?spm=1001.2014.3001.5502)
>
> 《Android进阶之光》

## Activity的构成









Android的事件分发机制是 UI 系统中处理用户交互（如触摸、滑动、点击等）的核心机制。理解它对于解决滑动冲突、优化交互体验非常重要。事件分发机制主要通过三个核心方法实现：**`dispatchTouchEvent`**、**`onInterceptTouchEvent`** 和 **`onTouchEvent`**。
