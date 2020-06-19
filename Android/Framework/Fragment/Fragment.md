# Fragment

## 设计理念

* 将Activity的UI进行分块，方便复用与扩展。  

* 给UI提供生命周期。



## 生命周期

同`Activity`的生命周期类似，`Fragment`的生命周期也有`onCreate`，`onStart`等方法与`Activity`对应。实际上，`Fragment`的生命周期也依赖于添加到的`Activity`的生命周期。

`Fragment`完整生命周期如下图：

![](./images/fragment_lifecycle.png)

与绑定的`Activity`的生命周期对比如下图：

![img](./images/activity_fragment_lifecycle.png)

* `onAttach()`

  此方法会在`Fragment`添加到`Activity`中时被调用，可以用于接口设置。

* `onCreateView()`

  此方法会在`Fragment`第一次呈现界面时被调用，需要返回一个`View`对象作为整个`Fragment` 视图的根节点被添加到`Activity`的`Container`中。

* `onSaveInstanceState()`

  此方法会在绑定的`Activity`调用`onSaveStateInstance()`方法时被调用。在`Activity`非用户主动退出时（比如旋转屏幕），会被调用。可以在此时保存`Fragment`页面的状态，便于在`Fragment`重建时进行数据恢复。

### 实例：Fragmet2替换Fragment1

这个例子要考虑两种情况，`FragmentTransaction`是否调用`addToBackStack()`。

如果没有调用过`addToBackStack()`，当第fragment2替换fragment1时，fragment1将会被销毁。因此，fragment1会调用`onDestroyView()`，`onDestroy()`，`onDetach()`。

```
DynamicFragment2.onAttach()
DynamicFragment2.onCreate()
                                            DynamicFragment1.onPause()
                                            DynamicFragment1.onStop()
                                            DynamicFragment1.onDestroyView()
                                            DynamicFragment1.onDestroy()
                                            DynamicFragment1.onDetach()
DynamicFragment2.onCreateView()
DynamicFragment2.onActivityCreated()
DynamicFragment2.onStart()
DynamicFragment2.onResume()
```

如果调用过`addToBackStack()`，fragment1被替换时只会调用`onDestroyView()`，不会调用`onDestroy`和`onDetach()`

```
DynamicFragment2.onAttach()
DynamicFragment2.onCreate()
                                            DynamicFragment1.onPause()
                                            DynamicFragment1.onStop()
                                            DynamicFragment1.onDestroyView()
DynamicFragment2.onCreateView()
DynamicFragment2.onActivityCreated()
DynamicFragment2.onStart()
DynamicFragment2.onResume()
```



## 使用Fragment

使用`Fragment`我们需要定义一个类继承自`Framgnet`。通常我们需要实现的方法有静态构造方法，`onCreate()`，`onCreateView()`。

> 建议使用support库（androidx库）中的`Fragment`类，而不要直接使用`android.app.Fragment`。

### 静态构造方法

我们应该使用静态方法创建`Fragment`，并使用`Fragment.setArguments()`传入参数，而不要直接使用带参数的构造方法传递参数。这是因为系统因内存不足回收`Fragment`后，会对`Fragment`进行重建，重建时不会再次调用有参数的构造方法，只会调用默认构造方法，而参数将通过`arguments`重建，所以使用`arguments`进行参数传递。

```java
public static AnimeDetailFragment newInstance(Anime anime) {
    Bundle argument = new Bundle();
    argument.putParcelable(ARGUMENT_KEY_ANIME, anime);
    AnimeDetailFragment fragment = new AnimeDetailFragment();
    fragment.setArguments(argument);
    return fragment;
}
```

### 创建Fragment的界面

在`Fragment.onCreateView()`回调时创建`Fragment`的界面。

```Java
public View onCreateView(LayoutInflater inflater, ViewGroup container, Bundle savedInstanceState) {
    View fragmentView = inflater.inflate(R.layout.fragment_anime_list, container, false);
    AnimeRecyclerView animeRecyclerView = fragmentView.findViewById(R.id.recyclerView_anime);
    animeRecyclerView.setOnAnimeClickedListener(mOnAnimeClickedListener);
    return fragmentView;
}
```

方法中会传递`inflater`，`container`，`savedInstanceState`三个参数。

* `inflater`用于`inflate`一个`xml`获取视图对象。
* `container` 是`Fragment`添加到的`ViewGroup`
* `savedInstanceState`是`onSaveInstanceState()`方法保存的页面状态数据，可以用于界面的重建。

需要注意，调用`LayoutInflater.inflate()`方法时，`attachToRoot`参数需要传递`false`，因为`Fragment`内部实现会进行添加，如果设置`true`会产生重复添加子View的异常。

```
java.lang.IllegalStateException: The specified child already has a parent. You must call removeView() on the child's parent first.
```

### 添加到Activity

有两种方式可以添加到`Activity`，一种是在`Activity`的layout中静态注册`<fragment>`标签，还有一种是在运行时，通过`FragmentManager`的进行添加。

* 静态添加`Fragment`

  静态添加需要在Activity的Layout文件中提前声明Fragment的布局。

  ```xml
  <fragment
      android:id="@+id/fragment_animeList"
      android:name="zhaoyun.techstack.android.fragment.list.AnimeListFragment"
      android:layout_width="0dp"
      android:layout_height="0dp"
      android:layout_marginStart="8dp"
      android:layout_marginTop="8dp"
      android:layout_marginEnd="8dp"
      android:layout_marginBottom="8dp"
      app:layout_constraintBottom_toTopOf="@id/fragment_container"
      app:layout_constraintEnd_toEndOf="parent"
      app:layout_constraintStart_toStartOf="parent"
      app:layout_constraintTop_toTopOf="parent"
      app:layout_constraintVertical_weight="2"/>
  ```

* 动态添加`Fragment`

  动态添加需要在运行时调用`FragmentManager`的方法进行。

  ```java
  Fragment fragment = AnimeDetailFragment.newInstance(anime);
  getSupportFragmentManager()
          .beginTransaction()
          .setCustomAnimations(
                  R.anim.fragment_in,
                  R.anim.fragment_out,
                  R.anim.fragment_in,
                  R.anim.fragment_out
          )
          .replace(R.id.fragment_container, fragment)
          .addToBackStack(null)
          .commit();
  ```

  只有在调用`FragmentTransaction.commit()`后，才会被添加到`Activity`。并且，它也不会立刻执行，会post到主线程的`Handler`中。如果需要立刻执行，可以调用`FragmentManager.executePendingTransactions()` 方法。

  另外，有一个常见异常是：

  ```
  java.lang.IllegalStateException: Can not perform this action after onSaveInstanceState
  ```

  `commit()`方法不能在`Activity.onSaveInstanceState()`之后再调用，因为`Activity`是在这个方法里完成了所有`Fragment`状态的保存。

  因此，我们要尽量避免在异步回调时操作`Fragment`。如果确实有这样的需求，可以使用`FragmentManager.commitAllowingStateLoss()`方法。

### FragmentManager

有以下常用api：

* `add()` 添加`Fragment`到父容器。

  会依次执行`onAttach()`，`onCreate()`，`onCreateView()`，`onStart()`，`onResume()`。

* `remove()` 从父容器中删除`Fragment`。

  会依次执行`onPause()`，`onStop()`，`onDestroyView()`，`onDestroy()`，`onDetach()`。

  > 当然，也需要考虑有没有调用`addToBackStack()`方法。它会影响`onDestory()`，`onDetach()`是否被调用。

* `replace()` 从父容器中删除所有`Fragment`，再添加新的`Fragment`。

  所有删除的`Fragment`会执行`remove()`操作里的回调。

  添加的`Fragment`会执行`add()`操作的里的回调。

  > 当然，也需要考虑有没有调用`addToBackStack()`方法。它会影响`onDestory()`，`onDetach()`是否被调用。

* `show()` 显示`Fragment`。

  不会执行生命周期方法，只是调整了`Fragment`的visibility。

* `hide()` 隐藏`Fragment`。

  不会执行生命周期方法，只是调整了`Fragment`的visibility。

### BackStack

可以使用`addToBackStack()`方法把操作加到BackStack中。用户点击Back键时会回退操作。

### Animation

可以使用`setCustomAnimations`设置fragment切换时的动画。



## 通信

`Fragment`与`Activity`通信分为三步实现：

1. `Fragment`中定义接口

   ```java
   public interface OnAnimeClickedListener {
       void onAnimeClicked(Anime anime);
   }
   ```

2. `Activity`实现接口

   ```java
   public class MainActivity extends AppCompatActivity implements AnimeListFragment.OnAnimeClickedListener
   ```

3. `Fragment.onAttach()`时获取接口

   ```java
   @Override
   public void onAttach(Context context) {
       super.onAttach(context);
   
       if (context instanceof OnAnimeClickedListener) {
           mOnAnimeClickedListener = (OnAnimeClickedListener) context;
       }
   }
   ```

与其他`Fragment`通信可以使用上述方法通过`Activity`转发。



## OverLapping

当`Activity`意外结束时（比如旋转屏幕），系统会重新创建`Activity`并试图还原视图状态，已经显示的`Fragment`也会被重新创建并恢复。

所以，如果我们在`Activity.onCreate()`方法中添加`Fragment`，会出现添加的`Fragment`和恢复的`Fragment`重叠的问题。

解决这个问题的办法非常简单，只需要判断`Activiy.onCreate()`方法中的`savedStateInstance`参数就能知道是否处于恢复状态。

```java
if (savedInstanceState == null) {
    createFragment();
}
```



## 黑科技

可以使用一个没有界面的空Fragment去代理Activity的方法，实现统一管理的目的。  
参考：[避免使用onActivityResult，提高代码可读性](https://mp.weixin.qq.com/s/w_n4ppDoWFTLttpdPx-Yww)。



## Reference

* [Fragments - Android Developers](https://developer.android.com/guide/components/fragments)
* [Android Fragment 非常详细的一篇](https://www.jianshu.com/p/11c8ced79193)
* [A diagram of the Android Activity / Fragment lifecycle](https://github.com/xxv/android-lifecycle)
* [避免使用onActivityResult，提高代码可读性](https://mp.weixin.qq.com/s/w_n4ppDoWFTLttpdPx-Yww)
* [《Android基础：Fragment，看这篇就够了》](https://mp.weixin.qq.com/s/dUuGSVhWinAnN9uMiBaXgw)





