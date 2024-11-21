# BottomSheet

### 1.  **BottomSheet**

`BottomSheet` 是与主界面同层级的视图，可以通过触发事件来显示，且不会影响主界面的交互。要使用 `BottomSheet`，需要在 XML 中使用 `CoordinatorLayout` 作为父布局，并添加相应的 `BottomSheetBehavior`。

#### XML 示例：

```xml
<android.support.design.widget.CoordinatorLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:background="@color/white">

    <LinearLayout
        android:id="@+id/ll_bottom_sheet"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="vertical"
        app:behavior_peekHeight="80dp"
        app:layout_behavior="@string/bottom_sheet_behavior">

        <!-- 子视图内容 -->
    </LinearLayout>
</android.support.design.widget.CoordinatorLayout>
```

在代码中，使用 `BottomSheetBehavior` 来控制 `BottomSheet` 的状态（如展开或折叠）：

```java
View bottomView = findViewById(R.id.ll_bottom_sheet);
bottomView.setOnClickListener(v -> {
    BottomSheetBehavior behavior = BottomSheetBehavior.from(bottomView);
    if (behavior.getState() == BottomSheetBehavior.STATE_EXPANDED) {
        behavior.setState(BottomSheetBehavior.STATE_COLLAPSED);
    } else {
        behavior.setState(BottomSheetBehavior.STATE_EXPANDED);
    }
});
```

### 2. **BottomSheetDialog**

`BottomSheetDialog` 是一个弹出的对话框样式底部弹窗，它会影响主界面的交互，可以在弹窗外点击时关闭弹窗。

#### 使用代码：

```java
public class MyBottomSheetDialog extends BottomSheetDialog {
    public MyBottomSheetDialog(@NonNull Context context) {
        super(context);
    }
}
```

展示方法：

```java
java复制代码MyBottomSheetDialog bottomSheetDialog = new MyBottomSheetDialog(this);
bottomSheetDialog.setContentView(R.layout.bottom_dialog_layout);
bottomSheetDialog.show();
```

### 3. **BottomSheetDialogFragment**

`BottomSheetDialogFragment` 是 `BottomSheetDialog` 的一个扩展，基于 `DialogFragment`，提供了更灵活的弹窗管理。

#### 示例代码：

```java
public class MyDialogFragment extends BottomSheetDialogFragment {
    @Nullable
    @Override
    public View onCreateView(@NonNull LayoutInflater inflater, @Nullable ViewGroup container, @Nullable Bundle savedInstanceState) {
        return inflater.inflate(R.layout.bottom_dialog_fragment_layout, container, false);
    }
}
```

展示方法：

```java
MyDialogFragment dialog = new MyDialogFragment();
dialog.show(getSupportFragmentManager(), "dialog_fragment");
```

### 4. **自定义效果**

**圆角效果**： 要给 `BottomSheet` 设置圆角，可以在 `style.xml` 中配置透明背景和圆角效果。

#### `style.xml` 示例：

```xml
<style name="MyBottomSheetDialog" parent="Theme.Design.Light.BottomSheetDialog">
    <item name="bottomSheetStyle">@style/BottomSheetStyleWrapper</item>
    <item name="android:background">@android:color/transparent</item>
    <item name="android:windowBackground">@android:color/transparent</item>
</style>

<style name="BottomSheetStyleWrapper" parent="Widget.MaterialComponents.BottomSheet.Modal">
    <item name="android:background">@android:color/transparent</item>
</style>
```

**去掉背景蒙版**：

```java
<item name="android:backgroundDimEnabled">false</item>
```

**设置蒙版透明度**：

```java
<item name="android:backgroundDimAmount">0.4</item>
```

**点击外部区域，dialog 不消失**：

```java
@Override
public Dialog onCreateDialog(Bundle savedInstanceState) {
    Dialog dialog = super.onCreateDialog(savedInstanceState);
    dialog.setCanceledOnTouchOutside(false);  // 不允许点击外部消失
    return dialog;
}
```

### 5. **高级配置**

- **禁止向下拖动**：

```java
bottomSheetBehavior.setHideable(false);
```

- **设置弹框固定高度**：

```java
bottomSheetBehavior.setPeekHeight(1200);
```

- **内容铺满全屏**：

```java
bottomSheetBehavior.setState(BottomSheetBehavior.STATE_EXPANDED);
```

- **监听展开收起状态**：

```java
bottomSheetBehavior.setBottomSheetCallback(new BottomSheetBehavior.BottomSheetCallback() {
    @Override
    public void onStateChanged(@NonNull View view, int newState) {
        switch (newState) {
            case BottomSheetBehavior.STATE_EXPANDED:
                // 展开状态
                break;
            case BottomSheetBehavior.STATE_COLLAPSED:
                // 折叠状态
                break;
            // 其他状态
        }
    }
});
```



