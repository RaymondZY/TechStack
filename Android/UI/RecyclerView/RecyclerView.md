# Recyclerview

## vs. ListView

`RecyclerView`是一个用于替代`ListView`的组件，它相比`ListView`有以下一些优点。

* `RecyclerView`提供了默认的`ViewHolder`机制。
* `RecyclerView`拥有4级缓存，`ListView` 只有两级缓存。
* `RecyclerView`可以实现局部刷新。
* `RecyclerView`可以通过更改`LayoutManager`轻松更改布局方式。
* `RecyclerView`支持了`NestedScrollingChild`接口。



## 类结构

`RecyclerView`包括以下一些重要的内部类：

* `Adapter`：用于提供数据和绑定`ViewHolder`。
* `ViewHolder` ：用于实现`ViewHolder`优化。
* `LayoutManager`：负责`RecyclerView`视图测量布局。
* `ItemDecoration`：负责Item的绘制。
* `ItemAnimator`：负责动画。
* `Recycler`：负责Item缓存。



## 缓存

`RecyclerView`的多级缓存都定义在`Recycler`内部类中。

#### mChangedScrap

用于`PreLayout()`操作的缓存。

#### mAttachedScrap

屏幕内缓存。

当`RecyclerView`重新布局时，`LayoutManager`会调用`detachAndScrapAttachedViews()`把显示的`ViewHolder`一个个`detach`，然后缓存在`mAttachedScrap`中，等布局时会先从`mAttachedScrap`查找，再把`ViewHolder`一个个的放回`RecyclerView`中，如果放回后还有剩余的`ViewHolder`没有参加新布局，会从`mAttachedScrap`移到`mCachedViews`中。

复用`mAttachedScrap`中的`ViewHolder`时，不需要重新调用`onBindViewHolder()`方法进行数据绑定。

#### mCachedViews

屏幕外缓存。

当`RecyclerView`滚动时，对于那些不在`RecyclerView`中显示的`ViewHolder`，`LayoutManager`会调用`removeAndRecycleAllViews()`把这些已经移除的`ViewHolder`缓存在`mCacheViews`中。它的默认大小是2，当它满了的时候，就会利用先进先出原则，把老的`ViewHolder`移到`mRecyclerPool`中。

复用`mCachedViews`中的`ViewHolder`时，不需要重新调用`onBindViewHolder()`方法进行数据绑定。

#### mViewCacheExtension

用户自定义缓存，一般不实现。

#### mRecyclerPool

`ViewHolder`对象池。这一级的`ViewHolder`除了`ViewType`，其它状态都将被清除。

`mRecyclerPool`内部维护了一个`Map<Integer, SparseArray<ViewHolder>>`，对不同的`ViewType`维护了一个默认大小为5的对象池。

复用`mRecyclerPool`中的`ViewHolder`需要重新调用`onBindViewHolder()`方法进行数据的绑定。

#### 命中策略

`LayoutManager.getViewForPosition()`方法会调用到`Recycler.tryGetViewHolderForPositionByDeadline()`方法。

这个方法中定义了详细的缓存命中策略。

* 尝试从`mChangedScrap`中获取。

  这一次尝试只有在`PreLayout()`阶段才会进行。

* 调用`getScrapOrHiddenOrCachedHolderForPosition()`方法依次尝试从`mAttachedScrap`，`mHiddenViews`，`mCachedViews`获取`ViewHolder`。

  如果成功获取到`ViewHolder`，检验它的有效性，包括`ViewType`和`Stable Id`。如果校验失败，将`ViewHolder`回收进`mRecyclerPool`。

  这一次尝试是根据Position进行查找。

* 调用`getScrapOrCachedViewForId()`方法尝试从`mAttachedScrap`，`mCachedViews`获取`ViewHolder`。

  如果成功获取到`ViewHolder`，检验它的有效性，包括`ViewType`。如果校验失败，将`ViewHolder`回收进`mRecyclerPool`。

  这一次尝试是根据id进行查找，`Adapter`设置了`hasStableIds()`才会进行。

* 尝试从`ViewCacheExtension`中获取。

* 尝试从`mRecyclerPool`中获取。

  如果成功获取`ViewHolder`，需要重新调用`onBindViewHolder()`方法进行数据的绑定。

* 调用`createViewHolder()`创建一个新的`ViewHolder`。



## DiffUtil

使用`DiffUtil`类，计算数据集的变化范围。



## 优化

* 使用粒度更细的`notifyXXX()`接口，而不是全量的`notifyDatasetChanged()`。

  这一过程可以考虑使用`DiffUtil`类。

* 使用`RecyclerView.setItemViewCacheSize(size)`方法定制`mCachedViews`大小。

* 使用`RecycledViewPool.setMaxRecycledViews(viewType, count)`方法定制不同`ViewType`的对象池大小。

* 监听回调接口在`onCreateView()`时进行绑定，然后通过`ViewHolder.getAdapterPosition()`获取点击的位置。如果设置了Header，需要减去Header的数量。

* 如果`RecyclerView`中元素比较高，一屏只能显示一个时，滑动就会出现卡顿。

  可以重写`LayoutManager.getExtraLayoutSpace()`方法，让数据加载有额外的空间。

  ```java
  LinearLayoutManager linearLayoutManager = new LinearLayoutManager(this) {
      @Override
      protected int getExtraLayoutSpace(RecyclerView.State state) {
          return 300;
      }
  };
  ```

*  `RecyclerView`嵌套时进行`RecyclerViewPool`的复用。



## 场景

### 滑动删除

使用`ItemTouchHelper`。

### 图片加载

调用`RecyclerView.addOnScrollListener()`添加回调。在状态为`SCROLL_STATE_DRAGGING`或者`SCROLL_STATE_SETTLING`时暂停图片加载。在状态为`SCROLL_STATE_IDLE`时，恢复加载。

给`ImageView`设置`Tag`，避免图片错乱。



## Reference

* [Anatomy of RecyclerView: a Search for a ViewHolder](https://android.jlelse.eu/anatomy-of-recyclerview-part-1-a-search-for-a-viewholder-404ba3453714)
* [Anatomy of RecyclerView: a Search for a ViewHolder (continued)](https://android.jlelse.eu/anatomy-of-recyclerview-part-1-a-search-for-a-viewholder-continued-d81c631a2b91)
* [Android ListView 与 RecyclerView 对比浅析--缓存机制](https://mp.weixin.qq.com/s/-CzDkEur-iIX0lPMsIS0aA?)
* [RecyclerView剖析](https://blog.csdn.net/qq_23012315/article/details/50807224)
* [RecyclerView一些你可能需要知道的优化技术](https://www.jianshu.com/p/1d2213f303fc)