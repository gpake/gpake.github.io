---
title: Swift 底层是怎么调用方法的
date: 2019-02-11 22:34:27
tags: swift dispatch
---

经过多年的发展和前辈们的科普，我们都对 Objective-C 的 Method Lookup 机制了然于胸。
但是 Swift 是怎么进行方法派发 ( dispatch ) 的，倒是一个不太被常问到的一个问题。

## TL;DR

| 对象\派发方式 | Static                           | VTable  | Witness Table | Message                      |
| ------------- | -------------------------------- | ------- | ------------- | ---------------------------- |
| NSObject      | @nonobjc<br/>final<br/>extension | default | : protocol    | dynamic                      |
| Swift Class   | extension<br/>final              | default | : protocol    | dynamic                      |
| Protocol      | extension                        | default | N/A           | : NSObjectProtocol<br/>@objc |
| Value Type    | 默认行为                         | N/A     | : protocol    | N/A                          |

表格中间为可以产生此影响的关键字。

<!-- more -->

## 具体点，结果是什么？

### 先总结

#### 派发一共有三种

* 静态派发
* Table 派发
* Message 派发

#### 主要规律如下

* 派发效率从高到低为： `静态派发` > `Table 派发` > `Message 派发`
* 对于值类型，总是静态派发
* extension 是静态派发，可以作用于  `protocol` 和 `class`
* `NSObject` 针对原生声明中的方法使用 Table 派发
* Protocol 中默认的方法实现都用

### 后解释

#### Value Type

因为 struct 这样的值类型不支持继承，所以无需动态，它所有的方法调用（包括协议方法），都是静态调用。

// 协议方法貌似一定走的 Witness 吧，存疑

虽然不支持继承，但 struct 能通过 Protocol 实现多态

#### Class

对于一个 pure Swift class 来说，影响它方法调用的关键字只有 final

函数如果被标记成 final ，编译器就会知道这个方法不会被 override，并把它的调用方式标记成静态调用。而对于未标记成 final 并在 class 内部（非 extension）中定义的方法，Swift 会用一种叫作 Virtual Table 的机制来查找这个方法并调用

因为定义在 extension 中的方法目前还不支持 override，所以定义在其中的方法都是静态派发的。

#### NSObject Subclass

影响这种类型的函数调用方式的关键字有很多

标记为 final 的函数是一定会静态调用的，原因同 class。

主类（非 extension）中定义的普通方法和标记为 @objc 的方法都使用 V-Table 机制派发。用 Swift 编写的类是不能被 Objective-C 继承的，@objc 只是把方法暴露给 Objective-C，并没有改变方法派发的本质

dynamic 的方法不管在主类还是 extension 中都是通过发消息动态调用的，因为 dynamic 就是干这个事儿的。

Extension 中的方法是无法基于 V-Table 派发的，被标记为 @objc 和 dynamic 的又无法使用静态派发，所以只能基于 message 派发。

以上介绍的都是在**没有编译器优化**的情况下方法的派发方式。在有优化的情况下，编译器会尽可能地把基于 Table 机制派发的方法变成静态派发，有的方法甚至会就地展开，变成 inline 的形式，一切为了效率嘛

### Protocol 对象调用的情况

用 Struct、Class 和 NSObject Subclass 分别实现了同一个协议，用本身的对象和协议对象调用协议中的方法。通过观察编译后的 SIL，我们可以得出以下结论，



用类本身的对象调用协议方法的时候，像我们上面发现的一样，该怎么派发还是怎么派发，跟正常的方法调用没有区别；但是当用协议对象调用协议方法的时，不管是结构体还是类，所有的方法都是使用一种基于 Witness Table 的形式派发

定义为 @objc 的协议方法会基于 message 派发的





静态调用

直接在调用的地方拿到函数的引用，然后直接调用。

这种方法效率最高，系统也会优先以这种方法进行调用。



VTable

注意，vtable 声明的引用是对于 class 可见的最小派生方法 ( least-derived method)。在上面的例子来说，B 的 vtable 引用 A.bar 而不是 b.bar，C 的 vtable 引用 A.bas 而不是 C.bas

Swift 的 AST 维护了声明之间的 override 关系，可以用来查找派生类 SIL vtable 中处于 override 状态的方法，比如 C 的 vtable 中的方法 C.bas

  

如果 SIL 函数是一个整体 ( thunk )，这个函数名会被处理成原函数实现的链接 (linkage)



Witness Table

Witness Table 是用于 protocol 的。

SIL 把需要动态派发的一般类型（ generic types）都编并存在 witness table 中。这些信息会在生成 binary code 的时候被用于产生 runtime 的 dispatch 表。他也可以被用于 SIL 针对特定的一般函数（specialize generic functions）优化。一张 witness 表可以准确的描述每一个声明。一般类型（generic types）的所有实例对象都共用一个 generic witness 表。派生类会从他们的基类继承 witness 表。



Witness 表依赖 protocol 的遵守情况，用唯一标识来表明一个类（concrete）类型的 protocol 遵守情况。

- A *normal protocol conformance* names a (potentially unbound generic) type, the protocol it conforms to, and the module in which the type or extension declaration that provides the conformance appears. These correspond 1:1 to protocol conformance declarations in the source code.
- If a derived class conforms to a protocol through inheritance from its base class, this is represented by an *inherited protocol conformance*, which simply references the protocol conformance for the base class.
- If an instance of a generic type conforms to a protocol, it does so with a *specialized conformance*, which provides the generic parameter bindings to the normal conformance, which should be for a generic type.

Witness tables are only directly associated with normal conformances. Inherited and specialized conformances indirectly reference the witness table of the underlying normal conformance.



Witness 表由下面这些组成:

- *Base protocol entries* provide references to the protocol conformances that satisfy the witnessed protocols' inherited protocols.
- *Method entries* map a method requirement of the protocol to a SIL function that implements that method for the witness type. One method entry must exist for every required method of the witnessed protocol.
- *Associated type entries* map an associated type requirement of the protocol to the type that satisfies that requirement for the witness type. Note that the witness type is a source-level Swift type and not a SIL type. One associated type entry must exist for every required associated type of the witnessed protocol.
- *Associated type protocol entries* map a protocol requirement on an associated type to the protocol conformance that satisfies that requirement for the associated type.