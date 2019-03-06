---
title: 试着读一下 SIL
date: 2019-03-06 20:20:13
tags:
---




这是 Swift 摸鱼系列的第二篇

上篇讲的是 [Swift 底层是怎么调度方法的](/2019/02/11/swiftMethodDispatchBrief/)

这篇我们说一下探究 Swift 运行过程中最常用的工具之一：Swift Intermediate Language (SIL)

# Swift Intermediate Language (SIL)

平常我们写 Swift 不用关心编译器怎么工作，了解底层逻辑可以帮你更好的理解语言本身的思想。

SIL 是一个专门为 Swift 定制，为了进行后续优化的语言。

这里我们不过多的介绍文档上的内容，而是关注它在解析中反映出 Swift 的底层逻辑。

<!-- more -->

## 例子

下面那一段例子来简单解释一下 SIL，

其实 SIL 也是一种语言，语法结构清晰，可以将我们使用高级语言被隐藏的很多细节都重新渲染出来，帮助我们理解语言实现的逻辑。

直接上代码：

对于一个简单的 class 声明和调用

```swift
class TestClass {
    func testFunc() {
        print("ahaha")
    }
}

TestClass().testFunc()
```


使用

`swiftc -emit-silgen -Onone test.swift > test.swift.sil`

可以生成 SIL，文件会有大概 200 行。[SIL 文件点这里](https://gist.github.com/gpake/a6028f5b1d0a009ef47a0a185ae65475)

## 例子解析

接下来我们一段一段分析：

### 头部声明


```swift
sil_stage raw

import Builtin
import Swift
import SwiftShims

class TestClass {
  func testFunc()
  init()
  deinit
}
```
sil_stage raw 是 SIL 的一个阶段，如果进行 guaranteed transformations 后，就可以生成 canonical 的 SIL

也可以使用下面的指令生成
`swiftc -emit-sil -Onone test.swift > test.swift.sil`

紧接着就是 import 和声明

### 调用

接下来是重头戏，方法的实现和调用

```swift
// main
sil @main : $@convention(c) (Int32, UnsafeMutablePointer<Optional<UnsafeMutablePointer<Int8>>>) -> Int32 {
bb0(%0 : $Int32, %1 : $UnsafeMutablePointer<Optional<UnsafeMutablePointer<Int8>>>):
  %2 = metatype $@thick TestClass.Type            // user: %4

  // function_ref TestClass.__allocating_init()
  %3 = function_ref @$S4test9TestClassCACycfC : $@convention(method) (@thick TestClass.Type) -> @owned TestClass // user: %4
  %4 = apply %3(%2) : $@convention(method) (@thick TestClass.Type) -> @owned TestClass // users: %7, %6, %5

  %5 = class_method %4 : $TestClass, #TestClass.testFunc!1 : (TestClass) -> () -> (), $@convention(method) (@guaranteed TestClass) -> () // user: %6
  %6 = apply %5(%4) : $@convention(method) (@guaranteed TestClass) -> ()
  destroy_value %4 : $TestClass                   // id: %7

  %8 = integer_literal $Builtin.Int32, 0          // user: %9
  %9 = struct $Int32 (%8 : $Builtin.Int32)        // user: %10
  return %9 : $Int32                              // id: %10
} // end sil function 'main'
```
先看 main 函数，可以看出 SIL 语法清晰，结构也不复杂。
根据单词我们也大概可以猜出他的作用。

metatype 获得类型
function_ref 获得函数地址的引用
apply 调用函数
destroy_value 清理内存

如果你想知道具体命令具体是如何工作的，可以看看：<https://github.com/apple/swift/blob/master/docs/SIL.rst>

上面分成五段
第一段声明函数，拿到 metatype，把类型赋值给 %2
第二段调用 init 函数
第三段调用 testFunc 函数
第四段销毁不需要的变量
第五段 return

每个表达式右侧都有 // user: %7, %6, %5 或者 // id: %7 其实是定位注释
// user: %7 说明改赋值句的结果，会被 id 是 7 的表达式用到，赋值语句都会注释使用者是谁
// id: %7 说明这句的 id 就是 7，非赋值语句都会标注 id 是什么
没有标注 id 的怎么办？找到最近的一个 id，往上往下数就好啦，一行语句 id 增加 1。

其中我们之前提到过 class 中实例方法都会用 vtable 调用，vtable 调用的特点就是通过 class_method 拿到函数引用，然后交由 apply 调用。

SIL 文件底部就是我们一直提到的 vtable
```swift
sil_vtable TestClass {
  \#TestClass.testFunc!1: (TestClass) -> () -> () : @$S4test9TestClassC0A4FuncyyF    // TestClass.testFunc()
  \#TestClass.init!initializer.1: (TestClass.Type) -> () -> TestClass : @$S4test9TestClassCACycfc    // TestClass.init()
  \#TestClass.deinit!deallocator: @$S4test9TestClassCfD    // TestClass.__deallocating_deinit
}
```
`@$S4test9TestClassC0A4FuncyyF` 表示这个方法的类型是：`test.TestClass.testFunc() -> ()`
这个技术叫做 Name demangle 后面我们会讲到。

### Protocol 的调用

同样的，我们也可以看看 protocol 的 witness 是什么样的
有 protocol

```swift
protocol aProtocol {
    func testFunc()
}
// extension aProtocol {
//     func testFunc() {}
// }
class aStruct: aProtocol {
    func testFunc() {} // override defualt implementation
}
// let ins = aStruct()
// ins.testFunc()
let proto: aProtocol = aStruct()
proto.testFunc()
```

可以得到
```swift
// 1==========
// main
%10 = witness_method $@opened("7EA915EE-3F22-11E9-9F70-784F4386410B") aProtocol, #aProtocol.testFunc!1 : <Self where Self : aProtocol> (Self) -> () -> (), %9 : $*@opened("7EA915EE-3F22-11E9-9F70-784F4386410B") aProtocol : $@convention(witness_method: aProtocol) <τ_0_0 where τ_0_0 : aProtocol> (@in_guaranteed τ_0_0) -> () // type-defs: %9; user: %11
  %11 = apply %10<@opened("7EA915EE-3F22-11E9-9F70-784F4386410B") aProtocol>(%9) : $@convention(witness_method: aProtocol) <τ_0_0 where τ_0_0 : aProtocol> (@in_guaranteed τ_0_0) -> () // type-defs: %9


// 2==========
// protocol witness for aProtocol.testFunc() in conformance aStruct
sil private [transparent] [thunk] @$S4test7aStructCAA9aProtocolA2aDP0A4FuncyyFTW : $@convention(witness_method: aProtocol) (@in_guaranteed aStruct) -> () {
// %0                                             // users: %5, %1
bb0(%0 : $*aStruct):
  %1 = load_borrow %0 : $*aStruct                 // users: %5, %3, %2
  %2 = class_method %1 : $aStruct, #aStruct.testFunc!1 : (aStruct) -> () -> (), $@convention(method) (@guaranteed aStruct) -> () // user: %3
  %3 = apply %2(%1) : $@convention(method) (@guaranteed aStruct) -> ()
  %4 = tuple ()                                   // user: %6
  end_borrow %1 from %0 : $aStruct, $*aStruct     // id: %5
  return %4 : $()                                 // id: %6
} // end sil function '$S4test7aStructCAA9aProtocolA2aDP0A4FuncyyFTW'


// 3==========
sil_vtable aStruct {
  \#aStruct.testFunc!1: (aStruct) -> () -> () : @$S4test7aStructC0A4FuncyyF    // aStruct.testFunc()
  \#aStruct.init!initializer.1: (aStruct.Type) -> () -> aStruct : @$S4test7aStructCACycfc    // aStruct.init()
  \#aStruct.deinit!deallocator: @$S4test7aStructCfD    // aStruct.__deallocating_deinit
}


// 4==========
sil_witness_table hidden aStruct: aProtocol module test {
  method #aProtocol.testFunc!1: <Self where Self : aProtocol> (Self) -> () -> () : @$S4test7aStructCAA9aProtocolA2aDP0A4FuncyyFTW    // protocol witness for aProtocol.testFunc() in conformance aStruct
}
```

`//1` 在 main 中通过 `witness_method` 拿到了一个函数指针，然后直接进行 apply
继续往下看，根据我们以往的只是 witness_method 是通过查表得到的函数地址
`//4` 文件底部就是我们要找的 sil_witness_table 在后面的注释有一个 demangle 的函数签名
`//2` 通过字符串搜索，我们可以找到 witness 的函数实现
`//3` 他在内部进行了 `class_method` 查找和调用，最终指向 sil_vtable 中的方法

通过这个我们可以很清楚的看到 witness table 做的不是一个引用，而是指向一个 witness 调用函数。这个函数内部只做了一件事，就是调用真正实现协议的方法。

从 SIL 的角度来说，使用协议调用的性能成本还是很明显的。多一次查表，多一次调用。

> 根据注释我们还可以得出：witness 函数会针对每个实现了协议的对象都生成一份映射
> 感兴趣的话，可以找找看 SIL 文件中是不是有我们猜想的证据。

## 结语

总的来说，SIL 不像 IR 或者汇编一样逻辑复杂或是晦涩难懂。基于你对 Swift 的认知可以很轻松的读懂 SIL，并借此来帮助你分析一些程序中一些被隐藏的细节。

想更多的了解 SIL，不妨去读一下官方文档吧

[https://github.com/apple/swift/blob/master/docs/SIL.rst](https://github.com/apple/swift/blob/master/docs/SIL.rst#vtables)