# 【Android】从事件分发开始：原理解析如何解决滑动冲突

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

以上就是我对AppCompatActivity的setContentView加载布局的理解，欢迎大家在评论区留言指正。







> 本部分参考：
>
> [Android开发Activity的setContentView源码分析_android activity setcontentview 源码-CSDN博客](https://blog.csdn.net/weixin_44965650/article/details/106749706)
>
> [AppCompatActivity#setContentView源码分析_appcompactactivity decorview-CSDN博客](https://blog.csdn.net/weixin_44965650/article/details/106882366)



## 触控三分显纷争，滑动冲突待分明









## 触控起源定归途，分发机制理千层











## 分发有序定乾坤，拦截消冲理滑争