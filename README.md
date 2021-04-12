- [视图&图像相关](#视图&图像相关)
    -   [为什么必须在主线程刷新UI](#为什么必须在主线程刷新UI)
    -   [什么是离屏渲染](#什么是离屏渲染)
    -   [设置cornerRadius一定会触发离屏渲染吗？](#设置cornerRadius一定会触发离屏渲染吗？)
- [开源项目](#开源项目)
    -   [AsyncDisplayKit](#AsyncDisplayKit)
    

## 视图&图像相关
### 为什么必须在主线程刷新UI

<details>
<summary> 参考内容 </summary>

- 线程：UIKit并不是一个线程安全的类，UI操作涉及到渲染访问各种View对象的属性，如果异步操作下会存在读写问题，而为其加锁则会耗费大量资源并拖慢运行速度。
- 事件：整个程序的起点UIApplication是在主线程进行初始化，所有的用户事件都是在主线程上进行传递（如点击、拖动），所以view只能在主线程上才能对事件进行响应。
- 渲染：由于图像的渲染需要以60帧的刷新率在屏幕上 同时 更新，在非主线程异步化的情况下无法确定这个处理过程能够实现同步更新。在子线程中如果要对UI 进行更新，必须等到该子线程运行结束才能把UI的更新提交给渲染服务。

</details>

### 什么是离屏渲染
<details>
<summary> 参考内容 </summary>

如果要在显示屏上显示内容，我们至少需要一块与屏幕像素数据量一样大的frame buffer，作为像素数据存储区域，而这也是GPU存储渲染结果的地方。如果有时因为面临一些限制，无法把渲染结果直接写入frame buffer，而是先暂存在另外的内存区域，之后再写入frame buffer，那么这个过程被称之为离屏渲染。

![参考图示](https://pic3.zhimg.com/80/v2-c448aaebe3cf19e37101ce16a799cdd2_720w.jpg)
渲染结果先经过了离屏buffer，再到frame buffer

</details>

### 设置cornerRadius一定会触发离屏渲染吗？
<details>
<summary> 参考内容 </summary>

不一定。
cornerRadius+clipsToBounds，原因就如同上面提到的，不得已只能另开一块内存来操作。而如果只是设置cornerRadius（如不需要剪切内容，只需要一个带圆角的边框），或者只是需要裁掉矩形区域以外的内容（虽然也是剪切，但是稍微想一下就可以发现，对于纯矩形而言，实现这个算法似乎并不需要另开内存），并不会触发离屏渲染。

</details>

## 开源项目
### AsyncDisplayKit
<details>
<summary> 参考内容 </summary>

>  实际上，从上面的一些解释也可以看出，ASDK最大的特点就是"异步"。
将消耗时间的渲染、图片解码、布局以及其它 UI 操作等等全部移出主线程，这样主线程就可以对用户的操作及时做出反应，来达到流畅运行的目的。Github：[AsyncDisplayKit](https://github.com/facebookarchive/AsyncDisplayKit)

- 布局：
iOS自带的Autolayout在布局性能上存在瓶颈，并且只能在主线程进行计算。（参考Auto Layout Performance on iOS）因此ASDK弃用了Autolayout，自己参考自家的ComponentKit设计了一套布局方式。

- 渲染：
对于大量文本，图片等的渲染，UIKit组件只能在主线程并且可能会造成GPU绘制的资源紧张。ASDK使用了一些方法，比如图层的预混合等，并且异步的在后台绘制图层，不阻塞主线程的运行。

- 系统对象创建与销毁：
UIKit组件封装了CALayer图层的对象，在创建、调整、销毁的时候，都会在主线程消耗资源。ASDK自己设计了一套Node机制，也能够调用。
</details>