---
title: Swift Currying
date: 2019-02-02 00:06:13
tags:
---

## TL;DR

Swift 中的实例方法是柯里化的。本质是把 self 当做隐式参数的类方法，它会返回一个给实例用的函数。 

我们先定义一个简单的  struct，然后调用：

```swift
struct aCls { 
    func aFunc() {} 
    func aPrint(str: String) { 
        print(str) 
    } 
} 

let ins = aCls() 
ins.aFunc() 
ins.aPrint(str: "Hello") 
```

其实，系统是这么帮你处理的 
```swift
aCls.aFunc(ins)() 
aCls.aPrint(ins)(str: "Hello") 
```

<!-- more -->



最近在看 Swift 方法调用这方面的内容，这篇也是方法调用系列的一个番外篇。

## 柯里化

如果你还不了解 Currying，可以看这里 

[柯里化 (CURRYING)](https://swifter.tips/currying/)

[wikipedia: Currying](https://en.wikipedia.org/wiki/Currying)

[swift-evolution: Removing currying `func` declaration syntax](https://github.com/apple/swift-evolution/blob/master/proposals/0002-remove-currying.md)

虽然 Swift 3 已经把这玩意去掉了，不过这个概念还是对降低程序复杂度有一些帮助的。

## 发生了什么？

*我们在这里使用 SIL 来进行辅助分析，如果你还不了解 SIL，先自己查查，我后面会有一篇专门说这个。*

还是拿着上面的类声明来看。

```swift
struct aCls { 
    var aPropStr: String = "Hello" 
    func aFunc() {} 
    func aPrint(str: String) { 
        print(self.aPropStr) 
    } 
} 

let ins = aCls() 
ins.aFunc() 
ins.aPrint(str: "Hello") 
```

可以得到一大串代码，我们简单清理一下，得到如下：

```swift
// aCls.aFunc() 
sil hidden @$S4test4aClsV5aFuncyyF : $@convention(method) (aCls) -> () { 
// %0                                             // user: %1 
bb0(%0 : $aCls): 
  debug_value %0 : $aCls, let, name "self", argno 1 // id: %1 
  %2 = tuple ()                                   // user: %3 
  return %2 : $()                                 // id: %3 
} // end sil function '$S4test4aClsV5aFuncyyF'


// -------------------
// aCls.aPrint(str:) 
sil hidden @$S4test4aClsV6aPrint3strySS_tF : $@convention(method) (@guaranteed String, @guaranteed aCls) -> () { 
// %0                                             // user: %2 
// %1                                             // users: %14, %3 
bb0(%0 : $String, %1 : $aCls): 
  debug_value %0 : $String, let, name "str", argno 1 // id: %2 
  debug_value %1 : $aCls, let, name "self", argno 2 // id: %3 
  %4 = integer_literal $Builtin.Word, 1           // user: %6 
  // function_ref _allocateUninitializedArray<A>(_:) 

  // init
    
  // 取出 aPropStr，copy 值（因为 String 是当做值类型操作的） 
  %14 = struct_extract %1 : $aCls, #aCls.aPropStr // user: %15 
  %15 = copy_value %14 : $String                  // user: %17 
  %16 = init_existential_addr %13 : $*Any, $String // user: %17 
  store %15 to [init] %16 : $*String              // id: %17 
  /* ... */ 
} 
```

可以看到 SIL 中只有 aCls.aFunc(), aCls.aPrint(str:) 的声明实现，并没有表现出实例相关的方法。
```swift
// aCls.aFunc()
bb0(%0 : $aCls):

// aCls.aPrint(str:)
bb0(%0 : $String, %1 : $aCls):
```
而且在两处 `bb0` 这行，你可以看到参数列表的最后一位是一个 aCls 类型的对象，也就是实例对象。

进一步，我们调用实例变量就可以看到这个实例参数会被怎么使用了，比如在 `aPrint(str:)` 方法中我们调用了一个实例属性。

```swift
// aCls.aPrint(str:)
bb0(%0 : $String, %1 : $aCls):

// 取出 aPropStr，并 copy 值（因为 String 是当做值类型操作的） 
%14 = struct_extract %1 : $aCls, #aCls.aPropStr // user: %15 
%15 = copy_value %14 : $String                  // user: %17 
%16 = init_existential_addr %13 : $*Any, $String // user: %17 
store %15 to [init] %16 : $*String              // id: %17 
```

可以看到通过传入的实例，来进行和实例有关的操作，比如读取 property：
`bb0 %1: aCls` 拿到实例变量
`%14 = struct_extract %1 : $aCls, #aCls.aPropStr // user: %15`  取出属性的值，并在下面进行 copy 等操作。

## 总结
柯里化是一种很好的拆分思想，可以在有重复操作的情况下降低耦合，提高代码复用性。