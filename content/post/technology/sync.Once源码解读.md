---
title: "sync.Once源码解读"
date: 2020-08-03T17:48:14+08:00
draft: true
description: "sync.Once源码解读"
keywords: ["Go","sync.Once","单例模式",]
tags: ["sync.Once","设计模式","Go","源码"]
categories: ["技术"]
---
最近开始看设计模式相关的内容，同时也在寻找 **Golang** 相关的技术岗位，就想如何用 **Go** 语言实现呢？结合自己之前常用的技术栈 **.NET**，实现了粗糙版的单例模式，本着学习的目的，用**Google**搜到了比较好的实现方式 **sync.Once**。照着例子敲了敲，运行结果预期一致。出于对**sync.Once**实现的好奇，于是乎就研究了下其实现源码。
<!--more-->
## 单例模式

***

我们先看下如何用sync.Once实现单例模式，代码如下

```Golang
package main

import (
    "fmt"
    "sync"
)

type Shape struct {
    width float32
}

func (s *Shape) print() {
    fmt.Println(s.width)
}

var instance *Shape

func createInstance() {
    instance = &Shape{width:30}
}

func main() {
    var once sync.Once
    once.Do(createInstance)
    instance.print()
}
```

单例模式的实现是不是很简单，你或许会有些疑惑 **sync.Once** 的 **Do** 方法是怎么实现仅执行一次逻辑的？我也是，那下面我们看看它的源码吧。

## sync.Once源码

***

```Golang
package sync

import (
    "sync/atomic"
)

// Once is an object that will perform exactly one action.
type Once struct {
    // done indicates whether the action has been performed.
    // It is first in the struct because it is used in the hot path.
    // The hot path is inlined at every call site.
    // Placing done first allows more compact instructions on some architectures (amd64/x86),
    // and fewer instructions (to calculate offset) on other architectures.
    done uint32
    m    Mutex
}

// Do calls the function f if and only if Do is being called for the
// first time for this instance of Once. In other words, given
// var once Once
// if once.Do(f) is called multiple times, only the first call will invoke f,
// even if f has a different value in each invocation. A new instance of
// Once is required for each function to execute.
//
// Do is intended for initialization that must be run exactly once. Since f
// is niladic, it may be necessary to use a function literal to capture the
// arguments to a function to be invoked by Do:
// config.once.Do(func() { config.init(filename) })
//
// Because no call to Do returns until the one call to f returns, if f causes
// Do to be called, it will deadlock.
//
// If f panics, Do considers it to have returned; future calls of Do return
// without calling f.
//
func (o *Once) Do(f func()) {
    // Note: Here is an incorrect implementation of Do:
    //
    // if atomic.CompareAndSwapUint32(&o.done, 0, 1) {
        //     f()
        // }
    //
    // Do guarantees that when it returns, f has finished.
    // This implementation would not implement that guarantee:
    // given two simultaneous calls, the winner of the cas would
    // call f, and the second would return immediately, without
    // waiting for the first's call to f to complete.
    // This is why the slow path falls back to a mutex, and why
    // the atomic.StoreUint32 must be delayed until after f returns.

    if atomic.LoadUint32(&o.done) == 0 {
        // Outlined slow-path to allow inlining of the fast-path.
        o.doSlow(f)
    }
}

func (o *Once) doSlow(f func()) {
    o.m.Lock()
    defer o.m.Unlock()
    if o.done == 0 {
        defer atomic.StoreUint32(&o.done, 1)
        f()
    }
}
```

有没有发现源码很少（注释比代码还多），看似很简单，但我认为里面还是有几点需要注意的。

### 为什么使用 **atomic.LoadUint32(&o.done)==0** 而不是直接用 **Mutex** 呢

在回答问题前，我们先应该明确 **sync.Do** 的使用场景：多个 **goroutines** 竞争初始化同一个资源，为了序列化**goroutines**的访问顺序。好，既然我们确定了是多个 **goroutines**这个场景，为了保证任务或资源仅执行一次我们必须使用锁机制，使用 **Mutex** 肯定能解决问题，但会有个性能问题，不管资源初始化是否已经完成，每次执行时都要获取释放锁，这个消耗再高频访问时肯定会出现性能问题。既然问题出现了，怎么解决？可以考虑增加一个标识变量 **done**，即 **done** 默认0，若任务已执行则赋值为1。这样就可以提升了性能。但这里又出现了问题，可能会出现多个**goroutines**访问**done**，如果还使用**Mutex**进行限制，性能的似乎并没有提升太多，所以这里可以改用原子操作 **atomic** 来解决。因为**atomic**是更底层硬件上的操作，其相关的CPU指令在执行时是不允许中断的，而**Mutex**是系统实现的（注：这是根据搜索的资料，我自己理解的，水平有限，如果您有更好的资料，可以发来让我学习进步下，谢谢！！！）。到这儿，第一个问题基本就回答完了，下面我们看一起看第二个问题吧！

### 为什么在 **doSlow** 方法中使用 **o.done==0**呢

这个问题其实相对简单些，因为**done**已经处于**Mutex**锁的操作范围内，即保证了对它的可观测，而且由于这里是读操作，所以在其它**goroutines**对**done**的访问也是可观测的。

### 既然已经使用 **Mutex** 锁定了资源块，为什么不能直接使用 **o.done=1** 赋值呢

在这个问题中，你会觉得很奇怪，既然已经对资源进行了锁操作，那这里赋值为何又使用**atomic**呢，是不是很奇怪？那现在我们回忆下**done**第一次访问时在哪里？是在**Do**方法里，那里我们使用的是**atomic**去访问的，假设我们在当前使用**o.done=1**赋值，会出现什么意外？由于这里没使用原子操作赋值，所以在其它**goroutines**调用**Do**方法时，对**done**进行的原子访问，获取可能还是一个旧值，即在内存模型是不可观测的。所以为了保证**done**的可观测性，所以在当前位置使用原子操作对其赋值，保证了其可观测性。

到这里源码基本解读完了，是不是有些收获呢。其实还是一个问题：**done**为什么不是uint8或bool而要使用uint32呢？首先我们看看**atomic**包里有没有**uint8, bool**呢，发现果真没有。难道真的是这个原因么？我们看看**Once**的声明定义。

```Golang
// Once is an object that will perform exactly one action.
type Once struct {
    // done indicates whether the action has been performed.
    // It is first in the struct because it is used in the hot path.
    // The hot path is inlined at every call site.
    // Placing done first allows more compact instructions on some architectures (amd64/x86),
    // and fewer instructions (to calculate offset) on other architectures.
    done uint32
    m    Mutex
}
```

从注释中，我们发现了更有深层次的原因：结构体内存对齐可以提升其性能，让程序跑的更块。

今天的源码解读就到这里了，由于个人水平问题，文章列举的内容会有错误，欢迎指正，谢谢！

## 参考

[sync.Once](https://github.com/golang/go/blob/go1.14.6/src/sync/once.go#L12)

[Go的内存模型](https://golang.org/ref/mem)

[atomic](https://blog.betacat.io/post/golang-atomic-value-exploration/)
