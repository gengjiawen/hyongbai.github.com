---
layout: post
title: "Android(Java)中关于内存管理的知识总结"
category: all-about-tech
tags: -[Android] -[Java] -[Memory]
date: 2017-04-21 10:41:00+00:00
---

一直想这里这一块知识，但是一直没有时间整理。最近在回顾这方面的逻辑，所以就基于本人目前阶段的理解，整理了一下，记录一个总结。

## 内存区域

### 堆Heap
- 线程共享
- 虚拟机启动时创建。
- 最大的一块内存占用模块。用来存储所有的对象实例。

### 方法区MethodArea
- 线程共享
- 存储类信息、常量、静态变量等的数据都是存储于此。

### VMStack
- 线程独有
- Java虚拟机栈。
- 每一个方法的调用都是一次入栈和出栈的过程。
- 通常所说的栈内存就是它。用来存储局部变量表等。

### NativeMethodStack
- 与虚拟机栈类似。主要用来处理native方法。
- SunHotSpot直接将其与VM Stack合并了。

### ProgramCounterRegister
- 线程独有
- 字节码执行时的行号记录器。
- 跳转/异常等都依赖于此来确立位置。

## 对象访问

Java中关于对象访问大概有如下两种方式：

### 句柄
- HEAP会划分出句柄池/对象实例池/对象类型池
- Reference里面保存的是句柄地址。找到句柄。
- 句柄中包含了实例地址以及类型地址。

###  指针
- Reference里面保存的是实例地址。
- 实例地址去Heap中获取实例数据。
- 实例从MethodArea中获取实例类型。

## 内存释放

有如下的GC算法：

### 引用计数
- 会标记每个变量被应用的次数，为0的时候将其释放
- 会导致一个问题：A和B相互应用，但是已经不被其他地方引用的时候会出现不能被释放的死循环

### 标记清除法(Mark-Sweep)
- 这里会有一个GC-Root的概念，当某个实例不能达到gc-root的时候会被标记为dead
- 标记成功之后会将所有dead的实例给清理掉
- 这样的做法会导致很多的内存碎片

###  标记清除整理法(Mark-Sweep-Compat)
- 基于标记清除法
- 标记清楚之后会将内对对象进行移动减少碎片
- 但是整理的过程中需要中断线程，故而不好。

### 拷贝回收法
- 也是标记法的一种
- 会有两块内存，分别为AB。当A满的时候，回收并将移到B中。然后新的会在B分配。B满的时候回收并移到A。如此循环。
- 此种效率高很多，但是比较吃内存。而且还有可能会引起中断。

### 逐代回收法
- 基于拷贝回收法
- 不同的是，它分为新(Young)老(Tenured)内存区域。老的移动到老区域，新区域负责新的对象。gc的时候会将新区域剩下的复制到老区域。
- 当老区域满了之后再对老区域进行回收。
- 如果出现老区域引用新区域时，会用Card Table标记新区域中，标记为dirty。当GC的时候，会把新区域里面的dirty的内存复制到老区域。从而避免出现老区域出现引用异常的问题。
- 出现老的引用新的这种方式概率非常小，1%可能不到。所以用时间换空间也是可以接受的。

## Android内存回收算法

- 基于逐代回收法
- 会有Young/Old/Permanent三个区域。Young被回收的时候，存活的移动到Old。当Old满了的时候才回收Old中的，存活的移动到Permanent。当Permanent满了的时候，老的会被回收掉。
- Young/Old/Permanent的GC执行时间是越来越慢的。
- Logcat中关于GC的原因的
    - [GC原因][释放内存总量],[可用百分比][已使用的内存空间],[申请Java层总内存空间],[线程中断时间][GC消耗时间]
    - GC_MALLOC, 内存分配失败时触发
    - GC_CONCURRENT，当分配的对象大小超过384K时触发
    - GC_EXPLICIT，对垃圾收集的显式调用(System.gc)
    - GC_EXTERNAL_ALLOC，外部内存分配失败时触发