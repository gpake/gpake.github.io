---
layout: post
title: Swift 如何给回调添加 @escaping 和 optional
date: 2018-12-12 11:50:38
tags:
typora-copy-images-to: ../images
---

## TL;DR

Swift 中写

```Swift
func aFunc(callback: @escaping (() -> Void)?) {}
```

这么声明函数你会得到一个错误：

```
@escaping attribute only applies to function types
// clang 会建议你
Replace '@escaping ' with ''
```

请接受他的建议，去掉 `@escaping`，留下参数的可选型 `?`

因为从 Swift 4 之后只有闭包 (Closure) 才会被隐式的标记为 `nonescaping` ，其他参数默认依然是 `escaping` 的。

<!-- more -->

同样的，给一个在方法参数中给一个 `String` 添加 `@escaping` 也会得到同样的报错和建议：

```swift
func tNormal(aStr: @escaping String) {}
// @escaping attribute only applies to function types
```

## 为什么？

因为 `optional` 在 swift 中其实是一个 `enum`

但是 `enum` 和 `@escaping` 有什么关系，一个是可选值，一个是生命周期。

### 关于生命周期

Swift 中函数的参数除了值类型默认都是递交生命周期管理权的，但是 closure 是一个特别的存在，因为他需要捕捉上下文中引用到的变量。

Swift 3 和之前默认所有参数都是 `@escaping` 的，closure 也不例外。一方面 Objective-C 就是这么处理的，一方面别的参数也这样。

不过这样用起来很不安全，一旦使用者不注意，就会循环引用。

所以在 Swift 4 开始，所有的 closure 都被默认处理为 `@nonescaping` ，来让使用者对闭包的生命周期有更强烈的感知。

### 回到 escaping 的问题

基于上面对参数声明周期的描述，我们可以知道，一旦闭包类型被声明为可选值后，他就会被一个 enum 捕捉，这时候他就成为了一个普通类型，而不是 closure。

所以编译系统不会对他进行特殊处理，默认也就是 escaping 的了。

### @escaping 和 optional 有什么区别？

结论：在声明周期上没有区别，我们可以用两段代码简单的对比一下：

> 这里用一个 iOSViewPlayground 来进行测试，环境基本和虚拟机一致。

1. 宿主持续存在，是否会被提前释放？
1. 宿主不存在后，闭包的生命周期是怎样的，宿主的生命周期是怎样的，捕获值的生命周期是怎样的？

#### 宿主持续存在

```swift
typealias Handler = () -> Void

class MyViewController : UIViewController {
    
    override func loadView() {
        let view = UIView()
        view.backgroundColor = .white
        self.view = view
    }
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        let t = testClass()
        t.tOptional { [weak self] in
            self?.pr(str: "in optinal caller")
        }
        t.tEscaping { [weak self] in
            self?.pr(str: "in escaping caller")
        }
        let s = Closure { [weak self] in
            print("super pre struct self:\(self)")
            self?.pr(str: "super in struct")
        }
    }
    
    func pr(str: String) {
        print(str)
    }
}

class testClass: NSObject {
    deinit {
        print("testClass deinit")
    }
    
    func tOptional(callback: Handler?) {
        DispatchQueue.main.asyncAfter(deadline: .now() + 1) {
            print("optinal: \(callback)")
            callback?()
        }
    }
    
    func tEscaping(callback: @escaping Handler) {
        DispatchQueue.main.asyncAfter(deadline: .now() + 1) {
            print("escaping: \(callback)")
            callback()
        }
    }
}
```

得到 console

可以看出，执行的 class 已经 deinit，callback 中的方法继续正常执行，打印出了我们期望的内容，说明在这个 case 下，escaping 和 optional 的效果是一样的。

```
testClass deinit

optinal: Optional((Function))
in optinal caller
escaping: (Function)
in escaping caller
```

即使对象先释放了，也可以正确的进行回调

#### 宿主被释放后的效果

和上面的代码结构类似

```swift
typealias Handler = () -> Void

struct Closure {
    let handler: Handler
}

class MyViewController : UIViewController {

    override func loadView() {
        let view = UIView()
        view.backgroundColor = .white
        self.view = view
    }

    override func viewDidLoad() {
        super.viewDidLoad()

        let t = testClass()
        // Closure 中 self 消失的 case
        t.test()
        // Closure 中 self 不消失的 case
        let s = Closure { [weak self] in
            print("super pre struct self:\(self)")
            self?.pr(str: "super in struct")
        }
        t.testStruct(aStruct: s)
    }

    func pr(str: String) {
        print(str)
    }
}

class testClass: NSObject {
    deinit {
        print("testClass deinit")
    }

    func pr(str: String) {
        print(str)
    }

    func test() {
        tOptional { [weak self] in
            print("pre optonal self:\(self)")
            self?.pr(str: "optinal")
        }
        tEscaping { [weak self] in
            print("pre escaping self:\(self)")
            self?.pr(str: "escaping")
        }
        let s = Closure { [weak self] in
            print("pre struct self:\(self)")
            self?.pr(str: "in struct")
        }
        testStruct(aStruct: s)
    }

    func tOptional(callback: Handler?) {
        DispatchQueue.main.asyncAfter(deadline: .now() + 1) {
            print("optinal: \(callback)")
            callback?()
        }
    }

    func tEscaping(callback: @escaping Handler) {
        DispatchQueue.main.asyncAfter(deadline: .now() + 1) {
            print("escaping: \(callback)")
            callback()
        }
    }

    func testStruct(aStruct: Closure) {
        DispatchQueue.main.asyncAfter(deadline: .now() + 1) {
            print("struct: \(aStruct)")
            aStruct.handler()
        }
    }
}
```

得到 console

在 testclass 中调用的内容，都因为 testclass (self) 被释放而无法继续，但是闭包除了捕获值以外的部分，还是在正常执行的。

```
testClass deinit

optinal: Optional((Function))
pre optonal self:nil
escaping: (Function)
pre escaping self:nil
struct: Closure(handler: (Function))
pre struct self:nil
struct: Closure(handler: (Function))

super pre struct self:Optional(<__lldb_expr_8.MyViewController: 0x7f8a471047e0>)
super in struct

```

### 结论

##### 对于可以传空的 closure，大胆使用 `?` 即可，不用担心 `@escaping` 的声明周期问题。


### 备注

最后对基于异步调用的函数参数类型和系统报错给个例子：

```swift
func aClosureError(callback: Handler) {
    DispatchQueue.main.async {
        print("\(callback)")
        // Closure use of non-escaping parameter 'callback' may allow it to escape
    }
}

// no error
func aClosureError(callback: Handler?) {
    DispatchQueue.main.async {
        print("\(callback)")
    }
}

// no error
func aNonClosure(str: String) {
    DispatchQueue.main.async {
        print("\(str)")
    }
}
```