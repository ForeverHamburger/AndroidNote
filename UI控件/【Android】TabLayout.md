# 【Android】TabLayout

`TabLayout` 是 Android 原生开发工具包中的一个强大的 UI 组件，用于创建可定制的选项卡式界面。它允许用户在不同选项卡之间切换，每个选项卡包含一组相关的内容。以下是 `TabLayout` 的一些关键特性和用法：

### 1. 基本构成及使用

- `TabLayout`：一个横向可滑动的菜单导航 UI 组件。
- `Tab`：`TabLayout` 中的项，可以通过 `newTab()` 创建。
- `TabView`：`Tab` 的实例，是一个包含 `ImageView` 和 `TextView` 的线性布局。
- `TabItem`：一种特殊的“视图”，在 `TabLayout` 中可以显式声明 `Tab`。

### 2. 功能拆解

**基础实现**：可以在 XML 中定义 `TabLayout`，也可以通过 Java/Kotlin 代码动态添加 `Tab`。

**XML 动态写法**：在 XML 布局文件中声明 `TabLayout`，并在代码中动态添加 `Tab`。

```xml
<com.google.android.material.tabs.TabLayout
    android:id="@+id/tab_layout1"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    app:tabIndicatorColor="@color/colorPrimary"
    app:tabMaxWidth="200dp"
    app:tabMinWidth="100dp"
    app:tabMode="fixed"
    app:tabSelectedTextColor="@color/colorPrimary"
    app:tabTextColor="@color/gray" />
```

**XML 静态写法**：在 XML 中直接声明 `TabItem`。

```xml
<com.google.android.material.tabs.TabLayout
    android:layout_height="wrap_content"
    android:layout_width="match_parent">
    <com.google.android.material.tabs.TabItem
        android:text="@string/tab_text"/>
    <com.google.android.material.tabs.TabItem
        android:icon="@drawable/ic_android"/>
</com.google.android.material.tabs.TabLayout>
```

**字体大小、加粗**：通过 `app:tabTextAppearance` 给 `TabLayout` 设置文本样式。

```xml
<com.google.android.material.tabs.TabLayout
    ...
    app:tabTextAppearance="@style/MyTabLayout" />
```

### 3. 与 `ViewPager` 关联

`TabLayout` 通常与 `ViewPager` 结合使用，实现滑动的标签选择器。通过 `setupWithViewPager()` 方法将 `TabLayout` 与 `ViewPager` 关联。

```java
TabLayout tabLayout = findViewById(R.id.tab_layout);
ViewPager viewPager = findViewById(R.id.view_pager);
viewPager.setAdapter(new MyPagerAdapter());
tabLayout.setupWithViewPager(viewPager);
```

### 4. 常用 API 整理

- **TabLayout**：设置背景颜色、指示器颜色、指示器高度等。
- **TabLayout.Tab**：设置自定义视图、图标、文本等。
- **BadgeDrawable**：设置小红点的显示状态、背景颜色、文本颜色等。

### 5. 自定义标签页样式

可以通过自定义布局来实现更复杂的标签页样式，例如添加图标和标题。

java

```java
private View makeTabView(int position) {
    View tabView = LayoutInflater.from(this).inflate(R.layout.tab_text_icon, null);
    TextView textView = tabView.findViewById(R.id.textview);
    ImageView imageView = tabView.findViewById(R.id.imageview);
    textView.setText(titles[position]);
    imageView.setImageResource(pics[position]);
    return tabView;
}
```

### 6. 监听标签页变化

可以使用 `addOnTabSelectedListener()` 方法来监听标签页的变化。

```java
tabLayout.addOnTabSelectedListener(new TabLayout.OnTabSelectedListener() {
    @Override
    public void onTabSelected(TabLayout.Tab tab) {
        // 标签被选中时的逻辑
    }

    @Override
    public void onTabUnselected(TabLayout.Tab tab) {
        // 标签未被选中时的逻辑
    }

    @Override
    public void onTabReselected(TabLayout.Tab tab) {
        // 标签被重新选中时的逻辑
    }
});
```

![image-20241023212215649](https://gitee.com/ForeverHamburger/picgo_imgs1/raw/master/202410232122994.png)
