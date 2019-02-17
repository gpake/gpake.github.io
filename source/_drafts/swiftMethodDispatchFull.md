---
title: swiftMethodDispatch
date: 2019-02-18 18:28:22
tags:

---

# Swift 是怎么调用方法的

经过多年的发展和前辈们的科普，我们都对 Objective-C 的 Method Lookup 机制了然于胸。
但是 Swift 是怎么进行方法派发 ( dispatch ) 的，倒是一个不太被常问到的一个问题。
不妨一起来看一看。

<!-- more -->

## Dispatch 是什么

> Dispatch 派发，指的是**语言底层**找到用户想要调用的方法，并执行调用过程的动作。
> Call 调用，指的是语言在**高级层面**，指示一个函数进行相关命令的行为。

对于一个编译型语言来说，一般有三种方式可以派发到方法：静态派发，基于 Table 的派发，消息派发。

Java 默认是使用 Table 方式派发的，你可以使用 final 关键字来强制动态派发。

C++ 默认是静态派发的，你可以使用 virtual 关键字来启用 Table 派发。

Objective-C 全都基于消息派发，不过也允许你使用 C level 的函数直接派发。

Swift 则巧妙的使用了这三种方法，分别应对不同的情况。



### Dispatch 的种类

> 派发的方法虽然不同，但主要是性能和灵活性的不同妥协。

#### Direct Dispatch 直接派发

直接派发（静态派发）最快。不只是因为他的汇编命令少，还因为他可以应用很多编译器黑魔法，比如 inline 优化等。

不过这种方式局限性最大，因为不够动态而无法支持子类。



#### Table 派发

基于 Table 的派发机制是编译语言最常用的方式，Table 一般是用函数地址的数组来存储每个类的声明。大多数语言包括 Swift 都把这个称作 VTable，不过 Swift 中还有一个术语叫做 Witness Table 是服务于 Protocol 的，下面会提到。

每个子类都会有自己的 VTable 副本，子类中 override 的方法指针也会被替换成新的，子类新添加的方法则会被添加在 Table 的尾部。程序会在运行时确定每个函数具体的地址。

表的查找就实现而言非常简单，而且性能可以预测，不过还是比直接派发更慢一些。从字节码的角度来说，这种方法多了两步查找和一部跳转，这些都是开销。而且，这种方法没法使用一些黑魔法来优化。



另一个不好的点在于，这种基于数组的派发让 extension 没法扩展这个 table。因为在子类添加方法列表到尾部后，extension 就没有一个安全的 index 可以添加他的方法到 table 中。这里介绍了更多的细节<https://lists.swift.org/pipermail/swift-evolution/Week-of-Mon-20151207/001922.html>



#### Message 派发

Message 派发是最灵活的派发方式。他是 Cocoa 的基石，也是 KVO，UIAppearance，Core Data 的核心。

他的关键功能是可以让开发者在运行时改变消息发送的行为。不仅可以通过 swizzling 修改 method，还可以通过 isa-swizzling 来修改对象。

一旦有消息发出，runtime 就会基于继承关系开始查找，虽然听起来很慢，但是有 cache 做保障，一旦 cache 经过预热，就和 Table 方式差不多快了。

[This blog post](http://www.friday.com/bbum/2009/12/18/objc_msgsend-part-1-the-road-map/) 有一些更深入的说明。



## Swift Dispatch 的方式

Swift 用什么派发方式？可以从下面四个方面来说：

1. 声明位置
2. 引用类型
3. 关键字的影响
4. 可见性的影响

在往下说之前，先说一个前提：swift 从来没说过他具体是怎么做派发的。唯一可以确定的是，用 `dynamic` 的时候，会通过 OC 的 runtime 来进行消息派发。



#### 声明所在位置的影响

我们可以在 extension 中声明一个方法，也可以在原始对象中声明。他们是有区别的

extension 中会直接使用静态派发

|               | 原始声明      | extension |
| ------------- | ------------- | --------- |
| Value Type    | Static        | Static    |
| Protocol      | Witness Table | Static    |
| Class         | V-Table       | Static    |
| NSObject 子类 | V-Table       | Message   |



#### 引用类型的影响 

```swift
protocol MyProtocol { 
} 

struct MyStruct: MyProtocol {
}

extension MyStruct {
    func extensionMethod() {
        print("In Struct")
    }
}

extension MyProtocol {
    func extensionMethod() {
        print("In Protocol")
    }
}

let myStruct = MyStruct()
let proto: MyProtocol = myStruct

myStruct.extensionMethod() // -> “In Struct”
proto.extensionMethod() // -> “In Protocol”
```

调用 proto.extensionMethod() 不会走 struct 的实现，只有对 protocol 可见的方法才会被调用。

如果 extensionMethod() 的声明放在 protocol 里，那 proto.extensionMethod() 就会走 struct 的实现了



从 SIL 的角度来说，如果不写在 protocol 声明中，witness table 无法把协议和类的函数产生映射关系，所以他无法 witness 实现协议的对象，只能按照默认实现进行直接派发执行了。



#### 关键字的影响

##### final

使用了 final 的，都用静态派发，因为 final 意味着完全没有动态性。final 用于类型，或是 function，都会造成这样的情况。

而且 final 作用范围的方法，都不会被 export 到 OC runtime，也不会生成 selector



##### dynamic

使用了 dynamic 的 class （只有 class 可以 dynamic），会开启 message 模式，让 OC runtime 可以调用。

必须 @import Foundation，必须有 @objc，如果是 class，还必须是 NSObject 的子类

延展阅读： dynamic vs @objc: dynamic 是强制使用 message 派发，KVO 需要。@objc 只是 export 给 objc，swift 还是静态派发



##### @objc / @nonobjc 

@objc / @nonobjc 控制方法对于 objc 的可见性。但是不会改变 swift 中的函数如何被派发。 

> @nonobjc 可以禁止函数使用 message 派发 （和 final 在汇编上看起来类似，偏向于使用 final)
> // 并不会这样，@nonobjc 依然是使用 Table 调用，但是 @nonobjc 之后无法使用 dynamic，会提示 `error: a declaration cannot be both '@nonobjc' and 'dynamic'`

@objc 的原理是生成两个函数引用，一个给 swift 调用，一个给 objc 调用 


@objc final func aFunc() {} 会让消息使用直接派发，不过依然会 export 给 objc， 



##### @inline

@inline 可以告诉编译器去优化直接派发的性能。 

see: <https://github.com/vandadnp/swift-weekly/blob/master/issue11/README.md#inline> 

@inline 可以给生成的函数加上标记，后期编译器进行指定的优化 

inline 可以选择的参数有两个  `never` 和 `__always` 

对于 inline 的使用建议： 

**0，默认行为是编译器自己决定要不要使用 inline 进行优化，你也应该保持默认行为。**

1，如果你的函数特别长，你不想让你的打包体积变得特别大，可以使用 @inline(never) 

2，如果你的函数很小，你希望他可以快一点，就可以使用 @inline(__always)，不过其实编译器也会帮你做这样的事情，所以你这么做也基本上不会让他变得更快 

> 有趣的是，如果你用 `dynamic @inline(__always) func dynamicOrDirect() {}` 依然会得到一个 message 派发的函数。
>
> 不过，这个没什么意义，应该是未定义的行为，忽略就好。



##### 总结

|         | class                                                  | Value Type        | Protocol          | extension           | func                                      | 备注        |
| ------- | ------------------------------------------------------ | ----------------- | ----------------- | ------------------- | ----------------------------------------- | ----------- |
| final   | Static                                                 | Static            | Static            | Static              | Static                                    | @objc final |
| dynamic | only class member                                      | only class member | only class member | 只有 class 的才可以 | Message 必须 @import Foundation必须 @objc |             |
| inline  | 根据属性决定直接派发的编译器优化行为，不影响派发原理。 |                   |                   |                     |                                           |             |



### 方法可见性的影响

Swift 编译器会尽可能的帮你优化派发，比如：你的方法没有被继承，那他就会注意到这个，并用尝试使用直接派发来优化性能。

虽然大多数时候这种机制很有效，不过偶尔也会有些问题： 

#### KVO

值得注意的是 KVO，被观察的属性也必须被声明为 `dynamic`，否则 setter 会走直接派发，无法触发变化。



> https://developer.apple.com/swift/blog/?id=27> 这篇文章里有更多关于优化的细节。



## 关于派发方式的总结： 

| 对象\派发方式     | StaticStatic        | VTableVTable | Witness TableWitness Table | MessageMessage           |
| ----------------- | ------------------- | ------------ | -------------------------- | ------------------------ |
| NSObject subclass | final<br>extension  | default      | : protocol                 | dynamic                  |
| Swift Class       | final<br/>extension | default      | : protocol                 | dynamic                  |
| Protocol          | extension           | default      | N/A                        | : NSObjectProtocol @objc |
| Value Type        | default             | N/A          | : protocol                 | N/A                      |



## 性能建议； 

使用 `final`， `private` ，Whole Module Optimization 

`final` 可以标记在一个 class, 属性或方法之前，表示这个对象无法被继承 / 覆盖。编译器会因此把关于这个对象的派发改变为直接派发。

`private` 可以标记一个对象的可见性，因为都在同一个文件内，所以编译器可以确定这个对象的继承情况，从而推断出你的方法可以被标记为 `final`。如果没有继承，

默认的方法可见性是 `internal`，这就代表着，如果你启用 `Whole Module Optimization` ，编译器就可以知道你的  `internal` 方法是否有被继承，如果没有的话，他们也会被优化为直接派发。
