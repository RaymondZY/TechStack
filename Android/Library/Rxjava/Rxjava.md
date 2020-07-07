# Rxjava

基于事件流的响应式编程框架。观察者模式，但是观察者只有一个。

## 重要的类

* `Observable`：被观察者
* `Observer`：观察者
  * `onNext()`：接收事件回调
  * `onError()`：错误回调
  * `onComplete()`：完成回调
* `ObservableOnSubscribe`：接口类，需要实现`subscribe()`方法，提供事件的发射流程。`Observable`创建时需要传递这个接口的实现方式来定义订阅开启时的执行逻辑。

## 操作符

### 创建型

* **create**

  <img src="./images/create.png" style="zoom:30%;" align="left" />

* **just**

  <img src="./images/just.png" style="zoom:30%;" align="left" />

* **fromArray**

  <img src="./images/from.png" style="zoom:30%;" align="left" />

* **empty**

  <img src="./images/empty.png" style="zoom:30%;" align="left" />

* **range**

  <img src="./images/range.png" style="zoom:30%;" align="left" />

### 变换型

* **map**

  <img src="./images/map.png" style="zoom:30%;" align="left" />

* **flatMap**

  <img src="./images/flatMap.png" style="zoom:30%;" align="left" />

* **concatMap**

  <img src="./images/concatMap.png" style="zoom:30%;" align="left" />

* **groupBy**

  <img src="./images/groupBy.png" style="zoom:30%;" align="left" />

* **buffer**

  <img src="./images/buffer.png" style="zoom:30%;" align="left" />

### 过滤型

* **filter**

  <img src="./images/filter.png" style="zoom:30%;" align="left" />

* **take**

  <img src="./images/take.png" style="zoom:30%;" align="left" />

* **distinct**

  <img src="./images/distinct.png" style="zoom:30%;" align="left" />

* **elementAt**

  <img src="./images/elementAt.png" style="zoom:30%;" align="left" />

### 条件型

* **all**

  <img src="./images/all.png" style="zoom:30%;" align="left" />

* **contains**

  <img src="./images/contains.png" style="zoom:30%;" align="left" />

* **isEmpty**

  <img src="./images/isEmpty.png" style="zoom:30%;" align="left" />

* **exists**

  <img src="./images/exists.png" style="zoom:30%;" align="left" />

### 合并型

* **startWith**

  <img src="./images/startWith.png" style="zoom:30%;" align="left" />

* **concat**

  <img src="./images/concat.png" style="zoom:30%;" align="left" />

* **merge**

  <img src="./images/merge.png" style="zoom:30%;" align="left" />

* **zip**

  <img src="./images/zip.png" style="zoom:30%;" align="left" />

### 异常型

* **onErrorReturn**

  <img src="./images/onErrorReturn.png" style="zoom:30%;" align="left" />

* **onErrorResumeNext**

  <img src="./images/onErrorResumeNext.png" style="zoom:30%;" align="left" />

* **onExceptionResumeNext**

  <img src="./images/onExceptionResumeNextViaObservable.png" style="zoom:30%;" align="left" />

* **retry**

  <img src="./images/retry.png" style="zoom:30%;" align="left" />



## 线程切换

![](./images/RxJava_Thread.jpg)



## 背压

使用`Flowable`，而不用`Observable`，下游使用`Subscriber`。

阻塞后放入缓存池。

### 策略

* `BackpressureStrategy.ERROR`：放入缓存池，缓存池满了，抛出异常。
* `BackpressureStrategy.ERROR`：放入缓存池，等待下游处理。
* `BackpressureStrategy.DROP`：放入缓存池，缓存池满了，抛弃后面发射的。