# DataBinding

## XML处理

* 解析layout文件，将`<data>`和布局进行拆分，生成两个xml文件。

  * 原始layout上在view上增加了`tag`标记。

    `/intermediates/incremental/mergeDebugResources/stipped.dir/layout/activity_main.xml`

    ```xml
    <androidx.constraintlayout.widget.ConstraintLayout android:tag="layout/activity_main_0">
    
        <EditText
                  android:id="@+id/textView"
                  android:layout_width="match_parent"
                  android:layout_height="wrap_content"
                  android:layout_marginLeft="16dp"
                  android:layout_marginTop="16dp"
                  android:layout_marginRight="16dp"
                  android:hint="name"
                  android:tag="binding_1"                 
                  app:layout_constraintLeft_toLeftOf="parent"
                  app:layout_constraintRight_toRightOf="parent"
                  app:layout_constraintTop_toTopOf="parent" />
    </androidx.constraintlayout.widget.ConstraintLayout>
    ```

    * 有binding的view：binding_1，binding_2，...

    * 没有binding的view：layout/activity_main_0，...

  * 将binding相关属性保存到了另一个xml

    `/intermediates/data_binding_layout_info_type_merge/debug/out/activity_main-layout.xml`

    ```xml
    <?xml version="1.0" encoding="utf-8" standalone="yes"?>
    <Layout directory="layout" filePath="app\src\main\res\layout\activity_main.xml" isBindingData="true" isMerge="false"
            layout="activity_main" modulePackage="zhaoyun.teckstack.android.databinding" rootNodeType="androidx.constraintlayout.widget.ConstraintLayout">
        <Variables name="viewModel" declared="true" type="zhaoyun.teckstack.android.databinding.ViewModel">
            <location endLine="9" endOffset="68" startLine="7" startOffset="8" />
        </Variables>
        <Targets>
            <Target tag="layout/activity_main_0" view="androidx.constraintlayout.widget.ConstraintLayout">
                <Expressions />
                <location endLine="65" endOffset="55" startLine="12" startOffset="4" />
            </Target>
            <Target id="@+id/textView" tag="binding_1" view="EditText">
                <Expressions>
                    <Expression attribute="android:text" text="viewModel.mUser.mName">
                        <Location endLine="25" endOffset="51" startLine="25" startOffset="12" />
                        <TwoWay>true</TwoWay>
                        <ValueLocation endLine="25" endOffset="49" startLine="25" startOffset="29" />
                    </Expression>
                </Expressions>
                <location endLine="28" endOffset="55" startLine="17" startOffset="8" />
            </Target>
            <Target id="@+id/editText" tag="binding_2" view="EditText">
                <Expressions>
                    <Expression attribute="android:text" text="viewModel.mUser.mAge">
                        <Location endLine="38" endOffset="50" startLine="38" startOffset="12" />
                        <TwoWay>true</TwoWay>
                        <ValueLocation endLine="38" endOffset="48" startLine="38" startOffset="29" />
                    </Expression>
                </Expressions>
                <location endLine="41" endOffset="65" startLine="30" startOffset="8" />
            </Target>
            <Target id="@+id/button_change" tag="binding_3" view="Button">
                <Expressions>
                    <Expression attribute="android:onClick" text="viewModel.mChangeListener">
                        <Location endLine="48" endOffset="57" startLine="48" startOffset="12" />
                        <TwoWay>false</TwoWay>
                        <ValueLocation endLine="48" endOffset="55" startLine="48" startOffset="31" />
                    </Expression>
                </Expressions>
                <location endLine="52" endOffset="65" startLine="43" startOffset="8" />
            </Target>
            <Target id="@+id/button_log" tag="binding_4" view="Button">
                <Expressions>
                    <Expression attribute="android:onClick" text="viewModel.mLogListener">
                        <Location endLine="59" endOffset="54" startLine="59" startOffset="12" />
                        <TwoWay>false</TwoWay>
                        <ValueLocation endLine="59" endOffset="52" startLine="59" startOffset="31" />
                    </Expression>
                </Expressions>
                <location endLine="63" endOffset="70" startLine="54" startOffset="8" />
            </Target>
        </Targets>
    </Layout>
    ```

    详细记录了被绑定的属性。



## 获取ViewDataBinding

`DataBindingUtil.setContentView()`这个方法会返回当前`Activity`对应的`ViewDataBinding`对象。

这个对象的类也是apt动态生成的。

然后再从根布局`android.R.id.content`上递归进行绑定。

```java
public static <T extends ViewDataBinding> T setContentView(
    @NonNull Activity activity,
    int layoutId, @Nullable DataBindingComponent bindingComponent) {
    activity.setContentView(layoutId);
    View decorView = activity.getWindow().getDecorView();
    // 找到根布局
    ViewGroup contentView = (ViewGroup) decorView.findViewById(android.R.id.content);
    // 返回ViewDataBinding对象
    return bindToAddedViews(bindingComponent, contentView, 0, layoutId);
}

private static <T extends ViewDataBinding> T bindToAddedViews(
    DataBindingComponent component,
    ViewGroup parent, int startChildren, int layoutId) {
    final int endChildren = parent.getChildCount();
    final int childrenAdded = endChildren - startChildren;
    if (childrenAdded == 1) {
        final View childView = parent.getChildAt(endChildren - 1);
        return bind(component, childView, layoutId);
    } else {
        final View[] children = new View[childrenAdded];
        for (int i = 0; i < childrenAdded; i++) {
            children[i] = parent.getChildAt(i + startChildren);
        }
        return bind(component, children, layoutId);
    }
}

static <T extends ViewDataBinding> T bind(
    DataBindingComponent bindingComponent, View[] roots,
    int layoutId) {
    return (T) sMapper.getDataBinder(bindingComponent, roots, layoutId);
}
```

走到`DataBinderMapperImpl.getDataBinder()`方法。这个类是自动生成的，通过当前ContentView创建对应的ViewDataBinding对象。

```java
public ViewDataBinding getDataBinder(DataBindingComponent component, View view, int layoutId) {
    int localizedLayoutId = INTERNAL_LAYOUT_ID_LOOKUP.get(layoutId);
    if(localizedLayoutId > 0) {
        final Object tag = view.getTag();
        if(tag == null) {
            throw new RuntimeException("view must have a tag");
        }
        switch(localizedLayoutId) {
            case  LAYOUT_ACTIVITYMAIN: {
                if ("layout/activity_main_0".equals(tag)) {
                    return new ActivityMainBindingImpl(component, view);
                }
                throw new IllegalArgumentException("The tag for activity_main is invalid. Received: " + tag);
            }
        }
    }
    return null;
}
```

根据View上的Tag，映射到`ActivityMainBindingImpl`对象。



## ViewDataBinding的初始化

```java
public ActivityMainBindingImpl(@Nullable androidx.databinding.DataBindingComponent bindingComponent, @NonNull View root) {
    this(bindingComponent, root, mapBindings(bindingComponent, root, 5, sIncludes, sViewsWithIds));
}
```

在构造方法中调用了`mapBindings()`方法，这个方法会去解析View上的Tag，并把和绑定相关的View保存下来。



## Model -> View

向Model注册回调，值发生变化后，进行`requestRebind()`，使用`Choreographer`注册`FrameCallback`，执行`RebindRunnable`，进行界面更新。

```java
private void handleFieldChange(int mLocalFieldId, Object object, int fieldId) {
    if (mInLiveDataRegisterObserver) {
        // We're in LiveData registration, which always results in a field change
        // that we can ignore. The value will be read immediately after anyway, so
        // there is no need to be dirty.
        return;
    }
    boolean result = onFieldChange(mLocalFieldId, object, fieldId);
    if (result) {
        requestRebind();
    }
}

protected void requestRebind() {
    if (mContainingBinding != null) {
        mContainingBinding.requestRebind();
    } else {
        final LifecycleOwner owner = this.mLifecycleOwner;
        if (owner != null) {
            Lifecycle.State state = owner.getLifecycle().getCurrentState();
            if (!state.isAtLeast(Lifecycle.State.STARTED)) {
                return; // wait until lifecycle owner is started
            }
        }
        synchronized (this) {
            if (mPendingRebind) {
                return;
            }
            mPendingRebind = true;
        }
        if (USE_CHOREOGRAPHER) {
            // FrameCallback
            mChoreographer.postFrameCallback(mFrameCallback);
        } else {
            mUIThreadHandler.post(mRebindRunnable);
        }
    }
}

mFrameCallback = new Choreographer.FrameCallback() {
    @Override
    public void doFrame(long frameTimeNanos) {
        mRebindRunnable.run();
    }
};

private final Runnable mRebindRunnable = new Runnable() {
    @Override
    public void run() {
        synchronized (this) {
            mPendingRebind = false;
        }
        processReferenceQueue();

        if (VERSION.SDK_INT >= VERSION_CODES.KITKAT) {
            // Nested so that we don't get a lint warning in IntelliJ
            if (!mRoot.isAttachedToWindow()) {
                // Don't execute the pending bindings until the View
                // is attached again.
                mRoot.removeOnAttachStateChangeListener(ROOT_REATTACHED_LISTENER);
                mRoot.addOnAttachStateChangeListener(ROOT_REATTACHED_LISTENER);
                return;
            }
        }
        // 回调给生成的类ActivityMainBindingImpl
        executePendingBindings();
    }
};
```



## View -> Model

在`ActivityMainBindingImpl`中初始化了接口回调，向View进行注册。