# 【Android】View绘制流程

View绘制过程中有三个大名鼎鼎的方法：

> **Measure**：通过 `measure()` 方法计算出 View 的尺寸（宽度和高度）。
>
> **Layout**：通过 `layout()` 方法确定 View 的位置（左、上、右、下）。
>
> **Draw**：通过 `draw()` 方法绘制 View 的内容。

暂且先不提这三个方法，我们往回倒一倒，从DecorView被加载到Window开始。









## View的measure流程

