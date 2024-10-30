# 【Android】滑动冲突解决方案：NestedScrollView

**[NestedScrollView](https://developer.android.com/reference/android/support/v4/widget/NestedScrollView)** 即 **支持嵌套滑动的 [ScrollView](https://developer.android.com/reference/android/widget/ScrollView.html)**。

因此，我们可以简单的把 **[NestedScrollView](https://developer.android.com/reference/android/support/v4/widget/NestedScrollView)** 类比为 [ScrollView](https://developer.android.com/reference/android/widget/ScrollView.html)，其作用就是作为控件父布局，从而具备（嵌套）滑动功能。

**[NestedScrollView](https://developer.android.com/reference/android/support/v4/widget/NestedScrollView)** 与 [ScrollView](https://developer.android.com/reference/android/widget/ScrollView.html) 的区别就在于 **[NestedScrollView](https://developer.android.com/reference/android/support/v4/widget/NestedScrollView)** 支持 *嵌套滑动*，无论是作为父控件还是子控件，嵌套滑动都支持，且默认开启。

因此，在一些需要支持嵌套滑动的情景中，比如一个 [ScrollView](https://developer.android.com/reference/android/widget/ScrollView.html) 内部包裹一个 `RecyclerView`，那么就会产生滑动冲突，这个问题就需要你自己去解决。而如果使用 **[NestedScrollView](https://developer.android.com/reference/android/support/v4/widget/NestedScrollView)** 包裹 `RecyclerView`，嵌套滑动天然支持，你无需做什么就可以实现前面想要实现的功能了。

举个例子：

我们通常为`RecyclerView`增加一个 Header 和 Footer 的方法是通过定义不同的 viewType来区分的，而如果使用 **[NestedScrollView](https://developer.android.com/reference/android/support/v4/widget/NestedScrollView)**，我们完全可以把`RecyclerView`当成一个单独的控件，然后在其上面增加一个控件作为 Header，在其下面增加一个控件作为 Footer。

具体布局如下所示：

```xml
<?xml version="1.0" encoding="utf-8"?>
<android.support.v4.widget.NestedScrollView xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:descendantFocusability="blocksDescendants"
        android:orientation="vertical">

        <!-- This is the Header -->
        <TextView
            android:layout_width="match_parent"
            android:layout_height="200dp"
            android:background="#888888"
            android:gravity="center"
            android:text="Header"
            android:textColor="#0000FF"
            android:textSize="30sp" />

        <android.support.v7.widget.RecyclerView
            android:id="@+id/rc"
            android:layout_width="match_parent"
            android:layout_height="wrap_content" />

        <!-- This is the Footer -->
        <TextView
            android:layout_width="match_parent"
            android:layout_height="200dp"
            android:background="#888888"
            android:gravity="center"
            android:text="Footer"
            android:textColor="#FF0000"
            android:textSize="30sp" />

    </LinearLayout>

</android.support.v4.widget.NestedScrollView>
```

**注：** **[NestedScrollView](https://developer.android.com/reference/android/support/v4/widget/NestedScrollView)** 与 [ScrollView](https://developer.android.com/reference/android/widget/ScrollView.html) 一样，内部只能容纳一个子控件。

效果如下所示：

<img src="https://upload-images.jianshu.io/upload_images/2222997-6a7cb811ae10a993.gif?imageMogr2/auto-orient/strip|imageView2/2/w/563/format/webp" alt="img" style="zoom:50%;" />

**ps：** 虽然 **[NestedScrollView](https://developer.android.com/reference/android/support/v4/widget/NestedScrollView)** 内嵌`RecyclerView`和其他控件可以实现 Header 和 Footer，但还是不推荐上面这种做法（建议还是直接使用`RecyclerView`自己添加 Header 和 Footer），因为虽然 **[NestedScrollView](https://developer.android.com/reference/android/support/v4/widget/NestedScrollView)** 支持嵌套滑动，但是在实际应用中，嵌套滑动可能会带来其他的一些奇奇怪怪的副作用，Google 也推荐我们能不使用嵌套滑动就尽量不要使用。

## 嵌套滑动机制 简析

我们知道，Android 的事件分发机制中，只要有一个控件消费了事件，其他控件就没办法再接收到这个事件了。因此，当有嵌套滑动场景时，我们都需要自己手动解决事件冲突。而在 Android 5.0 Lollipop 之后，Google 官方通过 **嵌套滑动机制** 解决了传统 Android 事件分发无法共享事件这个问题。

**嵌套滑动机制** 的基本原理可以认为是事件共享，即当子控件接收到滑动事件，准备要滑动时，会先通知父控件(`startNestedScroll`）；然后在滑动之前，会先询问父控件是否要滑动（`dispatchNestedPreScroll`)；如果父控件响应该事件进行了滑动，那么就会通知子控件它具体消耗了多少滑动距离；然后交由子控件处理剩余的滑动距离；最后子控件滑动结束后，如果滑动距离还有剩余，就会再问一下父控件是否需要在继续滑动剩下的距离（`dispatchNestedScroll`)...

上面其实就是 **嵌套滑动机制** 的工作原理，那么如果想让我们自定义的`View`或者`ViewGroup`实现嵌套滑动功能，应该怎样做呢？

其实，在 Android 5.0 之后，系统自带的`View`和`ViewGroup`都增加了 **嵌套滑动机制** 相关的方法了（但是默认不会被调用，因此默认不具备嵌套滑动功能），所以如果在 Android 5.0 及之后的平台上，自定义`View`只要覆写相应的 **嵌套滑动机制** 相关方法即可；但是为了提供低版本兼容性，Google 官方还提供了两个接口，分别作为 **嵌套滑动机制** 父控件接口和子控件接口：

- **[NestedScrollingParent](https://developer.android.com/reference/android/support/v4/view/NestedScrollingParent.html)**：作为父控件，支持嵌套滑动功能。
- **[NestedScrollingChild](https://developer.android.com/reference/android/support/v4/view/NestedScrollingChild.html)**：作为子控件，支持嵌套滑动功能。

前面我们说过 **[NestedScrollView](https://developer.android.com/reference/android/support/v4/widget/NestedScrollView)** 无论是作为父控件还是子控件都支持嵌套滑动，就是因为它同时实现了 **[NestedScrollingParent](https://developer.android.com/reference/android/support/v4/view/NestedScrollingParent.html)** 和 **[NestedScrollingChild](https://developer.android.com/reference/android/support/v4/view/NestedScrollingChild.html)**。文档如下所示：

<img src="https://gitee.com/ForeverHamburger/picgo_imgs1/raw/master/202410270000326.webp" alt="img" style="zoom:50%;" />

> 参考：
>
> [NestedScrollView、RecycleView、ViewPager 等布局方面的常见问题汇总，及解决 - 简书](https://www.jianshu.com/p/8dd1e902b7cd)
