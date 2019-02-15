---
title: Swift 底层是怎么调度方法的
date: 2019-02-11 22:34:27
tags: swift method dispatch
---

经过多年的发展和前辈们的科普，我们都对 Objective-C 的 Method Lookup 机制了然于胸。
但是 Swift 是怎么进行方法派发 ( dispatch ) 的，倒是一个不太被常问到的一个问题。
不妨一起来看一看。

## TL;DR

怎么让你的代码在运行时更快？

- 尽量使用 `值类型`，`final`
- 用 `private` 等缩小访问权限，开启 `Whole Module Optimization `
- 给协议添加默认实现

下表为 Swift 对象的派发方式与关键字的关系。

| 对象\派发方式 | Static              | VTable   | Witness Table | Message                      |
| ------------- | ------------------- | -------- | ------------- | ---------------------------- |
| Value Type    | 默认行为            | N/A      | : protocol    | N/A                          |
| Swift Class   | final<br/>extension | 默认行为 | : protocol    | dynamic                      |
| NSObject      | final<br/>extension | 默认行为 | : protocol    | dynamic                      |
| Protocol      | extension           | 默认行为 | N/A           | : NSObjectProtocol<br/>@objc |

<!-- more -->

## Swift 的派发类别

### 总结

#### 用法：

* 多用值类型
* 多用 `final`
* 少用 `dynamic`

#### 派发一共有三种

* 直接派发
* Table 派发
* Message 派发

#### 主要规律如下

* 派发效率从高到低为： `直接派发` > `Table 派发` > `Message 派发`
* 值类型的方法，总是直接派发
* Class, `NSObject` 子类的方法默认使用 Table 派发
* extension 的方法使用直接派发，可以作用于  `protocol` 和 `class`
* `@objc dynamic` 的方法使用 Objc runtime 进行 Message 派发
* Protocol 中默认的方法实现都用 Table 派发

### 为什么

#### 大前提

在用户无感知的情况下，**越快越好**，所以能直接派发的，就不用 Table

下面我们从常用的数据类型为视角，分为 Type 和 Protocol 详细说明。

#### Value Type

struct, enum 这样的值类型不支持继承，所以无需动态，它所有的方法调用（包括遵守的协议方法），都是**直接调用**。

虽然不支持继承，但值类型通过 Protocol 可以实现扩展。

#### Class Type

对于一个纯 Swift class 来说，默认使用 Table 派发，影响它方法调用的关键字有 `final`、 `dynamic` 和 `extension`。

函数如果被标记成 final ，编译器就会知道这个方法不会被 override，并把它的调用方式标记成直接调用。而对于未标记成 final 并在 class 内部（非 extension）中定义的方法，Swift 会用一种叫作 Virtual Table 的机制来查找这个方法并调用

当一个方法被标记为 `dymanic`，你必须同时把它标记上 `@objc`，此时这个方法会使用 Message 调用，依赖 Objc runtime。

因为定义在 extension 中的方法目前还不支持 override，所以定义在其中的方法都是**直接派发**的。

#### NSObject Subclass

影响这种类型的函数调用方式的关键字有很多

标记为 `final` 和 `dynamic` 的函数可以参考上面的 class。

在原生声明（非 extension）中定义的普通方法和标记为 @objc 的方法都使用 V-Table 机制派发。用 Swift 编写的类是不能被 Objective-C 继承的，@objc 只是把方法暴露给 Objective-C，并没有改变方法派发的本质。

Extension 中的方法是**直接派发**的，但标记为 @objc 的函数需要对 Objc runtime 可见，就变成了 Message 派发。而且加不加 `dynamic` 生成的底层代码是一样的，这里怀疑是编译器隐式的加上了 `dynamic` 关键字。



以上介绍的都是在**没有编译器优化**的情况下方法的派发方式。在有优化的情况下，编译器会尽可能地把基于 Table 机制派发的方法变成直接派发，有的方法甚至会就地展开，变成 inline 的形式，用空间换速度。

### Protocol

Protocol 是一个比较特殊的情况，不同于 Objective-C，Swift 在对待 Protocol 方法调用时更重视实例的类型，而不是实例的内在（比如 Objc 中的 `isa`）。

对于实现了 Protocol 的对象：

- 向对象自己的实例调用，效果和没有 Protocol 的状态一样；

- 把对象类型转换为 Protocol 类型，则会通过 Witness Table 进行派发

### 备注

- @objc / @nonobjc 只影响可见性，`dynamic` 才是真正产生影响的那个

- 协议方法的默认实现被覆盖也会走类型本身的派发方式，在 AST 阶段就确定好了的

### 问题

- 为什么 `extension` 中静态派发为什么无法和负责可见性的 `@objc` 共存？
