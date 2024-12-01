# 【Android】从事件分发开始：原理解析如何解决滑动冲突

[TOC]

> 参考：
>
> 《Android进阶之光》
>
> [【Android】图解View事件分发机制_android view 事件分发-CSDN博客](https://blog.csdn.net/weixin_73871834/article/details/136888568?spm=1001.2014.3001.5502)

## Activity层级结构

要知道Activity的层级结构，首先我们来看看Activity中总是出现的setContentView方法是干啥的。

### 浅析Activity的setContentView源码

点进Activity的setContentView。

```java
public void setContentView(@LayoutRes int layoutResID) {
    getWindow().setContentView(layoutResID);
    initWindowDecorActionBar();
}
```

这里调用了 getWindow().setContentView(layoutResID)。getWindow() 会得到什么呢？接着往下看，getWindow() 返回 mWindow：

```java
public Window getWindow() {
    return mWindow;
}
```

那么，这里的 mWindow 又是什么呢？我们继续查看代码，最终在 Activity 的 attach 方法中发现了 mWindow。

```java
final void attach(Context context, ActivityThread aThread,
    Instrumentation instr, IBinder token, int ident,
    Application application, Intent intent, ActivityInfo info,
    CharSequence title, Activity parent, String id,
    NonConfigurationInstances lastNonConfigurationInstances,
    Configuration config, String referrer, VoiceInteractor voiceInteractor) {
    attachBaseContext(context);
    fragments.attach(null /* opaque */);
    mWindow = new PhoneWindow(this);
    ...
}
```

由此可见，getWindow() 返回的是 PhoneWindow。下面来看看 PhoneWindow 的 setContentView 方法，代码如下所示：

```java
@Override
public void setContentView(View view, ViewGroup.LayoutParams params) {
    if (mContentParent == null) {
        installDecor();
    } else if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
        mContentParent.removeAllViews();
    }
    if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
        view.setLayoutParams(params);
        final Scene newScene = new Scene(mContentParent, view);
        transitionTo(newScene);
    } else {
        mContentParent.addView(view, params);
    }
}
```

```java
final Callback cb = getCallback();
if (cb != null && !isDestroyed()) {
    cb.onContentChanged();
}
```

原来 mWindow 指的就是 PhoneWindow，而 PhoneWindow 继承自抽象类 Window，这样就知道了getWindow() 得到的是一个 PhoneWindow，因为 Activity 中 setContentView 方法调用的是getWindow().setContentView(layoutResID)。

我们接着往下看，看看上面代码注释 1 处 installDecor 方法里面做了什么，代码如下所示：

```java
private void installDecor() {
    if (mDecor == null) {
        mDecor = generateLayout();
    }
}
```

在前面的代码中没发现什么，紧接着查看上面代码注释 1 处的 generateDecor 方法里做了什么：

```java
protected DecorView generateDecor() {
    return new DecorView(getContext(), -1);
}
```

这里创建了一个 DecorView。这个 DecorView 就是 Activity 中的根 View。接着查看 DecorView 的源码，发现 DecorView 是 PhoneWindow 类的内部类，并且继承了 FrameLayout。我们再回到 installDecor 方法中，查看 generateLayout(mDecor) 做了什么：

```java
protected ViewGroup generateLayout(DecorView decor) {
    //根据不同的情况，给 layoutResource 加载不同的布局
    int layoutResource;
    int features = getLocalFeatures();
    if ((features & (1 << FEATURE_SWIPE_TO_DISMISS)) != 0) {
        layoutResource = R.layout.screen_swipe_dismiss;
    } else if ((features & ((1 << FEATURE_LEFT_ICON) | (1 << FEATURE_RIGHT_ICON))) != 0) {
        if (isFloating()) {
            TypedValue res = new TypedValue();
            getContext().getTheme().resolveAttribute(
                R.attr.dialogTitleDecorLayout, res, true);
            layoutResource = res.resourceId;
        } else {
            layoutResource = R.layout.screen_title_icons;
        }
        removeFeatures(FEATURE_LEFT_ICON);
        removeFeatures(FEATURE_RIGHT_ICON);
    } else if ((features & (1 << FEATURE_ACTION_BAR)) != 0
            && (features & (1 << FEATURE_ACTION_BAR_OVERLAY)) == 0) {
        layoutResource = R.layout.screen_action_bar;
    } else if ((features & (1 << FEATURE_NO_TITLE)) == 0) {
        if (isFloating()) {
            TypedValue res = new TypedValue();
            getContext().getTheme().resolveAttribute(
                R.attr.dialogTitleDecorLayout, res, true);
            layoutResource = res.resourceId;
        } else {
            layoutResource = R.layout.screen_title;
        }
        removeFeatures(FEATURE_ACTION_BAR);
    } else if ((features & (1 << FEATURE_ACTION_BAR)) != 0) {
        layoutResource = R.styleable.Window_windowActionBarFullscreenDecorLayout,
            R.layout.screen_action_bar;
    } else if ((features & (1 << FEATURE_NO_TITLE)) == 0) {
        layoutResource = R.layout.screen_simple;
    } else {
        layoutResource = R.layout.screen_simple;
    }
    return contentParent;
}
```

PhoneWindow 的 generateLayout 方法比较长，这里只截取了一小部分关键的代码，其主要内容就是，根据不同的情况给 layoutResource 加载不同的布局。现在查看上面代码注释 1 处的布局 R.layout.screen_title，这个文件在 frameworks 源码中，它的代码如下所示：

```java
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:orientation="vertical"
    android:fitsSystemWindows="true">
    <!— Popup bar for action modes -->
    <ViewStub android:id="@+id/action_mode_bar_stub"
        android:inflatedLayout="@layout/action_mode_bar"
        android:layout_width="match_parent"
        android:layout_height="wrap_content" />
    <FrameLayout
        android:layout_width="match_parent"
        android:layout_height="0dp"
        android:layout_weight="1"
        style="?android:attr/windowTitleSizeStyle">
        <TextView android:id="@android:id/title"
            style="?android:attr/windowTitleStyle"
            android:background="?null"
            android:fadingEdge="horizontal" />
    </FrameLayout>
</LinearLayout>
```

上面的 ViewStub 是用来显示 ActionMode 的。下面的两个 FrameLayout：一个是 title，用来显示标题；另一个是 content，用来显示内容。看到上面的源码，大家就知道了一个 Activity 包含一个 Window 对象，该对象是由 PhoneWindow 实现的。PhoneWindow 将 DecorView 作为整个应用窗口的根 View，这个 DecorView 又将屏幕分为两个区域：一个区域是 TitleView，另一个区域是 ContentView，而我们平常做应用所写的布局正是显示在 ContentView 中的，如下图。

![image-20241201171333377](https://gitee.com/ForeverHamburger/picgo_imgs1/raw/master/202412011713531.png)



### 浅析AppCompatActivity的setContentView源码

我们在开发过程中用的基本都是AppCompatActivity，那么让我们看看AppCompatActivity的setContentView方法跟Activity的setContentView方法有什么不同。

首先来回忆以下Activity的setContentView方法中有两个很重要的东西：mDecor和mContentParent。前者是整个界面的根部局，后者是加载我们自定义布局的容器。其实AppCompatActivity的布局的层级跟Activity基本是一样的，只是在mContentParent中又多了一层布局而已。

话不多说，点开AppCompatActivity的setContentView方法：

```java
@Override
public void setContentView(@LayoutRes int layoutResID) {
    getDelegate().setContentView(layoutResID);
}
    
@NonNull
public AppCompatDelegate getDelegate() {
   	if (mDelegate == null) {
        mDelegate = AppCompatDelegate.create(this, this);
    }
    return mDelegate;
}

public static AppCompatDelegate create(Activity activity, AppCompatCallback callback) {
        return new AppCompatDelegateImpl(activity, activity.getWindow(), callback);
}
```

可以看到，里面是调用了一个委托对象的setContentView方法，这个对象就是AppCompatDelegateImpl，那么我们找到AppCompatDelegateImpl的setContentView方法：

```java
@Override
public void setContentView(int resId) {
    ensureSubDecor();
    ViewGroup contentParent = (ViewGroup) mSubDecor.findViewById(android.R.id.content);
    contentParent.removeAllViews();
    LayoutInflater.from(mContext).inflate(resId, contentParent);
    mOriginalWindowCallback.onContentChanged();
}
```

明白了Activity布局加载的过程我们可以猜到，方法中第二行获取到的contentParent的功能就相当于Activity中mContentParent，我们自定义的布局就是加载到这个contentParent中的，接下来的代码LayoutInflater.from(mContext).inflate(resId, contentParent)马上就证实了我们的判断。

那么第一行的ensureSubDecor()是干啥的呢？在开篇的时候说过，AppCompatActivity的布局的层级跟Activity基本是一样的，只是在mContentParent中又多了一层布局。这里提前给出结论：mContentParent中又多的那一层布局就是mSubDecor，而contentParent又是在mSubDecor下面的一个子布局。

点进ensureSubDecor方法：

```java
private void ensureSubDecor() {
    if (!mSubDecorInstalled) {
        mSubDecor = createSubDecor();
        ……
    }
```

可以看到通过createSubDecor方法创建mSubDecor。

```java
private ViewGroup createSubDecor() {
        TypedArray a = mContext.obtainStyledAttributes(R.styleable.AppCompatTheme);

        if (!a.hasValue(R.styleable.AppCompatTheme_windowActionBar)) {
            a.recycle();
            throw new IllegalStateException(
                    "You need to use a Theme.AppCompat theme (or descendant) with this activity.");
        }

        if (a.getBoolean(R.styleable.AppCompatTheme_windowNoTitle, false)) {
            requestWindowFeature(Window.FEATURE_NO_TITLE);
        } else if (a.getBoolean(R.styleable.AppCompatTheme_windowActionBar, false)) {
            // Don't allow an action bar if there is no title.
            requestWindowFeature(FEATURE_SUPPORT_ACTION_BAR);
        }
        if (a.getBoolean(R.styleable.AppCompatTheme_windowActionBarOverlay, false)) {
            requestWindowFeature(FEATURE_SUPPORT_ACTION_BAR_OVERLAY);
        }
        if (a.getBoolean(R.styleable.AppCompatTheme_windowActionModeOverlay, false)) {
            requestWindowFeature(FEATURE_ACTION_MODE_OVERLAY);
        }
        mIsFloating = a.getBoolean(R.styleable.AppCompatTheme_android_windowIsFloating, false);
        a.recycle();

        // Now let's make sure that the Window has installed its decor by retrieving it
        ensureWindow();
        
        //注释1
        mWindow.getDecorView();

        final LayoutInflater inflater = LayoutInflater.from(mContext);
        ViewGroup subDecor = null;
        
        //根据设置给subDecor加载不同的布局
        if (!mWindowNoTitle) {
            if (mIsFloating) {
                // If we're floating, inflate the dialog title decor
                subDecor = (ViewGroup) inflater.inflate(
                        R.layout.abc_dialog_title_material, null);

                // Floating windows can never have an action bar, reset the flags
                mHasActionBar = mOverlayActionBar = false;
            } else if (mHasActionBar) {
                /**
                 * This needs some explanation. As we can not use the android:theme attribute
                 * pre-L, we emulate it by manually creating a LayoutInflater using a
                 * ContextThemeWrapper pointing to actionBarTheme.
                 */
                TypedValue outValue = new TypedValue();
                mContext.getTheme().resolveAttribute(R.attr.actionBarTheme, outValue, true);

                Context themedContext;
                if (outValue.resourceId != 0) {
                    themedContext = new ContextThemeWrapper(mContext, outValue.resourceId);
                } else {
                    themedContext = mContext;
                }

                // Now inflate the view using the themed context and set it as the content view
                subDecor = (ViewGroup) LayoutInflater.from(themedContext)
                        .inflate(R.layout.abc_screen_toolbar, null);

                mDecorContentParent = (DecorContentParent) subDecor
                        .findViewById(R.id.decor_content_parent);
                mDecorContentParent.setWindowCallback(getWindowCallback());

                /**
                 * Propagate features to DecorContentParent
                 */
                if (mOverlayActionBar) {
                    mDecorContentParent.initFeature(FEATURE_SUPPORT_ACTION_BAR_OVERLAY);
                }
                if (mFeatureProgress) {
                    mDecorContentParent.initFeature(Window.FEATURE_PROGRESS);
                }
                if (mFeatureIndeterminateProgress) {
                    mDecorContentParent.initFeature(Window.FEATURE_INDETERMINATE_PROGRESS);
                }
            }
        } else {
            if (mOverlayActionMode) {
                subDecor = (ViewGroup) inflater.inflate(
                        R.layout.abc_screen_simple_overlay_action_mode, null);
            } else {
                subDecor = (ViewGroup) inflater.inflate(R.layout.abc_screen_simple, null);
            }

            if (Build.VERSION.SDK_INT >= 21) {
                // If we're running on L or above, we can rely on ViewCompat's
                // setOnApplyWindowInsetsListener
                ViewCompat.setOnApplyWindowInsetsListener(subDecor,
                        new OnApplyWindowInsetsListener() {
                            @Override
                            public WindowInsetsCompat onApplyWindowInsets(View v,
                                    WindowInsetsCompat insets) {
                                final int top = insets.getSystemWindowInsetTop();
                                final int newTop = updateStatusGuard(top);

                                if (top != newTop) {
                                    insets = insets.replaceSystemWindowInsets(
                                            insets.getSystemWindowInsetLeft(),
                                            newTop,
                                            insets.getSystemWindowInsetRight(),
                                            insets.getSystemWindowInsetBottom());
                                }

                                // Now apply the insets on our view
                                return ViewCompat.onApplyWindowInsets(v, insets);
                            }
                        });
            } else {
                // Else, we need to use our own FitWindowsViewGroup handling
                ((FitWindowsViewGroup) subDecor).setOnFitSystemWindowsListener(
                        new FitWindowsViewGroup.OnFitSystemWindowsListener() {
                            @Override
                            public void onFitSystemWindows(Rect insets) {
                                insets.top = updateStatusGuard(insets.top);
                            }
                        });
            }
        }

        if (subDecor == null) {
            throw new IllegalArgumentException(
                    "AppCompat does not support the current theme features: { "
                            + "windowActionBar: " + mHasActionBar
                            + ", windowActionBarOverlay: "+ mOverlayActionBar
                            + ", android:windowIsFloating: " + mIsFloating
                            + ", windowActionModeOverlay: " + mOverlayActionMode
                            + ", windowNoTitle: " + mWindowNoTitle
                            + " }");
        }

        if (mDecorContentParent == null) {
            mTitleView = (TextView) subDecor.findViewById(R.id.title);
        }

        // Make the decor optionally fit system windows, like the window's decor
        ViewUtils.makeOptionalFitsSystemWindows(subDecor);

        //注释2
        final ContentFrameLayout contentView = (ContentFrameLayout) subDecor.findViewById(
                R.id.action_bar_activity_content);
        final ViewGroup windowContentView = (ViewGroup) mWindow.findViewById(android.R.id.content);
        if (windowContentView != null) {
            // There might be Views already added to the Window's content view so we need to
            // migrate them to our content view
            while (windowContentView.getChildCount() > 0) {
                final View child = windowContentView.getChildAt(0);
                windowContentView.removeViewAt(0);
                contentView.addView(child);
            }

            // Change our content FrameLayout to use the android.R.id.content id.
            // Useful for fragments.
            //清除windowContentView的id
            windowContentView.setId(View.NO_ID);
            //将contentView的id设置成android.R.id.content
            contentView.setId(android.R.id.content);

            // The decorContent may have a foreground drawable set (windowContentOverlay).
            // Remove this as we handle it ourselves
            if (windowContentView instanceof FrameLayout) {
                ((FrameLayout) windowContentView).setForeground(null);
            }
        }

        // Now set the Window's content view with the decor
        //注释3
        mWindow.setContentView(subDecor);

        contentView.setAttachListener(new ContentFrameLayout.OnAttachListener() {
            @Override
            public void onAttachedFromWindow() {}

            @Override
            public void onDetachedFromWindow() {
                dismissPopups();
            }
        });

        return subDecor;
    }
```

在注释1处调用了mWindow的getDecorView方法，这里的mWindow就是PhoneWindow了，在PhoneWinow中找到getDecorView方法：

```java
    @Override
    public final View getDecorView() {
        if (mDecor == null || mForceDecorInstall) {
            installDecor();
        }
        return mDecor;
    }
```

可以看到里面调用了installDecor方法，在前面的文章里提过，做的事情就是初始化mDecor和mContentParent。接下来是根据设置给subDecor加载不同的布局。再接着，在注释2处，通过subDecor的findViewById(R.id.action_bar_activity_content)方法获取到了id为R.id.action_bar_activity_content的contentView，然后再通过mWindow的findViewById(android.R.id.content)方法获取到了windowContentView（对应着Activity中的mContentParent），接着通过windowContentView.setId(View.NO_ID)方法将windowContentView的id清除，之后再用contentView.setId(android.R.id.content)将contentView的id设为android.R.id.content。这样一来，contentView 就成为了Activity中的mContentParent，我们编写的的布局加载到contentView中。

最后在注释3处，调用了PhoneWindow的setContentView(subDecor)方法将创建的subDecor放到mContentParent中，但是此时的mContentParent已经不是加载我们自己编写布局的那个容器了，加载我们编写的布局的容器已经变成了subDecor中的contentView。

让我们再回到AppCompatDelegateImpl的setContentView方法：

```java
@Override
public void setContentView(int resId) {
    ensureSubDecor();
    ViewGroup contentParent = (ViewGroup) mSubDecor.findViewById(android.R.id.content);
    contentParent.removeAllViews();
    LayoutInflater.from(mContext).inflate(resId, contentParent);
    mOriginalWindowCallback.onContentChanged();
}
```

经过ensureSubDecor方法后，下面获取到的contentParent已经替换成了刚刚提到的contentView了，接下来通过LayoutInflater.from(mContext).inflate(resId, contentParent)将自己编写的布局加载到这个contentParent中来。

来看看整个流程：

![在这里插入图片描述](https://gitee.com/ForeverHamburger/picgo_imgs1/raw/master/202412011724980.png)

AppCompatActivity布局层级如下：

![在这里插入图片描述](https://gitee.com/ForeverHamburger/picgo_imgs1/raw/master/202412011725055.png)

最后总结一下Activity和AppCompatActivity的setContView方法区别：Activity的setContView直接将我们的布局渲染到mContentParent容器里面AppCompactActivity的setContView会根据不同的主题特性在mContentParent容器里面添加一个不同主题的subDecor容器，在subDecor容器里面有一个id为action_bar_activity_content的ContentFrameLayout容器（后来被替换成了R.id.content），并将我们的布局渲染到ContentFrameLayout里面(ContentFrameLayout也继承自FrameLayout)。

> 本部分参考：
>
> [Android开发Activity的setContentView源码分析_android activity setcontentview 源码-CSDN博客](https://blog.csdn.net/weixin_44965650/article/details/106749706)
>
> [AppCompatActivity#setContentView源码分析_appcompactactivity decorview-CSDN博客](https://blog.csdn.net/weixin_44965650/article/details/106882366)



## 触控三分显纷争，滑动冲突待分明

接下来进入本文的正文部分（那上面是什么？）

在之前写demo的时候，总能遇到这样的问题：

<img src="https://gitee.com/ForeverHamburger/picgo_imgs1/raw/master/202412011808311.png" alt="image-20241201180858241" style="zoom:33%;" />

如图所示，ViewPager2嵌套RecyclerView（由于ViewPager2底层用RecyclerView实现，故而说是双层RecyclerView嵌套也并无不可）。同时，这两层嵌套的View都是水平方向滑动的。

看似是比较常见的布局，但实际运行起来就出现了问题：

当我想滑动内层布局的时候，滑动了外层布局。我如果只想滑动内层布局，就只能用非常缓慢的速度滑动，且滑动非常滞涩。

这是什么原因呢？

上网搜索会发现这样的问题十分经典，叫**滑动冲突**，初遇时翻看许多相关文档，往往提及事件分发，触摸事件Event，拦截机制等，不解其意。

今天我们就从这里开始，简单说说产生滑动冲突的原因以及应该如何解决滑动冲突。

## 触控起源定归途

首先我们需要对事件分发机制有一定了解：

我们从手指点击屏幕的那一瞬间开始，当我们点击屏幕时，就产生了点击事件，这个事件被封装成了一个类，MotionEvent。因为当这个 MotionEvent 产生后，系统会将这个 MotionEvent 传递给 View 的层级，MotionEvent 在 View 中的传递过程就是点击事件分发。

我们从三个方法——`dispatchTouchEvent`、`onInterceptTouchEvent` 和 `onTouchEvent`开始介绍事件分发机制。

`dispatchTouchEvent(MotionEvent ev)` —— `dispatchTouchEvent` 是事件分发的入口，负责将触摸事件传递给视图的层级。所有的事件传递都会经过这个方法。

`onInterceptTouchEvent(MotionEvent ev)` —— 用来进行事件的拦截，在 `dispatchTouchEvent` 方法中调用，需要注意的是 View 没有提供该方法。

`onTouchEvent(MotionEvent ev)` —— 用来处理点击事件，在 `dispatchTouchEvent` 方法中进行调用。

放一张流程图在这里：这张图完美诠释了事件分发过程中的向上和向下传递。

![在这里插入图片描述](https://gitee.com/ForeverHamburger/picgo_imgs1/raw/master/202412012048514.png)



### 事件分发机制

**`Activity` 的 `dispatchTouchEvent` 方法**：

> 当用户点击屏幕时，系统首先会在 `Activity` 的 `dispatchTouchEvent` 方法中处理这个事件。
>
> `Activity` 会将事件交给其内部的 `PhoneWindow` 进行处理。具体来说，`PhoneWindow` 是 `Activity` 的一个成员，负责处理窗口相关的事件。

**`PhoneWindow` 的事件处理**：

> `PhoneWindow` 负责将事件传递到 `DecorView`。`DecorView` 是一个包含窗口布局的特殊 `ViewGroup`，它是 `Activity` 窗口的根视图，所有 UI 元素都在它的控制下。
>
> `PhoneWindow` 会调用 `DecorView` 的 `dispatchTouchEvent` 方法，将事件传递给 `DecorView`。

**`DecorView` 事件传递**：

> `DecorView` 会进一步将事件传递给其子视图。如果 `DecorView` 是根视图，通常它的子视图就是整个布局的根 `ViewGroup`。
>
> `DecorView` 通过 `dispatchTouchEvent` 将事件传递给这个根 `ViewGroup`，并开始处理事件分发。

我们从ViewGroup的dispatchTouchEvent方法开始分析：

代码如下所示：

```java
@Override
public boolean dispatchTouchEvent(MotionEvent ev) {
    ...
    if ((actionMasked == MotionEvent.ACTION_DOWN) ||
        cancelAndClearTouchTargets(ev)) {
        resetTouchState();
    }
    if (actionMasked == MotionEvent.ACTION_DOWN) {
        final boolean isTarget = mFirstTouchTarget != null;
        final boolean disallowIntercept = (mGroupFlags & FLAGDisallowIntercept) != 0;
        if (!isTarget || disallowIntercept) {
            intercepted = onInterceptTouchEvent(ev);
            if (intercepted) {
                ev.setAction(action);
            }
        } else {
            intercepted = false;
        }
    } else {
        ...
    }
    return handled;
}
```

首先，系统会判断是否需要重置触摸状态或清除目标视图。接着，通过检查是否有触摸目标（`mFirstTouchTarget`）以及是否允许拦截（`disallowIntercept` 标志位），决定是否调用 `onInterceptTouchEvent` 来判断是否拦截当前事件。如果事件被拦截，`ev.setAction(action)` 会确保事件继续保持正确的状态。如果不拦截，事件会继续传递给子视图，直到被处理或消费。最后，`handled` 标志返回事件是否已经处理。

> `mFirstTouchTarget` 是 `ViewGroup` 中的一个内部成员变量，表示当前触摸事件的第一个触摸目标视图。它的主要作用是在事件分发过程中，指示哪个视图正在处理触摸事件。
>
> **初始触摸目标**：在触摸事件开始（`ACTION_DOWN`）时，`mFirstTouchTarget` 会被设置为接收到该事件的第一个视图。这个视图会处理接下来的所有触摸事件，直到事件处理完成或被中断。

接下来看看 onInterceptTouchEvent() 方法：

```java
public boolean onInterceptTouchEvent(MotionEvent ev) {
    return false;
}
```

onInterceptTouchEvent() 方法默认为 false，不进行拦截。如果想要拦截事件，那么应该在自定义的 ViewGroup 中重写这个方法。

接着来看看 dispatchTouchEvent() 方法剩余的部分源码，如下所示：

```java
public boolean dispatchTouchEvent(MotionEvent event) {
    boolean result = false;
    if (onFilterTouchEventForSecurity(event)) {
        final int action = event.getAction();
        if (action == MotionEvent.ACTION_DOWN) {
            result = onTouchEvent(event);
        } else {
            result = onTouchEvent(event);
        }
    }
    return result;
}
```

如果 View 设置了点击事件监听器 OnClickListener，那么它的 onClick 方法就会被执行。

### 点击事件分发的传递规则

用伪代码来简单表示一下点击事件分发的这 3 个重要方法的关系：

```java
public boolean dispatchTouchEvent(MotionEvent event) {
    boolean result = false;
    if (onFilterTouchEventForSecurity(event)) {
        boolean intercepted = onInterceptTouchEvent(event);
        if (intercepted || (action == MotionEvent.ACTION_DOWN && mFirstTouchTarget != null)) {
            result = onTouchEvent(event);
        } else {
            for (int i = 0; i < childrenCount; i++) {
                final int childIndex = getChildIndex();
                View child = getChildAt(childIndex);
                if (canViewReceivePointerEvents(child) && isTransformedTouchPointInView(x, y, child, null)) {
                    ev.setTargetAccessibilityFocus(false);
                    newTouchTarget = getTouchTarget(child);
                    if (newTouchTarget != null) {
                        if (newTouchTarget.pointerIdBits != idBitsToAssign) {
                            break;
                        }
                        resetCancelNextUpFlag(child);
                        if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
                            mLastTouchDownIndex = childIndex;
                            mLastTouchDownX = ev.getX();
                            mLastTouchDownY = ev.getY();
                            newTouchTarget = addTouchTarget(child, idBitsToAssign);
                            alreadyDispatchedNewTouchTarget = true;
                            break;
                        }
                    } else {
                        mLastTouchDownIndex = childIndex;
                    }
                }
            }
        }
    }
    ev.setTargetAccessibilityFocus(false);
}
```

在这段代码中，我们可以看到触摸事件分发的过程，特别是如何判断哪个子视图处理触摸事件。

通过 `for` 循环遍历 `ViewGroup` 的子视图时，采用倒序遍历的方式。这意味着，系统首先从最上层的子视图（通常是最上面显示的视图）开始检查，依次检查每个子视图是否能够接收当前的触摸事件。

倒序遍历的原因通常是基于“Z轴顺序”原则，最上层的视图应该优先接收触摸事件。如果最上层的视图无法接收事件，则继续检查其他子视图，直到找到合适的视图来处理该事件。

在判断是否将事件传递给子视图时，代码会检查触摸点是否位于当前子视图的区域内（即是否在该视图的边界范围内）。如果触摸点位于子视图内部，说明该视图可以处理该事件。

另外，代码还会检查该子视图是否正在执行动画。如果子视图正在播放动画，通常会阻止事件的处理，因为动画可能正在改变视图的状态或位置，导致触摸事件的处理不一致或不准确。

如果这两个条件都不满足（即触摸点不在子视图范围内，且子视图不在播放动画），则使用 `continue` 语句跳过当前子视图，继续遍历下一个子视图。

接下来看 dispatchTransformedTouchEvent() 方法做了什么，代码如下所示：

```java
private boolean dispatchTransformedTouchEvent(MotionEvent event, boolean cancel, View child, int desiredPointerIndex) {
    final int oldAction = event.getAction();
    if (cancel || oldAction == MotionEvent.ACTION_CANCEL) {
        event.setAction(MotionEvent.ACTION_CANCEL);
        if (child == null) {
            handled = super.dispatchTouchEvent(event);
        } else {
            handled = child.dispatchTouchEvent(event);
        }
        event.setAction(oldAction);
        return handled;
    }
    ...
}
```

这个方法的作用是将事件分发到视图树中的特定视图，并根据需要调整事件的坐标或其他属性（例如，经过变换的坐标或视图的缩放）。

如果有子 View，则调用子 View 的 `dispatchTouchEvent()` 方法。如果是 ViewGroup 没有子 View，则调用 `super.dispatchTouchEvent(event)` 方法。ViewGroup 是继承自 View 的。下面再来看看 View 的 `dispatchTouchEvent` 方法：

```java
public boolean dispatchTouchEvent(MotionEvent event) {
    boolean result = false;
    if ((mViewFlags & ENABLED_MASK) == ENABLED
        && !(mViewFlags & CLICKABLE || mViewFlags & LONG_CLICKABLE)) {
        if (!mOnTouchListener.hasListeners()) {
            result = true;
        }
        if (result || onTouchEvent(event)) {
            result = true;
        }
    }
    ...
    return result;
}
```

我们看到如果 OnTouchListener 不为 null 并且 onTouch 方法返回 true，则表示事件被消费，就不会执行 `onTouchEvent(event)`，否则就会执行 `onTouchEvent(event)`，可以看出 OnTouchListener 中的 onTouch 方法优先级要高于 `onTouchEvent` 方法。

下面再来看看 `onTouchEvent` 方法的部分源码：

```java
public boolean onTouchEvent(MotionEvent event) {
    final int action = event.getAction();
    if (!(viewFlags & CLICKABLE || viewFlags & LONG_CLICKABLE || viewFlags & CONTEXT_CLICKABLE)) {
        return false;
    }
    switch (action) {
        case MotionEvent.ACTION_UP:
            boolean pressed = (mPrivateFlags & PFLAG_PRESSED) != 0;
            if ((mPrivateFlags & PFLAG_PRESSED) != 0 || pressed) {
                boolean focusTaken = false;
                if (!mHasPerformedLongPress && !ignoreNextUpEvent) {
                    if (mPerformClick != null) {
                        mPerformClick.onPerformClick();
                    }
                    if (post(mPerformClick)) {
                        performClick();
                    }
                }
            }
            break;
        ...
    }
    return true;
}
```

从上面的代码中可以看到，只要 View 的 CLICKABLE 和 LONG_CLICKABLE 有一个为 true，那么 `onTouchEvent()` 就会返回 true 消耗这个事件。CLICKABLE和 LONG_CLICKABLE 代表 View 可以被点击和长按点击，这可以通过 View 的 setClickable 和 setLongClickable 方法来设置，也可以通过 View 的 XML 文件来设置。接着在 ACTION_UP 事件中会调用 `performClick` 方法：

```java
public boolean performClick() {
    final boolean result;
    final ListenerInfo li = mListenerInfo;
    if (li != null && li.mOnClickListener != null) {
        playSoundEffect(SoundEffects.CLICK);
        li.mOnClickListener.onClick(this);
        result = true;
    } else {
        sendAccessibilityEvent(AccessibilityEvent.TYPE_VIEW_CLICKED);
        result = false;
    }
    return result;
}
```

如果 View 设置了点击事件监听器 OnClickListener，那么它的 onClick 方法就会被执行。

### 点击事件分发的传递规则

由前面事件分发机制的源码分析可知点击事件分发的这3个重要方法的关系，下面用伪代码来简单表示：

```java
public boolean dispatchTouchEvent(MotionEvent ev) {
    boolean result = false;
    if (onInterceptTouchEvent(ev)) {
        result = onTouchEvent(ev);
    } else {
        result = child.dispatchTouchEvent(ev);
    }
    return result;
}
```

`onInterceptTouchEvent` 方法和 `onTouchEvent` 方法都在 `dispatchTouchEvent` 方法中调用。现在我们根据这段伪代码来分析一下点击事件分发的传递规则。

首先讲一下点击事件由上而下的传递规则，当点击事件产生后会由 Activity 来处理，传递给 PhoneWindow，再传递给 DecorView，最后传递给顶层的 ViewGroup。一般在事件传递中只考虑 ViewGroup 的 `onInterceptTouchEvent` 方法，因为一般情况下我们不会重写 `dispatchTouchEvent` 方法。对于根 ViewGroup，点击事件首先传递给它的 `dispatchTouchEvent` 方法。如果该 ViewGroup 的 `onInterceptTouchEvent` 方法返回 true，则表示它要拦截这个事件，这个事件就会交给它的 `onTouchEvent` 方法处理；如果 `onInterceptTouchEvent` 方法返回 false，则表示它不拦截这个事件，这个事件就会交给它的子元素的 `dispatchTouchEvent` 来处理，如此反复下去。如果传递给底层的 View，该 View 是没有子 View 的，这时就会调用 View 的 `dispatchTouchEvent` 方法。一般情况下最终会调用 View 的 `onTouchEvent` 方法。

### 产生滑动冲突的原因

了解了滑动冲突的基本原理，我们现在大概清楚了，是由于在Move事件分发的时候，被外层的ViewPager2提前拦截了，而导致RecyclerView无法接收到Move事件，从而也无法处理滑动了。

但既然如此，那么按理来说，只要产生任何滑动都会被ViewPager2拦截，但为什么在缓慢滑动的时候，RecyclerView有时会接收到滑动事件从而进行滑动呢？

我们直接看看RecyclerView的拦截onInterceptTouchEvent方法，一探究竟。

```java
@Override
    public boolean onInterceptTouchEvent(MotionEvent e) {
        if (mLayoutSuppressed) {
            // When layout is suppressed,  RV does not intercept the motion event.
            // A child view e.g. a button may still get the click.
            return false;
        }

        // Clear the active onInterceptTouchListener.  None should be set at this time, and if one
        // is, it's because some other code didn't follow the standard contract.
        mInterceptingOnItemTouchListener = null;
        if (findInterceptingOnItemTouchListener(e)) {
            cancelScroll();
            return true;
        }

        if (mLayout == null) {
            return false;
        }

        final boolean canScrollHorizontally = mLayout.canScrollHorizontally();
        final boolean canScrollVertically = mLayout.canScrollVertically();

        if (mVelocityTracker == null) {
            mVelocityTracker = VelocityTracker.obtain();
        }
        mVelocityTracker.addMovement(e);

        final int action = e.getActionMasked();
        final int actionIndex = e.getActionIndex();

        switch (action) {
            case MotionEvent.ACTION_DOWN:
                if (mIgnoreMotionEventTillDown) {
                    mIgnoreMotionEventTillDown = false;
                }
                mScrollPointerId = e.getPointerId(0);
                mInitialTouchX = mLastTouchX = (int) (e.getX() + 0.5f);
                mInitialTouchY = mLastTouchY = (int) (e.getY() + 0.5f);

                if (stopGlowAnimations(e) || mScrollState == SCROLL_STATE_SETTLING) {
                    getParent().requestDisallowInterceptTouchEvent(true);
                    setScrollState(SCROLL_STATE_DRAGGING);
                    stopNestedScroll(TYPE_NON_TOUCH);
                }

                // Clear the nested offsets
                mNestedOffsets[0] = mNestedOffsets[1] = 0;

                int nestedScrollAxis = ViewCompat.SCROLL_AXIS_NONE;
                if (canScrollHorizontally) {
                    nestedScrollAxis |= ViewCompat.SCROLL_AXIS_HORIZONTAL;
                }
                if (canScrollVertically) {
                    nestedScrollAxis |= ViewCompat.SCROLL_AXIS_VERTICAL;
                }
                startNestedScroll(nestedScrollAxis, TYPE_TOUCH);
                break;

            case MotionEvent.ACTION_POINTER_DOWN:
                mScrollPointerId = e.getPointerId(actionIndex);
                mInitialTouchX = mLastTouchX = (int) (e.getX(actionIndex) + 0.5f);
                mInitialTouchY = mLastTouchY = (int) (e.getY(actionIndex) + 0.5f);
                break;

            case MotionEvent.ACTION_MOVE: {
                final int index = e.findPointerIndex(mScrollPointerId);
                if (index < 0) {
                    Log.e(TAG, "Error processing scroll; pointer index for id "
                            + mScrollPointerId + " not found. Did any MotionEvents get skipped?");
                    return false;
                }

                final int x = (int) (e.getX(index) + 0.5f);
                final int y = (int) (e.getY(index) + 0.5f);
                if (mScrollState != SCROLL_STATE_DRAGGING) {
                    final int dx = x - mInitialTouchX;
                    final int dy = y - mInitialTouchY;
                    boolean startScroll = false;
                    if (canScrollHorizontally && Math.abs(dx) > mTouchSlop) {
                        mLastTouchX = x;
                        startScroll = true;
                    }
                    if (canScrollVertically && Math.abs(dy) > mTouchSlop) {
                        mLastTouchY = y;
                        startScroll = true;
                    }
                    if (startScroll) {
                        setScrollState(SCROLL_STATE_DRAGGING);
                    }
                }
            }
            break;

            case MotionEvent.ACTION_POINTER_UP: {
                onPointerUp(e);
            }
            break;

            case MotionEvent.ACTION_UP: {
                mVelocityTracker.clear();
                stopNestedScroll(TYPE_TOUCH);
            }
            break;

            case MotionEvent.ACTION_CANCEL: {
                cancelScroll();
            }
        }
        return mScrollState == SCROLL_STATE_DRAGGING;
    }
```

直接截取核心部分：

```java
case MotionEvent.ACTION_MOVE: {
                final int x = (int) (e.getX(index) + 0.5f);
                final int y = (int) (e.getY(index) + 0.5f);
                if (mScrollState != SCROLL_STATE_DRAGGING) {
                    final int dx = x - mInitialTouchX;
                    final int dy = y - mInitialTouchY;
                    boolean startScroll = false;
                    if (canScrollHorizontally && Math.abs(dx) > mTouchSlop) {
                        mLastTouchX = x;
                        startScroll = true;
                    }
                    if (canScrollVertically && Math.abs(dy) > mTouchSlop) {
                        mLastTouchY = y;
                        startScroll = true;
                    }
                    if (startScroll) {
                        setScrollState(SCROLL_STATE_DRAGGING);
                    }
                }
            }
            break;
```

当触摸事件为 `ACTION_MOVE` 事件类型时，表示用户正在滑动手指。每次用户滑动手指时都会触发这个事件。

通过 `e.getX(index)` 和 `e.getY(index)` 获取触摸事件的当前位置。

这里检查当前的滑动状态是否是 **拖动中（SCROLL_STATE_DRAGGING）**。如果不是拖动状态，才会继续判断是否进入拖动状态。

计算当前触摸点与最初触摸点（`mInitialTouchX` 和 `mInitialTouchY`）的水平和垂直偏移量。`dx` 是水平方向上的偏移量，`dy` 是竖直方向上的偏移量。这个偏移量表示触摸点相对初始位置的滑动距离。

`startScroll` 用来标记是否开始滑动，初始设为 `false`。

如果支持水平滑动（`canScrollHorizontally` 为 `true`），并且水平方向的滑动距离 `dx` 超过了触摸滑动阈值 `mTouchSlop`，则认为开始了水平滑动。`mLastTouchX` 记录当前的 `x` 坐标，表示上次滑动位置，并将 `startScroll` 设置为 `true`，表示滑动开始。

同样，如果支持竖直滑动（`canScrollVertically` 为 `true`），并且竖直方向的滑动距离 `dy` 超过了 `mTouchSlop`，则认为开始了竖直滑动。`mLastTouchY` 记录当前的 `y` 坐标，表示上次滑动位置，并将 `startScroll` 设置为 `true`，表示滑动开始。

如果 `startScroll` 为 `true`，表示触摸滑动已经开始，此时调用 `setScrollState(SCROLL_STATE_DRAGGING)` 将滚动状态设置为 **拖动中（DRAGGING）**，标志着滑动已经开始，视图可以开始响应滚动。

> 综上所述，在判断是否拦截此滑动事件时，有一个属性mTouchSlop起到了关键作用。

那么是否是因为RecyclerView和ViewPager2的mTouchSlop的大小设置不同导致的呢？

我们再进入源码当中看一下，搜索mTouchSlop，可以找到这样一个方法。

<img src="https://gitee.com/ForeverHamburger/picgo_imgs1/raw/master/202412011916596.png" alt="image-20241201191612432" style="zoom:67%;" />

可以发现RecyclerView有两种TouchSlop，默认的TouchSlop是PagingTouchSlop的两倍。在RecyclerView中，TouchSlop默认是：

<img src="https://gitee.com/ForeverHamburger/picgo_imgs1/raw/master/202412011922036.png" alt="image-20241201192206938" style="zoom:67%;" />

而在ViewPager2中，我们找到了这样一句：

<img src="https://gitee.com/ForeverHamburger/picgo_imgs1/raw/master/202412011917064.png" alt="image-20241201191737962" style="zoom: 67%;" />

破案！滑动滞涩本质上是事件拦截，而如果滑动的比较慢，那么ViewPager2不会将此滑动事件拦截，而是继续传递给内层的RecyclerView，由于内层的TouchSlop比较小，所以将事件拦截下来。

## 拦截消冲理滑争

现在我们已经了解了事件分发的机制，并且了解了产生滑动冲突的原因，那我们应该如何解决这个问题呢？

网络上常见的解决方案有几种：

> 1. 禁用父 `ViewPager2` 的滑动：如果嵌套的 `ViewPager2` 实例不需要滚动，可以禁用父 `ViewPager2` 实例的滑动。这可以通过调用 `ViewPager2.setUserInputEnabled(false)` 方法来实现。（那为什么用VP2..？）
>
> 2. 子view拿到事件后，通过调用requestDisallowInterceptTouchEvent(true)来禁用父view拦截事件（父View不会再调用自己的onInterceptTouchEvent方法了）

学习解决方案之前，还有一件事需要我们了解：DOWN事件不会被拦截。

根据刚刚的学习，我们只了解了什么时候拦截器会拦截MOVE事件，但如果拦截器会将任何正常情况下的MOVE事件拦截，我们如何确保事件能传入下一层View中呢？

我们再回到拦截事件的源码当中：

<img src="https://gitee.com/ForeverHamburger/picgo_imgs1/raw/master/202412012019443.png" alt="image-20241201201943373" style="zoom:67%;" />

在这个方法中，判断是否拦截事件的依据是，当前的滑动状态是否为SCROLL_STATE_DRAGGING，那这个状态在哪里会改变呢？

```java
switch (action) {
            case MotionEvent.ACTION_DOWN:
                if (mIgnoreMotionEventTillDown) {
                    mIgnoreMotionEventTillDown = false;
                }
                mScrollPointerId = e.getPointerId(0);
                mInitialTouchX = mLastTouchX = (int) (e.getX() + 0.5f);
                mInitialTouchY = mLastTouchY = (int) (e.getY() + 0.5f);

                if (stopGlowAnimations(e) || mScrollState == SCROLL_STATE_SETTLING) {
                    getParent().requestDisallowInterceptTouchEvent(true);
                    setScrollState(SCROLL_STATE_DRAGGING);
                    stopNestedScroll(TYPE_NON_TOUCH);
                }

                // Clear the nested offsets
                mNestedOffsets[0] = mNestedOffsets[1] = 0;

                int nestedScrollAxis = ViewCompat.SCROLL_AXIS_NONE;
                if (canScrollHorizontally) {
                    nestedScrollAxis |= ViewCompat.SCROLL_AXIS_HORIZONTAL;
                }
                if (canScrollVertically) {
                    nestedScrollAxis |= ViewCompat.SCROLL_AXIS_VERTICAL;
                }
                startNestedScroll(nestedScrollAxis, TYPE_TOUCH);
                break;

            case MotionEvent.ACTION_POINTER_DOWN:
                mScrollPointerId = e.getPointerId(actionIndex);
                mInitialTouchX = mLastTouchX = (int) (e.getX(actionIndex) + 0.5f);
                mInitialTouchY = mLastTouchY = (int) (e.getY(actionIndex) + 0.5f);
                break;

            case MotionEvent.ACTION_MOVE: {
                final int index = e.findPointerIndex(mScrollPointerId);
                if (index < 0) {
                    Log.e(TAG, "Error processing scroll; pointer index for id "
                            + mScrollPointerId + " not found. Did any MotionEvents get skipped?");
                    return false;
                }

                final int x = (int) (e.getX(index) + 0.5f);
                final int y = (int) (e.getY(index) + 0.5f);
                if (mScrollState != SCROLL_STATE_DRAGGING) {
                    final int dx = x - mInitialTouchX;
                    final int dy = y - mInitialTouchY;
                    boolean startScroll = false;
                    if (canScrollHorizontally && Math.abs(dx) > mTouchSlop) {
                        mLastTouchX = x;
                        startScroll = true;
                    }
                    if (canScrollVertically && Math.abs(dy) > mTouchSlop) {
                        mLastTouchY = y;
                        startScroll = true;
                    }
                    if (startScroll) {
                        setScrollState(SCROLL_STATE_DRAGGING);
                    }
                }
            }
            break;

            case MotionEvent.ACTION_POINTER_UP: {
                onPointerUp(e);
            }
            break;

            case MotionEvent.ACTION_UP: {
                mVelocityTracker.clear();
                stopNestedScroll(TYPE_TOUCH);
            }
            break;

            case MotionEvent.ACTION_CANCEL: {
                cancelScroll();
            }
        }
```

我们发现在MotionEvent.ACTION_DOWN这一case中，滑动状态仅有一种可能会改变，即SCROLL_STATE_SETTLING时。而一般情况的点击是不会对这一属性进行更改的，也就是说RecyclerView并没有拦截down事件。

我们来梳理一下大致的流程：

> ##### **`ACTION_DOWN` 事件的处理**
>
> **触发 `ACTION_DOWN` 事件**：触摸事件从用户的手指接触屏幕开始，首先触发 `ACTION_DOWN` 事件。此事件首先进入 `RecyclerView` 的 `onInterceptTouchEvent` 方法。
>
> **`RecyclerView` 没有拦截事件**：此时 `RecyclerView` 的 `onInterceptTouchEvent` 返回 `false`，并将事件交给其子项（通常是 `ItemView`）进行处理。
>
> **`ItemView` 消费了 `ACTION_DOWN` 事件**：如果 `ItemView` 可以点击，并且它的 `onTouchEvent` 方法默认返回 `true`，那么它消费了该事件。此时，`RecyclerView` 依然没有拦截任何事件，`mFirstTouchTarget` 被设置为 `ItemView`，标记着 `ItemView` 现在是当前触摸的目标。
>
> #####  **`ACTION_MOVE` 事件的处理**
>
> **`ACTION_MOVE` 到达**：用户开始滑动时，`ACTION_MOVE` 事件会继续传递给 `RecyclerView`。此时，`mFirstTouchTarget` 已经指向了 `ItemView`，所以事件会先进入 `RecyclerView` 的 `onInterceptTouchEvent` 方法。
>
> **滑动距离超过最小阈值**：如果 `ACTION_MOVE` 事件的滑动距离超过了设定的最小滑动阈值（`mTouchSlop`），`RecyclerView` 会判断此次事件是滑动事件而非点击事件。
>
> **`RecyclerView` 返回 `true`**：由于滑动事件的发生，`RecyclerView` 会开始拦截事件，返回 `true`，表示它已消费了该事件，阻止进一步的分发。这时，`intercepted` 被设置为 `true`，表示触摸事件已经被拦截。
>
> #####  **`ACTION_CANCEL` 事件的触发**
>
> **`cancelChild` 被设置为 `true`**：由于滑动距离超过了阈值并且 `RecyclerView` 拦截了事件，`cancelChild` 被设置为 `true`，表示当前的子项（即 `ItemView`）需要取消事件的处理。
>
> **触发 `ACTION_CANCEL`**：当 `cancelChild` 为 `true` 时，`RecyclerView` 会调用 `dispatchTransformedTouchEvent` 方法，将一个 `ACTION_CANCEL` 事件发送给 `ItemView`，通知它取消当前的触摸事件处理。这意味着 `ItemView` 会清除所有当前的触摸状态，并停止响应当前的触摸事件。
>
> #####  **事件传递过程的重置**
>
> **`mFirstTouchTarget` 被重置为 `null`**：在事件流程中的某个时刻，`mFirstTouchTarget` 被清空，意味着 `RecyclerView` 不再跟踪 `ItemView` 作为当前的触摸目标。
>
> **后续事件被 `RecyclerView` 消费**：由于 `mFirstTouchTarget` 为空，后续的触摸事件将直接交由 `RecyclerView` 处理，`RecyclerView` 会重新开始处理触摸事件，直到触摸事件处理完毕。

由于RecyclerView并没有拦截down事件，我们可以在接受到down事件的时候请求父View不要拦截事件

后续move事件到我们这并且水平滑动距离大于最小滑动距离的时候，再问我们的子RecyclerVIew能否在这个滑动方向滑动，如果能，继续禁止父类拦截事件。如果不能则允许父类拦截事件（此时可以根据自己的想法来设置）。

### EasyTry嵌套，强制拦截

```java
public class easytryView extends LinearLayout {
    public easytryView(Context context) {
        super(context);
    }

    public easytryView(Context context, @Nullable AttributeSet attrs) {
        super(context, attrs);
    }

    @Override
    public boolean onInterceptTouchEvent(MotionEvent event) {
        getParent().requestDisallowInterceptTouchEvent(true);
        return false;
    }

    @Override
    public boolean onTouchEvent(MotionEvent event) {
        getParent().requestDisallowInterceptTouchEvent(true);
        return true;
    }
}
```

**防止父视图拦截事件**：

`requestDisallowInterceptTouchEvent(true)` 在 `onInterceptTouchEvent` 和 `onTouchEvent` 中都调用，告诉父视图不要拦截触摸事件，确保当前的 `easytryView` 能够完全控制事件的处理。

**确保当前视图处理触摸事件**：

`onInterceptTouchEvent` 返回 `false` 表示 `easytryView` 不打算拦截事件，而是将事件交给 `onTouchEvent` 处理。

`onTouchEvent` 返回 `true` 表示事件已被消费，不再交给其他视图处理。

### NestedScrollableHost嵌套，选择性拦截

```java
import android.content.Context;
import android.util.AttributeSet;
import android.view.MotionEvent;
import android.view.View;
import android.view.ViewConfiguration;
import android.widget.FrameLayout;

import androidx.viewpager2.widget.ViewPager2;

public class NestedScrollableHost extends FrameLayout {
    private int touchSlop;
    private float initialX = 0f;
    private float initialY = 0f;

    // 循环遍历找到ViewPager2
    private ViewPager2 parentViewPager;

    public NestedScrollableHost(Context context) {
        super(context);
        init(context);
    }

    public NestedScrollableHost(Context context, AttributeSet attrs) {
        super(context, attrs);
        init(context);
    }

    private void init(Context context) {
        touchSlop = ViewConfiguration.get(context).getScaledTouchSlop();
    }

    // 循环遍历找到ViewPager2
    private ViewPager2 getParentViewPager() {
        View v = (View) getParent();
        while (v != null && !(v instanceof ViewPager2)) {
            v = (View) v.getParent();
        }
        return (ViewPager2) v;
    }

    // 找到子RecyclerView
    private View getChildView() {
        return getChildCount() > 0 ? getChildAt(0) : null;
    }

    private boolean canChildScroll(int orientation, float delta) {
        int direction = (int) (-Math.signum(delta));
        if (orientation == 0) {
            return getChildView() != null && getChildView().canScrollHorizontally(direction);
        } else if (orientation == 1) {
            return getChildView() != null && getChildView().canScrollVertically(direction);
        } else {
            throw new IllegalArgumentException("Invalid orientation");
        }
    }

    @Override
    public boolean onInterceptTouchEvent(MotionEvent e) {
        handleInterceptTouchEvent(e);
        return super.onInterceptTouchEvent(e);
    }

    private void handleInterceptTouchEvent(MotionEvent e) {
        ViewPager2 viewPager2 = getParentViewPager();
        if (viewPager2 == null) return;

        int orientation = viewPager2.getOrientation();

        // 如果子RecyclerView在viewPager2的滑动方向上不能滑动直接返回
        if (!canChildScroll(orientation, -1f) && !canChildScroll(orientation, 1f)) {
            return;
        }

        if (e.getAction() == MotionEvent.ACTION_DOWN) {
            initialX = e.getX();
            initialY = e.getY();
            // down事件直接强制禁止父view拦截事件
            // 后续事件先交给子RecyclerView判断是否能够消费
            // 如果这一块不强制禁止父view会导致后续事件可能直接没到子RecyclerView就被父view拦截了
            getParent().requestDisallowInterceptTouchEvent(true);
        } else if (e.getAction() == MotionEvent.ACTION_MOVE) {
            // 计算手指滑动距离
            float dx = e.getX() - initialX;
            float dy = e.getY() - initialY;
            boolean isVpHorizontal = orientation == ViewPager2.ORIENTATION_HORIZONTAL;

            float scaledDx = Math.abs(dx) * (isVpHorizontal ? 0.5f : 1f);
            float scaledDy = Math.abs(dy) * (isVpHorizontal ? 1f : 0.5f);

            // 滑动距离超过最小滑动值
            if (scaledDx > touchSlop || scaledDy > touchSlop) {
                if (isVpHorizontal == (scaledDy > scaledDx)) {
                    // 如果viewPager2是横向滑动但手势是竖直方向滑动，则允许所有父类拦截
                    getParent().requestDisallowInterceptTouchEvent(false);
                } else {
                    // 手势滑动方向和viewPager2是同方向的，需要询问子RecyclerView是否在同方向能滑动
                    if (canChildScroll(orientation, isVpHorizontal ? dx : dy)) {
                        // 子RecyclerView能滑动直接禁止父view拦截事件
                        getParent().requestDisallowInterceptTouchEvent(true);
                    } else {
                        // 子RecyclerView不能滑动（划到第一个Item还往右滑或者划到最后一个Item还往左划的时候）允许父view拦截
                        getParent().requestDisallowInterceptTouchEvent(false);
                    }
                }
            }
        }
    }
}
```

> 本段内容参考：
>
> [【View系列】手把手教你解决ViewPager2滑动冲突 - 知乎](https://zhuanlan.zhihu.com/p/373087151)

稍微解释一下这段代码，判断是否存在的安全性代码，暂且略过，我们且看核心部分的事件处理逻辑。

```java
private void handleInterceptTouchEvent(MotionEvent e) {
        ViewPager2 viewPager2 = getParentViewPager();
        if (viewPager2 == null) return;

        int orientation = viewPager2.getOrientation();

        // 如果子RecyclerView在viewPager2的滑动方向上不能滑动直接返回
        if (!canChildScroll(orientation, -1f) && !canChildScroll(orientation, 1f)) {
            return;
        }

        if (e.getAction() == MotionEvent.ACTION_DOWN) {
            initialX = e.getX();
            initialY = e.getY();
            // down事件直接强制禁止父view拦截事件
            // 后续事件先交给子RecyclerView判断是否能够消费
            // 如果这一块不强制禁止父view会导致后续事件可能直接没到子RecyclerView就被父view拦截了
            getParent().requestDisallowInterceptTouchEvent(true);
        } else if (e.getAction() == MotionEvent.ACTION_MOVE) {
            // 计算手指滑动距离
            float dx = e.getX() - initialX;
            float dy = e.getY() - initialY;
            boolean isVpHorizontal = orientation == ViewPager2.ORIENTATION_HORIZONTAL;

            float scaledDx = Math.abs(dx) * (isVpHorizontal ? 0.5f : 1f);
            float scaledDy = Math.abs(dy) * (isVpHorizontal ? 1f : 0.5f);

            // 滑动距离超过最小滑动值
            if (scaledDx > touchSlop || scaledDy > touchSlop) {
                if (isVpHorizontal == (scaledDy > scaledDx)) {
                    // 如果viewPager2是横向滑动但手势是竖直方向滑动，则允许所有父类拦截
                    getParent().requestDisallowInterceptTouchEvent(false);
                } else {
                    // 手势滑动方向和viewPager2是同方向的，需要询问子RecyclerView是否在同方向能滑动
                    if (canChildScroll(orientation, isVpHorizontal ? dx : dy)) {
                        // 子RecyclerView能滑动直接禁止父view拦截事件
                        getParent().requestDisallowInterceptTouchEvent(true);
                    } else {
                        // 子RecyclerView不能滑动（划到第一个Item还往右滑或者划到最后一个Item还往左划的时候）允许父view拦截
                        getParent().requestDisallowInterceptTouchEvent(false);
                    }
                }
            }
        }
    }
```

**`ACTION_DOWN`**：记录触摸初始位置，禁止父视图拦截事件（`requestDisallowInterceptTouchEvent(true)`），确保后续事件能够到达子视图。

**`ACTION_MOVE`**：

计算当前手指滑动的距离，并根据滑动方向判断是否允许父视图拦截。

如果手指滑动的方向与 `ViewPager2` 的滚动方向不一致（例如，`ViewPager2` 是横向滑动，用户手势是竖直滑动），则允许父视图（`ViewPager2`）拦截事件。

如果手指滑动的方向与 `ViewPager2` 一致，则检查子 `RecyclerView` 是否能够滚动。如果可以滚动，禁止父视图拦截事件，反之，允许父视图拦截。

## 结语

本篇博客的代码部分仍然存在一些问题，关于Activtiy层级的部分也并没有多做阐释，学习不够深入，日后考虑写几篇博客填充一下这部分内容。