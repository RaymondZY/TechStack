# NetworkLibraries

## 系统API

* HttpClient

  在Android6.0已经废弃，维护较复杂，不使用。

* HttpURLConnection

  在Android2.2以前连接池有Bug。在Android4.4以后底层实现改为使用OKHttp。

  轻量级，只有小的Demo使用。



## Okhttp

* 支持http2，内部使用NIO效率更高。



## Volley

### 优点

* 集成了ImageLoader，支持图片加载。
* 方便进行请求的取消。
* 网络请求排序，优先级。

### 缺点

* 大量数据消耗更多内存，内部Request，Response使用的Byte数组进行存储。



## Retrofit

### 优点

* Restful api，简化代码。
* 可以结合多种数据格式转换gson，protobuff等。
* 可以结合rxjava。

### 缺点

* 扩展性较差，封装的太好。
* 使用了统一converter。