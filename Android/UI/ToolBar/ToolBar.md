# App Bar

## ActionBar v.s. ToolBar

ActionBar是一个旧组件，它从Android 3.0（API level 11）开始被加入。只要使用了默认主题的Activity，都会使用ActionBar作为AppBar。但是，随着Android版本的发展，不同的系统版本ActionBar提供的体验不相同。因此，发展出了support library版本的ToolBar。

我们应该尽量使用ToolBar作为AppBar，以在不同的系统版本上提供一致的用户体验。



## 使用ToolBar

设置Window的style为NoActionBar：

```xml
<resources>

    <!-- Base application theme. -->
    <style name="AppTheme" parent="Theme.AppCompat.Light.NoActionBar">
        <!-- Customize your theme here. -->
        <item name="colorPrimary">@color/colorPrimary</item>
        <item name="colorPrimaryDark">@color/colorPrimaryDark</item>
        <item name="colorAccent">@color/colorAccent</item>
    </style>

</resources>
```

设置Activity的布局：

```xml
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".MainActivity">

    <androidx.appcompat.widget.Toolbar
        android:id="@+id/toolbar"
        android:layout_width="match_parent"
        android:layout_height="?attr/actionBarSize"
        android:background="@color/colorPrimary"
        android:theme="@style/ThemeOverlay.AppCompat.ActionBar"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toTopOf="parent"
        app:navigationIcon="@drawable/baseline_menu_white_36"
        app:popupTheme="@style/ThemeOverlay.AppCompat.Light"
        app:subtitle="This is subtitle"
        app:subtitleTextColor="@android:color/secondary_text_dark"
        app:title="This is title"
        app:titleTextColor="@android:color/primary_text_dark" />

    <TextView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Hello World!"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintLeft_toLeftOf="parent"
        app:layout_constraintRight_toRightOf="parent"
        app:layout_constraintTop_toTopOf="parent" />

</androidx.constraintlayout.widget.ConstraintLayout>
```

