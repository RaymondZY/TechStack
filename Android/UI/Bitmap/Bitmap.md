# Bitmap

## Density

* densityDpi：每英寸有多少个像素点，具体计算方式是：

  > 对角线像素个数 / 屏幕对角线英寸

  当然这个是理论值，各个手机会在配置中定义机器的dpi值，在理论值基础上取一个整数。

* density：densityDpi / 160，如每英寸160个像素点时，density为1。

* dp/dip：device independent pixel，在不同dpi的设备上，对应的pixel值会进行缩放。标准为160dpi时，1dp就等于1px，类似的320dpi时，1dp等于2px。



## res文件夹

| 密度类型 | 分辨率    | 对角线长 | 像素密度 | dp/px      |      |      |
| -------- | --------- | -------- | -------- | ---------- | ---- | ---- |
| ldpi     | 240x320   | 400      | 120      | 1dp=0.75px |      |      |
| mdpi     | 320x480   | 576      | 160      | 1dp=1px    |      |      |
| hdpi     | 480x800   | 932      | 240      | 1dp=1.5px  |      |      |
| xhdpi    | 720x1280  | 1468     | 320      | 1dp=2px    |      |      |
| xxhdpi   | 1080x1920 | 2202     | 480      | 1dp=3px    |      |      |



## Bitmap.Config

* ALPHA_8：只有alpha信息，8位，1byte。
* RGB_565：一共16位，2byte。
* ARGB_4444：16位，2byte。
* ARGB_8888：32位，4byte。



## BitmapFactory.Options

* **inBitmap**：在解析Bitmap时重用该Bitmap，但是必须相同大小的Bitmap & inMutable = true 才可重用。
* **inMutable**：配置Bitmap是否可更改，如每隔几个像素给Bmp添加一条直线。
* **inPreferredConfig：Config**：颜色位数，默认值为Bitmap.Config.ARGB_888。
* **inJustDecodeBounds**：为true时仅返回 Bitmap 宽高等属性，返回bmp=null，为false时才 返回占内存的 bmp。
* **inSampleSize**：表示 Bitmap 的压缩比例，值必须 > 1 & 是2的幂次方。inSampleSize = 2 时，表示压缩宽高各1/2，最后返回原始图1/4大小的Bitmap。
* **inDensity**：表示 Bitmap 像素密度。
* **inTargetDensity**：表示 Bitmap 最终的像素密度。
* **inScreenDensity**：表示当前屏幕的像素密度。
* **inScaled**：默认为true，是否支持缩放，设置为true时，Bitmap将以 inTargetDensity 的值进行缩放。
* **outputWidth**：返回的 Bitmap的宽。
* **outputHeight**：返回的 Bitmap的高。
* inPremultiplied：默认true，一般不改变其值。
* inTempStorage：解码时的临时空间，建议16K。
* inDither：是否抖动，默认false（Android Depracated）。
* inPurgeable：当存储像素内存空间在系统内存不足时 是否可被回收（Android L Deprecated）。
* inInputShareable：是否可以共享一个 InputStream（Android L Deprecated）。
* inPreferQualityOverSpeed：为true时会优先保证Bitmap 质量，其次是解码速度（Android N Deprecated）。



## 内存大小

Bitmap实际宽高受到`inTargetDensity`，`inDensity`和`inSampleSize`的影响，会根据这三个值进行缩放。

```c++
mBitmapWidth = mOriginalWidth * (scale = opts.inTargetDensity / opts.inDensity) * 1 / inSampleSize;
mBitmapHeight = mOriginalHeight * (scale = opts.inTargetDensity / opts.inDensity) * 1 / inSampleSize;
```

最终计算：

```
图片宽 * (opt.inTargetDensity / opts.inDensity) / inSampleSize * 图片高 * (opt.inTargetDensity / opts.inDensity) / inSampleSize * 像素占用byte值。
```

另外还会受到`inScreenDensity`的影响。如果图片存放位置的density和`inScreenDensity`值相同，将不会进行`(scale = opts.inTargetDensity / opts.inDensity)`计算，但是还会经过`1 / inSampleSize`计算。

### api

通过`Bitmap.getAllowcationByteCount()`方法或`Bitmap.getByteCount()`获取。

`Bitmap.getAllowcationByteCount()`能获得更精准的值，因为可能存在Bitmap的重用，重用时，分配的内存大小大于实际pixel占用大小。



## 内存存放位置

* Android版本<3.0，像素数据分配在native堆，需要手动调用`recycle()`方法进行回收，难以操作。
* 3.0<=Android版本<8.0，像素分配在java堆，不用手动调用`recycle()`，Bitmap也能自动通过垃圾回收机制进行回收。并且还有`BitmapFinalizer`保证同时回收Native内存。
* 8.0<=Android版本，像素分配在native堆，减少了OOM的可能。



## 加载大图

### 按照分辨率进行缩放

```java
//1.获取手机的分辨率  获取windowmanager 实例
WindowManager wm = (WindowManager) getSystemService(WINDOW_SERVICE);
screenWidth = wm.getDefaultDisplay().getWidth();
screenHeight = wm.getDefaultDisplay().getHeight();

//2.把xxxx.jpg 转换成bitmap
//创建bitmap工厂的配置参数
BitmapFactory.Options options = new Options();
//=true返回一个null 没有bitmap:不去为bitmap分配内存 但是能返回图片的一些信息(宽和高)
options.inJustDecodeBounds = true;
BitmapFactory.decodeFile("/mnt/sdcard/xxxx.jpg",options);

//3.获取图片的宽和高
int imgWidth = options.outWidth;
int imgHeight = options.outHeight;

 //4.计算缩放比
int scalex = imgWidth/screenWidth;
int scaley = imgHeight /screenHeight;
scale =min(scalex, scaley,scale);

//5.按照缩放比显示图片,inSampleSize给图片赋予缩放比，其值大于1时，会按缩放比返回一个小图片用来节省内存
options.inSampleSize = scale;

//6.=false开始真正的解析位图,可显示位图
options.inJustDecodeBounds = false;
Bitmap bitmap = BitmapFactory.decodeFile("/mnt/sdcard/dog.jpg",options);
```

### 分区域显示

图片的分块加载在地图绘制的情况上最为明显，当想获取一张尺寸很大的图片的某一小块区域时，就用到了图片的分块加载，`BitmapRegionDecoder`类的功能就是加载一张图片的指定区域。



## 图片的三级缓存

### 内存缓存

使用`LRUCache`类进行内存的缓存。

`LRUCache`类内部使用的是`LinkedHashMap`作为存储结构，`LinkedHashMap`内部使用的是`HashMap`加上链表来维护最后使用的记录。

`LRUCache`内部操作通过`synchronized`关键字加锁，它是线程安全的。

保存`Bitmap`时，需要注意几点：

* 构造`LRUCache`时，传入最大占用内存的大小。
* 重写`LRUCache.sizeOf()`方法，返回`Bitmap`占用内存的大小。
* 重写`LRUCache.entryRemoved()`方法，进行必要的回收。

```java
mLruCache = new LruCache<String, Bitmap>(4 * 1024 * 1024) {

    @Override
    protected int sizeOf(String key, Bitmap bitmap) {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.KITKAT) {
            return bitmap.getAllocationByteCount();
        } else {
            return bitmap.getByteCount();
        }
    }

    @Override
    protected void entryRemoved(boolean evicted, String key, Bitmap oldValue, Bitmap newValue) {
        Log.d("DEBUG", "ImageLoader.entryRemoved() " + key);
    }
};
```



## Reference

* [Android性能优化：Bitmap详解&你的Bitmap占多大内存？](https://www.jianshu.com/p/4ba3e63c8cdc)

* [Android O Bitmap 内存分配](https://www.cnblogs.com/xiaji5572/p/7794083.html)