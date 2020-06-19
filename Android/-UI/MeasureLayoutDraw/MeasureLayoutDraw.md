# MeasureLayoutDraw

## 入口

在`Activity`启动并执行`ActivityThread.handleResumeActivity()`中，会将`DecorView`设置给`WindowManagerImpl`，最终调用到`ViewRootImpl.setView()`，`ViewRootImpl.requestLayout()`，`ViewRootImpl.scheduleTraversals()`。

`scheduleTraversals()`方法中会向`Chereographer`提交一个`TraversalRunnable`，在屏幕渲染的下一帧进行视图遍历。

```java
void scheduleTraversals() {
    if (!mTraversalScheduled) {
        mTraversalScheduled = true;
        mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
        /* ------ 向Choreographer提交TraversalRunnable ------ */
        mChoreographer.postCallback(
            Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
        if (!mUnbufferedInputDispatch) {
            scheduleConsumeBatchedInput();
        }
        notifyRendererOfFramePending();
        pokeDrawLockIfNeeded();
    }
}

final class TraversalRunnable implements Runnable {
    @Override
    public void run() {
        doTraversal();
    }
}

void doTraversal() {
    if (mTraversalScheduled) {
        mTraversalScheduled = false;
        mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);

        if (mProfile) {
            Debug.startMethodTracing("ViewAncestor");
        }

        /* ----- 最终执行视图遍历 ----- */
        performTraversals();

        if (mProfile) {
            Debug.stopMethodTracing();
            mProfile = false;
        }
    }
}
```

`ViewTreeImpl.performTraversals()`，先后执行下面的方法：

```java
ViewTreeImpl.performMeasure()
ViewTreeImpl.performLayout()
ViewTreeImpl.performDraw()
```
进而一步一步调用到：
```java
View.measure()
View.layout()
View.draw()
```

`ViewRootImpl`中存储的View就是在`handleResumeActivity`步骤设置的`DecorView`，因此会从`DecorView`开始执行整个视图树的`measure()`，`layout()`，`draw()`。



## Measure

Measure用来测量View的宽高。

核心思路：

* `MeasureSpec`对象包装了parent对child的大小的限制要求。
* parent需要综合自身的`MeasureSpec`和child的`LayoutParams`参数计算child适当的`MeasureSpec`。
* parent调用child的`View.measure()`方法让child自己计算measure大小。同时使用`MeasureSpec`传入相应的限制要求。
    * child的`View.measure()`方法会调用`View.onMeasure()`，并在里面实现具体的大小计算，共使用`View.setMeasuredDimension()`方法进行记录。
    * child的`View.measure()`方法会调用`View.measure()`，递归进行子View的Measure。

源码关键路径：
```java
ViewRootImpl.performTraversals()
ViewRootImpl.performMeasure()
View.measure()
FrameLayout.onMeasure()
ViewGroup.measureChildWithMargins()
ViewGroup.getChildMeasureSpec()
```

### MeasureSpec
`MeasureSpec`是用一个32位`int`值表示的，高2位表示mode，低30位表示size。

mode包括：

* `MeasureSpec.UNSPECIFIED` (0 << 30)
* `MeasureSpec.EXACTLY` (1 << 30)
* `MeasureSpec.AT_MOST` (2 << 30)

`MeasureSpec`类提供了方便的api进行mode和size拆分或合并：

```java
MeasureSpec.makeMeasureSpec()
MeasureSpec.getMode()
MeasureSpec.getSize()
```

### View.onMeasure()
源码关键路径：
```java
View.onMeasure()
View.getDefaultSize()
View.getSuggestMinimumWidth() / View.getSuggestMinimumHeight()
```

标准：
|Parent MS|EXACTLY|AT_MOST|UNSPECIFIED|
|-|:-:|:-:|:-:|
|Child Size|specSize|specSize|suggestMinimumSize|

SuggestMinimumWidth(Height)：
```java
return (mBackground == null) ? mMinHeight : max(mMinHeight, mBackground.getMinimumHeight());
```

### ViewGroup.onMeasure()
默认情况下，`ViewGroup`调用的是父类`View.onMeasure()`。

### 如何计算child的MeasureSpec限制
核心代码在`ViewGroup`的静态方法`getChildMeasureSpec()`方法中。

该方法综合了parent自身的`MeasureSpec`限制及child的`LayoutParams`需求，计算得到一个最合理的值。

标准：
|Child LP, Parent MS|EXACTLY|AT_MOST|UNSPECIFIED|
|:-:|:-:|:-:|:-:|
|x (>=0)|(x, EXACTLY)|(x, EXACTLY)|(x, EXACTLY)|
|MATCH_PARENT (-1)|(parentSize, EXACTLY)|(parentSize, AT_MOST)|(0, UNSPECIFIED)|
|WRAP_CONTENT (-2)|(parentSize, AT_MOST)|(parentSize, AT_MOST)|(0, UNSPECIFIED)|

说明：
* parentSize是减过child的margin和parent的padding后的值。
    ```java
    protected void measureChild(View child, int parentWidthMeasureSpec, int parentHeightMeasureSpec) {
        final LayoutParams lp = child.getLayoutParams();

        final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
                mPaddingLeft + mPaddingRight, lp.width);
        final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
                mPaddingTop + mPaddingBottom, lp.height);

        child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    }
    ```
* `(0, UNSPECIFIED)`在`targetSdkVersion < M`时，使用的是`0`。否则使用的是`parentSize`。
    ```java
    // View构造方法

    // In M and newer, our widgets can pass a "hint" value in the size
    // for UNSPECIFIED MeasureSpecs. This lets child views of scrolling containers
    // know what the expected parent size is going to be, so e.g. list items can size
    // themselves at 1/3 the size of their container. It breaks older apps though,
    // specifically apps that use some popular open source libraries.
    sUseZeroUnspecifiedMeasureSpec = targetSdkVersion < Build.VERSION_CODES.M;
    ```
    ```java
    // ViewGroup.getChildMeasureSpec()

    resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
    ```

### 为什么有两次Measure
第一次measure时无法确定`LayoutParams`为`MATCH_PARENT`的child具体的大小。需要在Parent确定大小后，再次调用Child的`View.measure()`方法进行测量。



## Layout

核心思路：
* parent调用child的`View.layout()`方法确定child的layout坐标。
    * child在`View.layout()`方法中调用`View.setFrame()`方法记录坐标。
    * child在`View.layout()`方法中调用`View.onLayout()`方法。
    * 重写`View.onLayout()`方法，递归进行子View的Layout。

### View.onLayout()

`View`因为没有子View，所以并不需要在`onLayout`中进行操作，这是一个空方法。

### ViewGroup.onLayout()

`ViewGroup.onLayout()`是一个抽象方法，不同的布局根据需求进行不同的排列。自定义`ViewGroup` 需要重写这个抽象方法。



## Draw

核心思路：
* parent调用child的`View.draw()`绘制child。
    * child在`View.draw()`方法中调用`View.drawBackgrond()`方法绘制背景。
    * child在`View.draw()`方法中调用`onDraw()`方法绘制自己的内容。
    * child在`View.draw()`方法中调用`dispatchDraw()`方法，递归进行子View的绘制。



## 主动触发Api

* `View.invalidate()`只会执行`View.onDraw()`
    
    > onDraw() will be called at some point in the future.
* `View.requestLayout()`只会执行`View.onMeasure()`和`View.onLayout()`方法。



# Reference

* [Android自定义view之measure、layout、draw三大流程](https://blog.csdn.net/zuguorui/article/details/70160323)

