# Bitmap

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

### 磁盘缓存

使用`DiskLRUCache`三方库。

### 从网络获取图片

