# Bitmap

## 加载大图

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





