# ViewPager2

`ViewPager2`是`androidx`下的组件，用于替代`ViewPager`。

## 新特性

* 内部实现基于`RecyclerView`。可以使用`DiffUtil`进行局部更新。
* 支持竖向滑动。
* 提供`CompositePageTransformer`，允许将多个`PageTransformer`叠加使用。
* 提供`setUserInputEnabled()`方法来开启或者关闭用户的输入。



## 使用

### 添加依赖

```
implementation 'androidx.viewpager2:viewpager2:1.1.0-alpha01'
```

### XML中使用

```xml
<androidx.viewpager2.widget.ViewPager2
        android:id="@+id/viewpager"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:clipChildren="false"
        android:layout_margin="50dp"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toTopOf="parent" />
```

### 差异

* Adapter的变化。

  如果`ViewPager`时使用的是`PagerAdapter`，现在需要改为`RecyclerView.Adapter`。

  如果`ViewPager`时使用的是`FragmentPagerAdapter`，现在需要改为`FragmentStateApdater`。

  如果`ViewPager`时使用的是`FragmentStatePagerAdapter`，现在需要改为`FragmentStateAdapter`。

* 滑动回调监听改为`ViewPager2.OnPageChangeCallback()`，通过`ViewPager2.registerOnPageChangeCallback`注册监听，通过`ViewPager2.unregisterOnPageChangeCallback`取消监听。

* 没有直接可以反射的`Scroller`控制滑动速度。需要用代码通过`ViewPager2.fakeDragBy()`来实现。
* 尚没有提供`PageTransformer`的`reverseDrawingOrder`功能。