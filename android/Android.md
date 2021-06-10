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

异步消息同样要遵守时间约定，没到到搭预定时间时，消息队列会阻塞

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





# 三 Apk打包

## 3.1 打包流程



## 3.2 签名

### 3.2.1 v1签名

### 3.2.2 v2签名

### 3.2.3 v3签名



# 四 dex文件

## 4.1 dex文件结构

## 4.2 multidex

### 4.2.1 multidex的原因

### 4.2.2 不分dex直接加载会不会有问题



# 五 View

## 5.1 measure layout & draw

## 5.2 事件分发

## 5.3 requestLayout 

## 5.4 invalidate



# 六 Context

## 6.1 Context的种类



# 七 Activity

## 7.1 启动流程



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

## 14.1 生命周期

## 14.2 startService

## 14.3 bindService





# 十五 ContentProvider



# 十六 序列化

## 16.1 Serializable

## 16.2 Parcelable



# 三方库

## Retrofit







# 性能调优

MemoryLeak原理



65536个方法的原因





# Messenger



# Parcelable VS Serialized

