# 一 运行时数据区域

程序计数器 虚拟机栈 本地方法栈 堆区 方法区

| 头       | 程序计数器 | 虚拟机栈               | 本地方法栈             | 堆   | 方法区 |
| -------- | ---------- | ---------------------- | ---------------------- | ---- | ------ |
| 线程私有 | Y          | Y                      | Y                      | N    | N      |
| OOM      | N          | Y                      | Y                      | Y    | Y      |
| 异常     |            | StackOverflowError/OOM | StackOverflowError/OOM | OOM  | OOM    |
|          |            |                        |                        |      |        |
|          |            |                        |                        |      |        |

## 1.1 程序计数器



## 1.2 虚拟机栈

### 1.2.1 栈帧

存储数据和部分过程结果的数据结构，也用来处理动态链接、方法返回值和异常分派

* 局部变量表

  长度在编译器决定,保存在方法的code属性中

  long/double需要两个槽,其他原始类型占一个槽, 引用&returnAdress占一个槽

  对于实例方法,第0个局部变量是this.其他显示参数顺序向后

* 操作数栈

  最大深度由编译期决定,保存在方法的code属性中 max_stacks

  每个位置可以保存任意类型的值,包括long & double

  Long & double占据两个单位的深度,其他类型一个

* 运行时常量池的引用

  该引用用来实现动态链接: 将符号引用转换为实际方法的引用

## 1.3 本地方法栈



## 1.4 堆



## 1.5 方法区

## 1.5.1 运行时常量池

class文件中每个类/接口的常量池表

运行时常量池在方法区中分配



# 二 内存分配 & 垃圾回收



# 三 类加载机制

## 3.1 类的生命周期

加载 -> 链接 -> 初始化  -> 调用 -> 卸载

链接: 验证 -> 准备 -> 解析(可选)

## 3.1 加载

根据类名找到其二进制表示(class文件),并构建该类的Class对象

加载由ClassLoader来完成

简单理解:

* 通过类名找到class文件
* 将class文件转化为方法区的数据结构
* 生产对应的Class对象



## 3.2  链接

链接包含三个阶段: 验证 -> 准备 -> 解析(可选)

解析阶段是可选的，所有jvm在规定先后顺序的时候,不涉及解析

在对一个类进行加载、链接、初始化时，会对其父类递归进行



## 3.3 验证

验证class文件的二进制形式是否合规

* 文件格式验证

  魔数、版本号、常量类型、索引

* 元数据验证

  父类是否有、是否继承了final类、是否实现了abstract方法

* 字节码验证

  跳转指令是否跳转至违规的位置(方法体以外、游离的指令位置)

* 符号引用验证

  全限定名能否找到类、字段、方法

## 3.4 准备

为类/接口的static字段分配内存，并赋系统默认值 (非代码里写的初始值)

只处理类变量,不处理实例变量

未执行任何字节码指令,包括 <clinit>, 这两个方法在初始化阶段执行



## 3.5 解析

根据运行时常量池中符号引用来动态决定具体值(直接引用)的过程

符号引用: 用于描述目标的一组符号,以字面量的形式明确定义在class文件中

直接引用: 直接指向目标的指针

以下指令会出发解析: anewarray/checkcast/getfield/getstatic/instanceof/invokedynamic/invokeinterface/invokespecial/invokestatic/invokevirtual/ldc/ldc_w/multianewarray/new/putfiled/putstatic

* 类/接口解析

  将全限定名委托给本类的classloader,加载成功则解析完成

* 字段解析

  先查找本来是否有名称/描述符一致的字段,有则成功,没有则向父类递归

* 类方法解析

  如果发现方法属于接口,则抛出异常

  如果类的某个方法具有相同的名字和描述符,且未设置ACC_ABSTRACT，则成功

  递归向父类查找

* 接口方法解析

  如果方法属于类,则抛出异常

  先找本接口，然后找Object,然后找超接口

  名称/描述符一致且不带ACC_PRIVATE/ACC_STATIC则成功



## 3.6 初始化

* 初始化时机: 

  new/getstatic/putstatic/invokestatic

  使用reflect进行反射

  子类初始化时,先初始化父类

  虚拟机启动时对初始类进行初始化

  使用JDK7的动态语言支持是

  使用JDK8接口默认方法时,实现类初始化之前先初始化接口

* 虚拟机启动 & 初始类

  虚拟机启动通过BootstrapClassloader加载初始类来完成。jvm对初始类进行链接、初始化，并调用main方法

初始化即<clinit>的执行过程。(这里是类的初始化,所以是<clinit>,不是对象的初始化,所以不执行<init>)

<clinit> = 类变量赋值语句 + static{}代码块,按书写顺序 (接口不允许有 static{})

jvm保证子类的<clinit>执行之前先执行父类的<clinit>

接口与超接口的<clinit>执行顺序没有任何关系,应用到那个接口的字段，就执行那个接口的<clinit>

jvm通过`初始化锁LC`保证<clinit>是线程安全的, 每个类/接口的LC是唯一的



## 3.7 Classloader & 双亲委派模型



* 定义类加载器 & 初始类加载器

  定义类加载器: 直接加载该class文件的classloader

  初始加载器: 发出加载请求的加载器



* JVM ClassLoader分类: 

|          | 引导类加载器         | 用户自定义加载器 |
| -------- | -------------------- | ---------------- |
| 英文名   | BootstrapClassloader |                  |
| 提供者   | jvm                  | 用户             |
| 实现方法 | 虚拟机内部实现       | 继承ClassLoader  |

* JAVA 常用ClassLoader

| 表头     | 引导类加载器         | 扩展类加载器         | 应用类加载器           |
| -------- | -------------------- | -------------------- | ---------------------- |
| 英文名   | BootstrapClassLoader | ExtensionClassLoader | ApplicationClassLoader |
| 加载范围 | <JAVA_HOME>\lib      | <JAVA_HOME>\lib\ext  | CLASS_PATH             |

AndroidClassLoader



* 双亲委派模型

首先委托父ClassLoader，父ClassLoader无法加载时,再自己加载

父ClassLoader为null，表示父ClassLoader是BootstrapClassLoader

确保同一个class会被同一个classloader来加载

实现方式是职责链，而不是继承



# 四 内存模型



# 五 class文件

| 类型           | 名称                                                  | 数量                  |
| -------------- | ----------------------------------------------------- | --------------------- |
| u4             | magic                                                 | 1                     |
| u2             | Minor_verison                                         | 1                     |
| U2             | Major_version                                         | 1                     |
| u2             | Constant_pool_count 常量池长度                        | 1                     |
| Cp_info        | Constant_pool[] 常量池                                | Constant_pool_count-1 |
| u2             | Access_flags                                          | 1                     |
| u2             | this_class                                            | 1                     |
| u2             | Super_class                                           | 1                     |
| u2             | interfaces_count 直接超接口的数量                     | 1                     |
| u2             | Interfaces[]                                          | Interfaces_count      |
| u2             | Fields_count 类字段+实例字段的个数                    | 1                     |
| Field_info     | Fields[] 字段表,不含父类                              | Fields_count          |
| u2             | Methods_count 方法个数,不含父类                       | 1                     |
| Method_info    | methods[] 类方法+实例方法+类初始化方法+实例初始化方法 | methods_count         |
| u2             | Attributes_count 属性个数                             | 1                     |
| Attribute_info | attributes[] 属性表                                   | Attributes_count      |



## 5.1 名称描述

| 二进制名称       | 简单名称/非限定名 | 全限定名          | 描述符             |
| ---------------- | ----------------- | ----------------- | ------------------ |
| java.lang.Thread | Thread            | java/lang/Thread; | Ljava/lang/Thread; |
| void start()     | start             | -                 | ()V                |



# 六 指令集

## 6.1 方法调用

* `invokestatic`静态方法
* `invokespecial` <init>,私有方法, 父类方法
* `invokevirtua1`虚方法(包括final方法)
* `invokeinterface`接口方法
* `invokedynamic`

非虚方法: 静态方法、<init>方法、私有方法、父类方法、final方法

虚方法: 非虚方法以外的所有



JIT VS AOT
