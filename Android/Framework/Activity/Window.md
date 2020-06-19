# Window

Activity，Window和View有什么关系？要理清楚它们之间的关系，要从Activity的启动过程说起。



## Window的创建

在Activity启动的过程中，其中一个步骤是`ActivityStackSupervisor`通过Binder机制远程调用使`ActivityThread.performLaunchActivity()`。这个步骤会创建`Activity`对象，并调用`Activity.attach()`方法。

* `Activity.attach()`

  在`Activity.attach()`方法中，新建了一个`PhoneWindow`对象，并将调用`PhoneWindow.setWindowManager()`进行`WindowManager`的绑定。

  ```java
  final void attach(Context context, ActivityThread aThread,
              Instrumentation instr, IBinder token, int ident,
              Application application, Intent intent, ActivityInfo info,
              CharSequence title, Activity parent, String id,
              NonConfigurationInstances lastNonConfigurationInstances,
              Configuration config, String referrer, IVoiceInteractor voiceInteractor,
              Window window, ActivityConfigCallback activityConfigCallback, IBinder assistToken) {
      attachBaseContext(context);
  
      mFragments.attachHost(null /*parent*/);
      
      /* ---------- 创建Window ---------- */
      mWindow = new PhoneWindow(this, window, activityConfigCallback);
      mWindow.setWindowControllerCallback(this);
      mWindow.setCallback(this);
      mWindow.setOnWindowDismissedCallback(this);
      mWindow.getLayoutInflater().setPrivateFactory(this);
      if (info.softInputMode != WindowManager.LayoutParams.SOFT_INPUT_STATE_UNSPECIFIED) {
          mWindow.setSoftInputMode(info.softInputMode);
      }
      if (info.uiOptions != 0) {
          mWindow.setUiOptions(info.uiOptions);
      }
      /* ---------- End of 创建Window ---------- */
      
      mUiThread = Thread.currentThread();
  
      mMainThread = aThread;
      mInstrumentation = instr;
      mToken = token;
      mAssistToken = assistToken;
      mIdent = ident;
      mApplication = application;
      mIntent = intent;
      mReferrer = referrer;
      mComponent = intent.getComponent();
      mActivityInfo = info;
      mTitle = title;
      mParent = parent;
      mEmbeddedID = id;
      mLastNonConfigurationInstances = lastNonConfigurationInstances;
      if (voiceInteractor != null) {
          if (lastNonConfigurationInstances != null) {
              mVoiceInteractor = lastNonConfigurationInstances.voiceInteractor;
          } else {
              mVoiceInteractor = new VoiceInteractor(voiceInteractor, this, this,
                                                     Looper.myLooper());
          }
      }
  
      /* ---------- 绑定WindowManager ---------- */
      mWindow.setWindowManager(
          (WindowManager)context.getSystemService(Context.WINDOW_SERVICE),
          mToken, mComponent.flattenToString(),
          (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);
      if (mParent != null) {
          mWindow.setContainer(mParent.getWindow());
      }
      mWindowManager = mWindow.getWindowManager();
      /* ---------- End of 绑定WindowManager ---------- */
      
      mCurrentConfig = config;
  
      mWindow.setColorMode(info.colorMode);
  
      setAutofillOptions(application.getAutofillOptions());
      setContentCaptureOptions(application.getContentCaptureOptions());
  }
  ```

  * `Window`是一个抽象类，`PhoneWindow`是继承自它的实现类。

  * `WindowManger`是一个接口，`WindowManagerImpl`是它具体的实现类。

    `Window`类的注释上标注了它主要负责与`WindowManagerService`系统服务进行交互。

    ```java
    /**
     * The interface that apps use to talk to the window manager.
     */
    ```

    `Context.getService()`最终会通过`SystemServiceRegistry`类获取具体的服务。由此我们可以发现，`WindowManager`接口的具体实现是创建了一个`WindowManagerImpl`对象。

    ```java
    registerService(Context.WINDOW_SERVICE, WindowManager.class,
                    new CachedServiceFetcher<WindowManager>() {
                @Override
                public WindowManager createService(ContextImpl ctx) {
                    return new WindowManagerImpl(ctx);
                }});
    ```

  总的来说，在`Window`的创建过程中，一个`Activity`与一个`PhoneWindow`和`WindowManager`绑定。



## `DecorView`的创建

`ActivityThread.performLaunchActivity()`方法中，然后会回调`Activiy.onCreate()`方法，进而执行到`Activity.setContentView()`初始化`DecorView`和整个界面。

* `Activity`

  ```java
  public void setContentView(@LayoutRes int layoutResID) {
      getWindow().setContentView(layoutResID);
      initWindowDecorActionBar();
  }
  ```

* `Window`

  `Window.setContentView()`方法中，会先进行`DecorView`的初始化，然后再进行整个界面的渲染。

  在`installDecor()`方法中将会判断各种Window上设置的Feature，比如`FEATURE_NO_TITLE`，来进行特殊的处理。这也就是为什么`Window.requestFeature(Window.Feature_NO_TITLE)`方法必须在`Activity.setContentView()`之前调用的原因。

  ```java
  @Override
  public void setContentView(int layoutResID) {
      if (mContentParent == null) {
          /* ---------- 创建DecorView ---------- */
          installDecor();
          /* ---------- End of 创建DecorView ---------- */
      } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
          mContentParent.removeAllViews();
      }
  
      if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
          final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID,
                                                         getContext());
          transitionTo(newScene);
      } else {
          /* ---------- 界面渲染 ---------- */
          mLayoutInflater.inflate(layoutResID, mContentParent);
          /* -------------------- */
      }
      mContentParent.requestApplyInsets();
      final Callback cb = getCallback();
      if (cb != null && !isDestroyed()) {
          cb.onContentChanged();
      }
      mContentParentExplicitlySet = true;
  }
  ```

  

## `DecorView`与`Window`的绑定

在`ActivityThread.handleResumeActivity()`方法中，会调用到`Activity.makeVisible()`。

* `Activity`

  ```java
  void makeVisible() {
      if (!mWindowAdded) {
          ViewManager wm = getWindowManager();
          wm.addView(mDecor, getWindow().getAttributes());
          mWindowAdded = true;
      }
      mDecor.setVisibility(View.VISIBLE);
  }
  ```

* `WindowManagerImpl`

  会调用`WindowManagerGlobal.addView()`。

  ```java
  private final WindowManagerGlobal mGlobal = WindowManagerGlobal.getInstance();
  
  ...
      
  public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
      applyDefaultToken(params);
      mGlobal.addView(view, params, mContext.getDisplay(), mParentWindow);
  }
  ```

* `WindowManagerGlobal`

  `WindowManagerGlobal`是一个单例，意味着，一个应用将使用同一个单例对象。

  ```java
  public static WindowManagerGlobal getInstance() {
      synchronized (WindowManagerGlobal.class) {
          if (sDefaultWindowManager == null) {
              sDefaultWindowManager = new WindowManagerGlobal();
          }
          return sDefaultWindowManager;
      }
  }
  ```

  在`addView()`方法中，创建了`ViewRootImpl`对象，并将`DecorView`添加到`ViewRootImpl`中。

  ```java
  public void addView(View view, ViewGroup.LayoutParams params,
              Display display, Window parentWindow) {
      ...
  
      ViewRootImpl root;
      View panelParentView = null;
  
      synchronized (mLock) {
          ...
  
          root = new ViewRootImpl(view.getContext(), display);
  
          view.setLayoutParams(wparams);
  
          mViews.add(view);
          mRoots.add(root);
          mParams.add(wparams);
  
          // do this last because it fires off messages to start doing things
          try {
              root.setView(view, wparams, panelParentView);
          } catch (RuntimeException e) {
              // BadTokenException or InvalidDisplayException, clean up.
              if (index >= 0) {
                  removeViewLocked(index, true);
              }
              throw e;
          }
      }
  }
  ```

* `ViewRootImpl`

  `ViewRootImpl`是整个视图树的根节点，通过`DecorView`保存了整个试图树的结构信息。虽然它并不是一个`View`对象，只是实现了`ViewParent`接口，但是它是整个视图树对外的封装类，向外提供了`requestLayout()`，`invalidate()`等接口。

  这里会进而调用`WindowSession.addToDisplay()`，调用`WindowManagerService`添加视图。

  ```java
  public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
      synchronized (this) {
          if (mView == null) {
              mView = view;
              
              ...
                  
              requestLayout();
              
              ...
  
              res = mWindowSession.addToDisplay(...);
                  
              ...
      }
  }
  ```

* `Session`

  转发给`WindowManagerService`。

  ```java
  final WindowManagerService mService;
  
  ...
      
  public int addToDisplay(...) {
      return mService.addWindow(...);
  }
  ```

  

## 总结

每个`Activity`都拥有一个`PhoneWindow`，每个`PhoneWindow`内都有一个`WindowManagerImpl`负责与系统`WindowManagerService`进行交互。

在`Activity.attach()`时，`Window`对象被创建，并绑定`WindowManagerImpl`。

在`Activity.setContentView()`时，`Window`对象创建内部`DecorView`对象，进行整个视图树的初始化。

在`ActivityThread.handleResumeActivity()`时，调用`Activity.makeVisible()`，将`DecorView`绑定到`WindowManagerImpl`。

`Activiy`是一个上下文相关，生命周期相关，任务栈相关的实例，它本身并不直接操作视图树。

`Window`是一个视图树的顶层封装，用于和`WindowManager`交互，提供了一些接口来做统一的界面处理（如隐藏`TitleBar`等）。

`View`是视图树的底层实现，视图树中的每个节点都是`View`对象，它关心的粒度最小，只关心界面相关的问题。