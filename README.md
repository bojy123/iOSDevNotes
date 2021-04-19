- [RunTime](#RunTime)
    -   [load和initialize方法的区别](#load和initialize方法的区别)
    -   [category和extension的区别](#category和extension的区别)
    -   [为什么Runtime可以给category添加属性](#为什么Runtime可以给category添加属性)
    -   [iOS内存布局](#iOS内存布局)
- [视图和图像](#视图和图像)
    -   [为什么必须在主线程刷新UI](#为什么必须在主线程刷新UI)
    -   [什么是离屏渲染](#什么是离屏渲染)
    -   [设置cornerRadius一定会触发离屏渲染吗](#设置cornerRadius一定会触发离屏渲染吗)
    -   [如何优化离屏渲染](#如何优化离屏渲染)
    -   [UIView和CAlayer](#UIView和CAlayer)
- [性能优化](#性能优化)
     -   [iOS如何监控线程卡顿](#iOS如何监控线程卡顿)
     -   [FPS监测](#FPS监测)
- [架构](#架构)
    -   [模块解耦](#模块解耦)
- [设计模式](#设计模式)
    -   [六大设计模式原则](#六大设计模式原则)  
- [开源项目](#开源项目)
    -   [AsyncDisplayKit](#AsyncDisplayKit)
    -   [BeeHive](#BeeHive)
   
   
## RunTime
### load和initialize方法的区别
<details>
<summary> 参考内容 </summary>

注意点:
+initialize和+load的很大区别是，+initialize是通过objc_msgSend进行调用的，所以有以下特点

- 如果子类没有实现+initialize，会调用父类的+initialize（所以父类的+initialize可能会被调用多次）
- 如果分类实现了+initialize，就覆盖类本身的+initialize调用

1.调用方式
- load是根据函数地址直接调用
- initialize是通过objc_msgSend调用

2.调用时刻
- load是runtime加载类、分类的时候调用（只会调用1次）
- initialize是类第一次接收到消息的时候调用，每一个类只会initialize一次（父类的initialize方法可能会被调用多次）



3.load、initialize的调用顺序？

**load:**
1.先调用类的load
- 先编译的类，优先调用load
- 调用子类的load之前，会先调用父类的load

2.再调用分类的load
- 先编译的分类，优先调用load

**initialize:**
1.先初始化父类
2.再初始化子类（可能最终调用的是父类的initialize方法）
</details>

### category和extension的区别
<details>
<summary> 参考内容 </summary>

> extension可以添加实例变量，而category是无法添加实例变量的（因为在运行期，对象的内存布局已经确定，如果添加实例变量就会破坏类的内部布局，这对编译型语言来说是灾难性的）。

- extension在编译期决议，它就是类的一部分，但是category则完全不一样，它是在运行期决议的。extension在编译期和头文件里的@interface以及实现文件里的@implement一起形成一个完整的类，它、extension伴随类的产生而产生，亦随之一起消亡。

- extension一般用来隐藏类的私有信息，你必须有一个类的源码才能为一个类添加extension，所以你无法为系统的类比如NSString添加extension，除非创建子类再添加extension。而category不需要有类的源码，我们可以给系统提供的类添加category。

- extension可以添加实例变量，而category不可以。

- extension和category都可以添加属性，但是category的属性不能生成成员变量和getter、setter方法的实现。

</details>

### 为什么Runtime可以给category添加属性

<details>
<summary> 参考内容 </summary>

关联对象都由AssociationsManager管理，AssociationsManager里面是由一个静态AssociationsHashMap来存储所有的关联对象的。这相当于把所有对象的关联对象都存在一个全局map里面。而map的的key是这个对象的指针地址（任意两个不同对象的指针地址一定是不同的），而这个map的value又是另外一个AssAssociationsHashMap，里面保存了关联对象的kv对。

</details>

### iOS内存布局
<details>
<summary> 参考内容 </summary>

iOS程序安装之后，是以Mach-o文件的格式保存在iOS设备里面，当启动程序时，对应的Mach-o文件就会被加载进内存。内存布局已确定，寻址快，开销小。
![参考图示](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91cGxvYWQtaW1hZ2VzLmppYW5zaHUuaW8vdXBsb2FkX2ltYWdlcy8xMzg3NDcyLTUzMzgxNDU1ZGIzYmIwYzkucG5n?x-oss-process=image/format,png)

</details>
   
## 视图和图像
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

### 设置cornerRadius一定会触发离屏渲染吗
<details>
<summary> 参考内容 </summary>

不一定。
cornerRadius+clipsToBounds，原因就如同上面提到的，不得已只能另开一块内存来操作。而如果只是设置cornerRadius（如不需要剪切内容，只需要一个带圆角的边框），或者只是需要裁掉矩形区域以外的内容（虽然也是剪切，但是稍微想一下就可以发现，对于纯矩形而言，实现这个算法似乎并不需要另开内存），并不会触发离屏渲染。

</details>

### 如何优化离屏渲染
<details>
<summary> 参考内容 </summary>

- 应用AsyncDisplayKit(Texture)作为主要渲染框架，对于文字和图片的异步渲染操作交由框架来处理。

- 对于图片的圆角，统一采用“precomposite”的策略，也就是不经由容器来做剪切，而是预先使用CoreGraphics为图片裁剪圆角。

- 对于视频的圆角，由于实时剪切非常消耗性能，我们会创建四个白色弧形的layer盖住四个角，从视觉上制造圆角的效果。

- 对于view的圆形边框，如果没有backgroundColor，可以放心使用cornerRadius来做。
- 对于所有的阴影，使用shadowPath来规避离屏渲染。
- 对于特殊形状的view，使用layer mask并打开shouldRasterize来对渲染结果进行缓存。
- 对于模糊效果，不采用系统提供的UIVisualEffect，而是另外实现模糊效果（CIGaussianBlur），并手动管理渲染结果。

</details>

### UIView和CAlayer
<details>
<summary> 参考内容 </summary>

UIView主要处理事件传递，CAlayer主要做视图显示，之所以这样设计，是因为单一功能原则。

</details>

## 性能优化
### iOS如何监控线程卡顿
<details>
<summary> 参考内容 </summary>

**卡顿原因：**
- 死锁
- 抢锁
- 大量的Ui绘制,复杂的UI，图文混排
- 主线程大量IO、大量计算

**监控方法：**
- 首先，创建一个观察者runLoopObserver，用于观察主线程的runloop状态。同时，还要创建一个信号量dispatchSemaphore，用于保证同步操作。
- 其次，将观察者runLoopObserver添加到主线程runloop中观察。
- 然后，开启一个子线程，并且在子线程中开启一个持续的loop来监控主线程runloop的状态。
- 如果发现主线程runloop的状态卡在为BeforeSources或者AfterWaiting超过88毫秒时，即表明主线程当前卡顿。这时候，我们保存主线程当前的调用堆栈即可达成监控目的。

</details>

### FPS监测
<details>
<summary> 参考内容 </summary>

通常情况下，屏幕会保持60hz/s的刷新速度，每次刷新时会发出一个屏幕刷新信号，CADisplayLink允许我们注册一个与刷新信号同步的回调处理。可以通过屏幕刷新机制来展示fps值。

正常情况下，屏幕会保持60hz/s的刷新速度，所以1秒内fpsDisplayLinkAction:方法会调用60次。

</details>

## 架构
### 模块解耦

<details>
<summary> 参考内容 </summary>

- URLRouter：将不同的模块定义成为不同的URL，通过URL的形式进行跨模块调用。
- Target-Action：利用OC的runtime能力，动态的调用指定Target的action；
- ProtocolClass：模块实现指定的协议，依赖方通过协议对象以及协议方法对模块进行访问。

</details>

## 设计模式
### 六大设计模式原则
<details>
<summary> 参考内容 </summary>

- 单一功能原则：对象功能要单一，不要在一个对象里添加很多功能。
- 里氏替换原则：子类对象，可以替代基类对象。
- 依赖倒置原则：方法应该依赖抽象，不要依赖实例。iOS 开发就是高层业务方法依赖于协议。
- 接口隔离原则：接口的用途要单一，不要在一个接口上根据不同入参实现多个功能。
- 迪米特法则：类间解耦，低耦合，高内聚。
- 开放封闭原则：尽量通过扩展软件实体来解决需求变化，而不是通过修改已有的代码来完成变化。

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

### BeeHive
<details>
<summary> 参考内容 </summary>

>  BeeHive是用于iOS应用开发的App模块化编程的框架实现方案，吸收了Spring框架Service的理念来实现模块间的API耦合。Github：[BeeHive](https://github.com/alibaba/BeeHive)

</details>
