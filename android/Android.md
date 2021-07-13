# 一 消息机制

## 1.1 java层

<img src="Message.png" alt="map" style="zoom:50%;" />

## Message 

消息体，链表结构

每个Message都会绑定一个Handler作为target

Message维护了一个静态的sPool作为消息池，通过obtain方法复用回收的message

### MessageQueue

消息队列

`mMessages`成员为消息队列, 按时间升序排列

`mIdleHandlers`成员为IdleHandler列表，一个List

`next()`方法用来提取一条消息

当最近的一条消息也是延迟消息时,调用 `nativePollOnce`方法阻塞等待

`nativePollOnce`方法会完成一次native层的消息轮询

如果链表头的消息已经到达触发时间,则返回该消息

否则计算延迟时间，执行所有IdleHandler

如果执行了IdleHandler,从新轮询链表头，否则阻塞等待

`enqueueMessage`在合适位置插入一条消息

如果插入到链表头，则唤醒线程

唤醒线程通过nativeWake方法，以mPtr表示唤醒那个线程。该值有nativeInit方法产生

同步消息插入链表中部，不唤醒线程

异步消息插入链表中部，且链表头是同步栅栏，唤醒线程



## Handler

最终都会执行sendMessageAtTime方法，再往下是 enqueueMessage方法，转调MessageQueue的enqueue

最终消息会通过dispatch方法被分派给自己

如果callback成员不为空，说明是一个Runnable则执行run方法

如果mCallback成员不为空，说明是一个Callback对象，执行起handleMessage方法

否则执行`Handler#handleMessage`方法



## Looper

消息泵,每个线程最多有一个

私有构造方法 + ThreadLocal 确保线程唯一

loop方法死循环，从messagequeue中取消息

取出消息后，将消息dispatch给其target

之后回收消息



### 异步消息 & 同步屏障

消息默认均为同步消息 

调用`setAsynchronous`方法会将消息设置为异步消息

`MessageQueue#postSyncBarrier()`方法会向messageque最前端插入一个同步屏障

有了同步屏障之后，`MessageQueue#next()`只会返回屏障后面的异步消息，而忽略所有同步消息

异步消息同样要遵守时间约定，没到到达预定时间时，消息队列会阻塞

`MessageQueue#removeSyncBarrier`会移除一条同步屏障,根据token

postSyncBarrier是hide方法，只在`VRI#scheduleTraversals`进行了调用。目的提高UI刷新相关message的优先级



## 1.2 native层

<img src="NativeMessage.png" alt="map" style="zoom:50%;" />

### 1.2.1 启动阶段

java层Looper创建时，会创建java MessageQueue, MessageQue创建时会调用nativeInit()

nativeInit()会创建NativeMessageQueue

NativeMessageQueue创建时，会确保一个Looper在TLS中(native的Looper也是线程唯一)

Looper创建时,先重置唤醒文件描述符mWakeEventFd(eventFd()),再执行rebuildEpollLocked方法,此方法

* 重置epoll文件描述符mEpollFd(epoll_create)
* 将mWakeEvendFd添加至mEpollFd的监听(epoll_ctl)
* 也会将一些其他fd加入监听



### 1.2.2 阻塞阶段

java层MessageQueue#next()会调用nativePollOnce()进入阻塞等待

nativePollOnce->NativeMessageQueue#pollOnce->Looper#pollOnce->pollInner

pollInner调用epoll_wait进入阻塞等待，等待管道读端可读



### 1.2.3 唤醒阶段

java#nativeWake->NativeMessageQueue->wake()->Looper#wake()

Looper#wake()通过write方法向mWakeEventFd写入内容“1”

管道中有内容可读，epoll_wait会结束阻塞,Looper#PollInner方法得以继续

如果是mWakeEventFd,通过awoken方法从管道中将数据读出清空

如果当前native消息队列不为空，则轮询执行所有native消息

如果reponse不为空，则轮询执行所有reponse

之后java线程得以继续



### 1.2.4 native消息的来源

Surfaceflinger/Audio/Choreographer/InputDispatcher...

均有对Looper#sendMessage的调用





# 二 Binder

<img src="binder.png" alt="binder" style="zoom:50%;" />



## 2.1 角色分析

### 2.1.1 IInterface

aidl自动生成的java文件会继承该接口, ```asBinder()```方法用来返回实际干活的binder, 对于stub来说返回自己，对于proxy返回mRemote(即stub)

### 2.1.2 IBinder

用于跨进程通信的接口, ```transact```方法用于执行具体的跨进程通信，由```Binder```类来具体实现

```transact()```方法是同步的,客户端进程调用之后,对应的线程会阻塞. 服务端进程接收之后,会在线程池中选择线程执行

```transact()```方法会在一个线程中执行，支持递归跨进程调用，即A.transact()调用B.transact(), B.transact()又调用A.transact()。分析```transact()```运行的线程时，完全不用考虑跨进程因素，当做本地对象考虑即可

### 2.1.3 Binder

```IBinder```的实现类，实现了```transact()```方法，同时新增了```onTransact()```方法, ```transact()```会将请求全部转调到```onTransact()```中。

### 2.1.4 IWindowManger

根据aidl文件自动生成的接口，继承自```IInterface```, aidl文件中定义的方法会被添加到该接口中。同时该接口包含一个```Stub```抽象内部类作为服务段角色,```Stub```又有一个```Proxy```内部类作为服务代理。

### 2.1.5 Stub

服务端角色，抽象类，其抽象方法也就是aidl中定义的方法。继承```Stub```并实现了抽象方法的类，才是真正的服务端。

```Stub```类有一个静态方法```asInterface(IBinder)```，用于返回一个具体服务端对象。当本进程调用时，返回```Stub```本身，当跨进程调用时,返回一个```Proxy```,参数IBinder是真正的服务端

在```Stub```的```onTransact()```方法中，根据```code```参数，转调Stub中对外提供服务的方法(aidl中的方法)。而code参数，是在使用```IWindowManager```调用时通过方法名转换而来的(见```Proxy```)，从而为客户端的调用和服务端的具体执行建立了联系

### 2.1.6 Proxy

服务端代理,此类存在于客户端进程，客户端调用该类的方法，该类最终调用Stub。Stub类的内部类，非抽象。对aidl中定义的方法做了默认实现，通过方法名找到对应的code，并把参数转化为Parcel,然后调用```mRemote```的```transact()```方法发起跨进程通信。```mRemote```是一个```IBinder```对象，也就是上面```Stub```类的对象,```Stub```类是抽象的，这个```mRemote```的真是身份是```WindowManagerService```

### 2.1.7 WindowManagerService

真正的服务端。在跨进程通信时,该类作为```IBinder``` 对象,经```Proxy```的构造方法，保存在```mRemote```参数中。当```Stub```中的```onTransact()```响应一条跨进程调用时，会根据```code```参数转调对应的抽象方法(```aidl```中的方法),而```WindowManagerService```则继承了```Stub```并实现了抽象方法，因此最终会执行```WindowManagerService```中的方法产生服务

### 2.1.8 Parcel

Android中用于序列化/反序列化的类，常用于```Binder```通信。







# 三 Apk打包

## 3.1 打包流程

* 生成R文件

  输入: 各资源文件

  工具: aapt (Android Asset Packaing Tool)

  输出: R.java, resource.arsc

  资源文件会分配 id，放在resource.arsc文件中。该文件记录了id与对应xml文件的对应关系

* aidl生成.java

  输入: 各aidl文件

  工具: aidl

  中间会生成对应的.stub/.proxy类。如果没有用到aidl，则跳过此步骤

* 生成class文件

  输入: 各java文件，包括前两步生成的java

  工具: javac

  标准的javac编译器动作，生成.class文件,位于工程bin/classes目录

* 生成dex文件

  输入: 各class文件

  工具: dx

  三方lib中的class文件也会在这一步被转换成dex

  dex文件可以压缩常量池，消除冗余信息，减少最终apk大小

* 生成apk文件

  输入: 位图、asset目录、dex文件

  工具: apkbuilder

  此时apk尚未签名

* 签名

  输入: 生肉apk

  工具: jarsigner、apksigner

* 对齐

  输入: 已签名apk

  工具: zipalign

  以4的倍数进行对齐，加快访问速度

## 3.2 签名

### 3.2.1 v1签名

支持版本: Android6.0及以前

签名工具: `jarsigner`

签名文件: META-INF目录，三个文件: MANIFEST.MF, CERT.SF, CERT.RSA

MANIFEST.MF： 对所有文件生成摘要(SHA1)

CERT.SF： 对MANIFEST.MF及文件中的内容进行二次摘要(SHA1)

CERT.RSA: 使用私钥对CERT.SF进行签名, 签名+公钥(数字证书)放入CERT.RSA中

验证: 使用公钥对签名进行解密，得到摘要，根据摘要依次比对各文件是否经过修改

### 3.2.2 v2签名

<img src="sign-v2.png" alt="map" />

支持版本: Android7.0及以上

在apk(zip)文件中新增了`apk签名块`数据区域，用于存放签名信息

### 3.2.3 v3签名

支持版本: Android9.0及以上

支持apk秘钥轮替。及可以更新签名。但是要使用旧的签名为新签名担保



# 四 dex文件

## 4.1 dex文件结构

## 4.2 multidex

### 4.2.1 multidex的原因

### 4.2.2 不分dex直接加载会不会有问题



# 五 View

## 5.1 measure layout & draw

## 5.2 事件分发

### MotionEvent

事件类型:

* ACTION_DOWN: 第一根手指按下
* ACTION_POINTER_DOWN: 多指按下
* ACTION_MOVE: 滑动
* ACTION_POINTER_UP: 多指中的一指抬起
* ACTION_UP: 所有手指抬起

getAction()

​	getAction()返回一个int值action,其中低16位表示事件类型,高16位表示事件索引

​	如果有多指按下的情况，需要通过事件索引查询对应的PointerID，表明当前是第几根手指

​	只有DOWN/UP事件带有PointerID, MOVE事件没有PointerID

​	当一个View要消费某个DOWN事件时，该事件PointerID对应的后续MOVE和UP事件也全部交给该View处理

​	当一个View不消费某个DOWN事件时，后续的MOVE/UP事件不会传递给该View

### TouchTarget

​	将一个View及其关心的PointerID绑定在一起，一个View可以对应多个PointerID

​	PointerID存放在一个int数据(pointerIdBits)中，按位存放

### 事件拦截

​	DOWN事件或者已经有TouchTarget,则需要通过onInterceptTouchEvent决定是否需要拦截

​	其他情况则认为一定需要拦截(没有TouchTarget说明所有子view都不关系事件,只能自己消费，也就是拦截)

​	子view可以requestDisallowInterupt要求不拦截事件,常用于解决滑动冲突

### 事件分发

​	依次遍历所有子view，如果事件发生在view的范围内，则认为是潜在消费者

​	如果潜在消费者消费了多指事件第一个DOWN，则默认它也会消费后续的DOWN

​	向潜在消费者分发事件,调用其dispatch方法，返回成功则将其加入TouchTarget链表头

​	如果所有潜在消费者均不感兴趣，最终自己消费

### 事件拆分

​	如果设置了事件拆分，则把事件按pointerID拆分病分发给绑定该ID的view

​	如果view收到的第一个事件是POINTER_DOWN，则将其修改为DOWN	

### 事件消费 (View#dispatchTouchEvent)

​	如果有TouchListener则执行,并以listener返回值作为最终结果

​	没有TouchListener则执行onTouchEvent,以onTouchEvent为最终结果

​	onTouchEvent主要用来触发ClickListener、LongClickListener

​	如果clickable=true，则会触发listener，并最终返回true。否则返回false

## 5.3 requestLayout 

## 5.4 invalidate



# 六 Context

## 6.1 Context的种类

<img src="ContextType.png" alt="binder" style="zoom:50%;" />





# 七 Activity

## 7.1 启动流程

Activity#startActivity包装为startActivityForResult,进入Instrument#execStartActivity
Instrument获取到AMS代理，通过binder调用startActivity方法
AMS构建ActivityStarter对象，执行其execute方法进入startActivityMayWait
ActivtiyStarter根据Intent#flag构建一个TaskRecord和ActivityRecord并绑定，之后调用ActivityStackSupervisor#resumeFocusedStackTopActivityLocked方法
ASS查找当前站定的activity并进入ActivityStack的resumeTopActivityUnchecked方法
ActivityStack先对栈顶Activity执行startPausing,触发其onPause
然后执行ActivityStackSupervisor#startSpecificActivityLocked方法
ActivityStackSupervisor检查进程启动情况，如果未启动则创建进行，否则realStartActivityLocked
构建一个ClientTransaction对象，通过LifecycleManager对其进行schedule
ClientTransaction有用一个IApplicationThread，通过binder调用其schedule方法，进入目标app进程
app进程响应transaction的是ClientTransactionHandler，其向H抛出一条消息
消息被响应时，会执行LaunchActivityItem#execute，调用ActivityThread#handleLaunchActivity
之后执行ActivityThread#performLaunch
通过Instrument，以反射的形式，创建Activity对象，将Activity与Application进行attach,然后进入Instrument处理生命周期
依次处理onCreate，onStart，onResume
在handleResumeActivity中会执行Activity#makeVisible，将DecorView添加至Window

# 八 Window



# 九 性能

## 9.1 内存

### 9.1.1 工具 & 原理

### 9.1.2 LeakCanary原理



# 十 BroadCast

## 10.1 静态 & 动态

## 10.2 粘性 & 非粘性

## 10.3 优先级





# 十一 虚拟机

## 11.1 Dalvik

## 11.2 ART





## 十二 屏幕适配



# 十三 gradle

循环依赖会有问题吗，如何解决



# 十四 Service

* 前台Service: 绑定一个常驻通知的Service
* 后台Service: 没有通知





## 14.1 生命周期

## 14.2 startService

## 14.3 bindService





# 十五 ContentProvider



# 十六 序列化

## 16.1 Serializable

## 16.2 Parcelable



# 三方库

## Retrofit

* 技术基础

  动态代理

* 



## LeakCanary

* 技术基础

  弱引用 & 引用队列

* 检测对象

  Activity、Fragment、View、ViewModel

* 原理

  在onDestroy后将Activity的弱引用注册到RefrenceQueue,等待5s如果未被添加到引用队列，说明存在泄漏嫌疑

  手动执行GC，如果依旧没添加到引用队列,判定为存在泄漏

# Android版本更新

## Android 12

### 所有应用

* 优化沉浸模式下手势导航

* 前台服务通知延迟: 延迟10s再展示前台服务通知
* 限制mac地址访问. target31返回null,其他返回占位符

### target=31

* 后台应用禁止启动前台service
* 无法在service、广播中startActivity
* 使用IntentFilter的Activity/Service/BroadcastReceiver必须显示指明export

## Android 11

### target=30

* 强制分区存储
* 自动重置权限,长时间未使用的应用重置敏感权限
* 必须在系统设置页面,才能授权后台位置访问权限
* 软件包可见性。从获取安装列表，改为提供列表向系统查询哪些存在
* 无法安装仅使用v1签名的应用

### 所有应用

* 单次授权。位置/麦克风/摄像头可以临时设置权限
* 用户拒绝某项权限增加”不在询问“选项
* 针对5G的模拟器

## Android 10

### target=29

* 分区存储 （非强制）

  * 非强制, 在Manifest中声明```requestLegacyExternalStorage```可规避

  * 内部存储空间

    本应用专属```getFilesDir```

    ```/data/data/package_name/```

    ```/data/user/0/package_name/```

    其他应用无论如何不能访问

  * 外部存储空间

    本应用专属空间 ```getExernalFilesDir```

    其他应用需要权限才能访问

    ```/storage/emulated/0/Android/data/package_name/```

* 后台启动activity限制

* 不能获取序列号 & IMEI

* WLAN/BT扫描需要FINE_LOCATION权限

### 所有应用

* 限制非sdk接口使用 (反射)

  * 黑名单

    禁止访问，访问则抛出异常

  * 灰名单

    暂时还可以用

  * 白名单

    公开api,可正常使用



# 调试工具

## Android Size Analyzer





# 性能调优

MemoryLeak原理



65536个方法的原因





# Messenger



# Parcelable VS Serialized

