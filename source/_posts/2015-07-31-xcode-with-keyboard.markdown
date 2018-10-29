---
layout: post
title:  "用键盘掌握你的Xcode"
date:   2015-07-31 23:58:24
categories: Xcode
---


要尽量少用鼠标！因为会变成鼠标手！ = ='''

哈哈，其实键盘也好不到那里去，但是把手从键盘上移开去找鼠标，然后再定位屏幕上的位置其实是个负担挺大的问题，如果你有不止一块屏幕的话那就更糟了。

好在Xcode为大多数操作都预留了快捷键。下面我会介绍一下我平常是怎么用键盘与Xcode`快乐地玩耍`的。

###Mac 键盘符号
首先来看一下这些神奇的符号。[来自这里](http://www.raizlabs.com/dev/2015/03/spicing-up-xcode/)

⌘ Command

⇧ Shift

⌥ Option

⌃ Control

↩ Return

← ↓ → ↑ 方向键

### 开胃菜
先来点常规的大家都知道的
⌘O 打开  
⌘S 保存（随手保存的好习惯，谨防被Xcode搞崩溃）  
当然还有我（da jia）最喜欢的⌘C & ⌘V 😂

### 导航
***
#### ⌘1 ⌘2...8 就对应了下图从左到右的八个标签

![XcodeCmdNavi](../images/XcodeCmd1-8Navi.png)

功能嘛，分别是这些 Navigator（下表中项目==workspace）

  ⌘+Number   | Navigator名称 | 备注
------------ | ---------- | ------------
     1       | Project    | 项目目录，文件等
     2       | Symbol     | 项目中所有的类名和其实现的方法名
     3       | Find       | 全项目查找/替换
     4       | Issue      | 项目中的 Warning 和 Error
     5       | Test       | 项目中的测试
     6       | Debug      | 运行时调试区
     7       | Breakpoint | 可爱的断点们
     8       | Report     | 编译报告（成功了失败了什么的）

点击其中列出来的条目即可查看对应的代码。
***
#### 焦点在左侧栏了，还怎么回去敲BUG？！
别着急，按一下`⌘J` 是不是看到了一个可爱的小窗口？！

![焦点转移](../images/xcode-move-fouus.png)

它默认就指向代码区，只要一下 ↩ 就可以将焦点转移到代码上了。

当然，你也可以用其他的办法，比如：`⌘L`。然后你会看到这个：

![跳转到当前文件的某行代码](../images/xcode-line-number.png)

输入一下行号，然后就可以快速跳转咯。（我觉的还是上面那个快，不过这个在利用 log 查代码的时候悔恨有用，因为 log 一般都会带有出问题代码的行号，对吧对吧？快回答：“是的！”）

哦，顺便说一下，`⌘J` 之后出现的窗口可以用方向键上下左右哦，快玩玩吧。
***
#### 右边的那一堆呢？

右边是一堆就不叫 Navigator 了，他们叫Inspector。
只要`⌥⌘ 12345678`就可以啦~

功能嘛，分别是这些 Inspector（下表中项目==workspace）

文本文件状态下:

  ⌥⌘+Number  | Inspector名称 | 备注
------------ | ---------- | ------------
     1       | File       | 当前文件的属性
     2       | Quick Help | 当前类或方法的帮助，相当于按着`⌥`点，用鼠标点某个类看到的东西

Xib/Storyboard状态下:

  ⌥⌘+Number  | Inspector名称 | 备注
------------ | ---------- | ------------
     1       | File       | 当前 xib/storyboard的文件属性，还可以控制是否使用autoLayout, size classes
     2       | Quick Help | 木有帮助
     3       | Identity   | 定义当前 View 的标识
     4       | Attribute  | 控制 View 的一些属性，比如 title，backgroundColor
     5       | Size       | 控制 view 的尺寸，位置，或是constrains
     6       | Connections| 控制IBOutlets

Debug View Hierarchy状态下：

  ⌥⌘+Number  | Inspector名称 | 备注
------------ | ---------- | ------------
     1       | File       | 不可用
     2       | Quick Help | 不可用
     3       | Object     | 当前选中 View 的 ClassName 和 内存地址
     5       | Size       | 当前选中 View 的 Ui 布局属性

