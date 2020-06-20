# Animation

## 分类

* View动画
* 帧动画
* 属性动画



## View动画

View动画作用在View上，系统提供了4种默认的View动画：Translate，Alpha，Rotate，Scale，并且可以将四种动画进行组合使用。

有两种方式进行动画的定义：

* 从xml中定义动画
* 从代码中动态创建动画

### XML中定义Animation

在`res/anim/`文件夹下创建XML文件定义动画效果。

```xml
<?xml version="1.0" encoding="utf-8"?>
<set xmlns:android="http://schemas.android.com/apk/res/android"
    android:duration="2000">

    <translate
        android:fromXDelta="0"
        android:fromYDelta="0"
        android:repeatCount="-1"
        android:repeatMode="reverse"
        android:toXDelta="300"
        android:toYDelta="300" />

    <alpha
        android:fromAlpha="1"
        android:repeatCount="-1"
        android:repeatMode="reverse"
        android:toAlpha="0.5" />

    <scale
        android:fromXScale="1"
        android:fromYScale="1"
        android:pivotX="50"
        android:pivotY="50"
        android:repeatCount="-1"
        android:repeatMode="reverse"
        android:toXScale="0.3"
        android:toYScale="0.3" />

    <rotate
        android:fromDegrees="0"
        android:pivotX="100"
        android:pivotY="100"
        android:repeatCount="-1"
        android:repeatMode="reverse"
        android:toDegrees="360" />

</set>
```

* `<translate>`

  * `fromXDelta`：开始X偏移量
  * `toXDelta`：结束X偏移量
  * `fromYDelta`：开始Y偏移量
  * `toYDelta`：结束Y偏移量

* `<alpha>`

  * `fromAlpha`：开始Alpha值
  * `toAlpha`：结束Alpha值

* `<scale>`

  * `fromXScale`：开始X方向缩放比例

  * `toXScale`：结束X方向缩放比例

  * `fromYScale`：开始Y方向缩放比例

  * `toYScale`：结束Y方向缩放比例

  * `pivotX`：X方向缩放轴

  * `pivotY`：Y方向缩放轴

    > 默认pivotX和pivotY为0，表示View的中心位置

* `<rotate>`

  * `fromDegrees`：开始旋转角度

  * `toDegress`：结束旋转角度

    > 取值从0到360，顺时针方向，默认为0度（3点钟方向）

  * `pivotX`：旋转轴X坐标

  * `pivotY`：旋转轴Y坐标

    > 默认pivotX和pivotY为0，表示View的中心位置

* `<?>`

  * `repeatCount`：重复播放次数，-1表示无限循环

  * `repeatMode`：值可以为反向播放`reverse`或者重新播放`restart`

  * `duration`：动画播放时长

    > 需要注意repeatCount和repeatMode必须设置在动画标签一级，设置在set一级不能起作用

### 代码中定义Animation

View动画对应的核心类是`Animation`类。系统实现的默认4种动画都继承自`Animation`类。

```java
private Animation createTranslateAnimation() {
    TranslateAnimation translateAnimation = new TranslateAnimation(0, 300, 0, 300);
    translateAnimation.setDuration(2000L);
    translateAnimation.setRepeatCount(Animation.INFINITE);
    translateAnimation.setRepeatMode(Animation.REVERSE);
    return translateAnimation;
}

private Animation createAlphaAnimation() {
    AlphaAnimation alphaAnimation = new AlphaAnimation(1, 0.5f);
    alphaAnimation.setDuration(2000L);
    alphaAnimation.setRepeatCount(Animation.INFINITE);
    alphaAnimation.setRepeatMode(Animation.REVERSE);
    return alphaAnimation;
}

private Animation createScaleAnimation() {
    ScaleAnimation scaleAnimation = new ScaleAnimation(1, 0.3f, 1, 0.3f, 50f, 50f);
    scaleAnimation.setDuration(2000L);
    scaleAnimation.setRepeatCount(Animation.INFINITE);
    scaleAnimation.setRepeatMode(Animation.REVERSE);
    return scaleAnimation;
}

private Animation createRotateAnimation() {
    RotateAnimation rotateAnimation = new RotateAnimation(0, 360, 100, 100);
    rotateAnimation.setDuration(2000L);
    rotateAnimation.setRepeatCount(Animation.INFINITE);
    rotateAnimation.setRepeatMode(Animation.REVERSE);
    return rotateAnimation;
}

private Animation createCombinedAnimation() {
    AnimationSet combined = new AnimationSet(true);
    combined.addAnimation(createTranslateAnimation());
    combined.addAnimation(createAlphaAnimation());
    combined.addAnimation(createScaleAnimation());
    combined.addAnimation(createRotateAnimation());
    return combined;
}
```

### 执行动画

* 通过调用`View.startAnimation()`开始动画。
* 通过调用`View.clearAnimation()`停止动画。

### 设置回调

通过`Animation.setAnimationListener()`设置回调监听动画执行状态。

或者使用`AnimationListenerAdapter`类简化使用。

```java
Animation.AnimationListener animationListener = new Animation.AnimationListener() {
    @Override
    public void onAnimationStart(Animation animation) {
        Log.d("DEBUG", "MainActivity.onAnimationStart()");
    }

    @Override
    public void onAnimationEnd(Animation animation) {
        Log.d("DEBUG", "MainActivity.onAnimationEnd()");
    }

    @Override
    public void onAnimationRepeat(Animation animation) {
        Log.d("DEBUG", "MainActivity.onAnimationRepeat()");
    }
};

animation.setAnimationListener(animationListener);
```

### 自定义Animation

自定义Animation需要继承`Animation`类，并重写`Animation.applyTransformation()`方法。

```java
public class CustomAnimation extends Animation {

    @Override
    public void initialize(int width, int height, int parentWidth, int parentHeight) {
        super.initialize(width, height, parentWidth, parentHeight);
    }

    @Override
    protected void applyTransformation(float interpolatedTime, Transformation transformation) {
        transformation.getMatrix().setTranslate(
                (float) Math.sin(interpolatedTime * 50) * 8,
                (float) Math.sin(interpolatedTime * 50) * 8
        );
    }
}
```

* `interpolatedTime`：动画执行进度，值从0到1
* `transformation`：通过操作`transformation.getMatrix()`可以改变View的Canvas画布



## 帧动画

帧动画的原理是将动画拆分成一帧一帧的图像，它使用起来简单方便，但是却容易引起内存溢出。

### XML中定义帧动画

在`res/drawable/`下创建XML文件定义帧动画。

```xml
<animation-list xmlns:android="http://schemas.android.com/apk/res/android">
    <item
        android:drawable="@drawable/eva1"
        android:duration="1000" />
    <item
        android:drawable="@drawable/eva2"
        android:duration="1000" />
    <item
        android:drawable="@drawable/eva3"
        android:duration="1000" />
</animation-list>
```

* `<item>`
  * `drawable`：显示的drawable
  * duration：显示时长

将AnimatinDrawable设置给ImageView显示动画。

```xml
<ImageView
    android:id="@+id/imageView"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    app:layout_constraintBottom_toBottomOf="parent"
    app:layout_constraintEnd_toEndOf="parent"
    app:layout_constraintStart_toStartOf="parent"
    app:layout_constraintTop_toTopOf="parent"
    app:srcCompat="@drawable/frame"
    tools:ignore="ContentDescription" />
```

### 执行动画

* 通过调用`AnimationDrawable.start()`方法开始动画。
* 通过调用`AnimationDrawable.stop()`方法停止动画。



## 属性动画

与View动画不同，动画的变化不仅限于View，可以实现任意对象的变化。

主要有两个系统类提供属性动画：`ValueAnimator`和`ObjectAnimator`。

`ValueAnimator`不直接改变目标对象的属性，只提供变化的属性值。

`ObjectAnimator`直接改变目标对象的属性。

### ValueAnimator

#### 创建

通过`ValueAnimator.ofInt()`，`ValueAnimator.ofFloat()`，`ValueAnimator.ofObject()`等方法定义属性的变化范围，并获取`ValueAnimator`对象。

例如，属性的变化范围是从400到200，可以使用`ValueAnimator.ofInt()`方法。

```java
ValueAnimator valueAnimator = ValueAnimator.ofInt(400, 200);
```

#### 设置参数

`ValueAnimator`对象可以设置动画的其它一些属性，例如`Interpolator`，`Duration`等，与`Animation`类相似。

```java
valueAnimator.setDuration(2000);
valueAnimator.setRepeatCount(-1);
valueAnimator.setRepeatMode(ValueAnimator.REVERSE);
valueAnimator.setInterpolator(new LinearInterpolator());
```

#### 执行动画

 通过`ValueAnimator.start()`方法开始动画。

通过`ValueAnimator.cancel()`方法停止动画。

#### 设置监听

`ValueAnimator`需要设置`ValueAnimator.AnimatorUpdateListener`回调接口来具体处理属性值。

例如，将变化的int值设置给TextView的宽高。

```java
valueAnimator.addUpdateListener(animation -> {
    int size = (int) animation.getAnimatedValue();
    ViewGroup.LayoutParams layoutParams = textView.getLayoutParams();
    layoutParams.width = size;
    layoutParams.height = size;
    textView.setLayoutParams(layoutParams);
});
```

#### 变化的属性为Object

如果变化的属性是一个Object类型，不是原始数值类型，那么需要使用`ValueAnimator.ofObject()`方法进行构造。

构造函数中需要传入一个`TypeEvaluator`类型的参数，它的作用是Evaluate动画不同阶段属性具体的值。

例如，设置变化的属性为`Point`对象。

```java
ValueAnimator valueAnimator = ValueAnimator.ofObject(
    (fraction, startValue, endValue) -> {
        Point startPoint = (Point) startValue;
        Point endPoint = (Point) endValue;
        int x = (int) ((endPoint.x - startPoint.x) * fraction + startPoint.x);
        int y = (int) ((endPoint.y - startPoint.y) * fraction + startPoint.y);
        return new Point(x, y);
    },
    new Point(0, 0),
    new Point(100, 100)
);
```

### ObjectAnimator

`ObjectAnimator`将属性的变化作用于一个`Target`对象，动画执行的过程中自动更改`Target`对象属性的值。

`ObjectAnimator`继承自`ValueAnimator`，内部的实现是通过反射去操作`Target`对象属性的get和set方法。

#### 创建

`Primitives`对应的`ObjectAnimator`创建需要传入`Target`对象，以及需要操作的属性名称。

需要保证属性拥有get和set方法。

```java
ObjectAnimator objectAnimator = ObjectAnimator.ofFloat(textView, "alpha", 1, 0.5f);
```

#### 设置参数

同`ValueAnimator`。

#### 执行动画

同`ValueAnimator`。

#### 设置监听

同`ValueAnimator`。

#### 变化的属性为Object

与`ValueAnimator`类似，需要设置`TypeEvaluator`。



## 插值器与估值器

插值器控制的是动画执行速度，比如加速减速、匀速、减速加速等。

估值器是用来计算动画中间一帧属性的值。