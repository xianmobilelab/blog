---
title: Swift4.0 Migration
date: 2017-10-24 20:44:32
tags: iOS
---
> 作者: 郭鑫
> 原文: [http://blog.emagrorrim.com/2017/10/22/Swift4-0-Migration/](http://blog.emagrorrim.com/2017/10/22/Swift4-0-Migration/)

{% asset_img swift-4.png "" %}

最近完成了Swift 4.0的迁移，记录下迁移过程中遇到的坑

## Agenda

- Swift 4.0 简介
- Swift 4.0 语法变化简介
- Swift 4.0 Migration 流程
- Swift 4.0 Migration 过程中遇到的“坑”

<!-- more -->

## Swift 4.0 简介
Swift 4.0的目标由ABI(Application Binary Interface)稳定，变为了源码兼容，其中ABI的兼容和源码的兼容都代表什么意思呢？

- ABI 兼容：Swift 3 预先编译出来的库，不用重新编译，也可以在 Swift 4 中正常链接
- 源码兼容：Swift 3 编写的源码不用经过修改，就可以在 Swift 4 中正常编译

Swift 3.2，源码兼容，Swift 3.0 几乎不需要修改，只需重新编译，Swift 4.0 与 Swift 3.2 framework支持混编，可见Swift 4.0的兼容性还是比较高的

{% asset_img re-learn-the-swift.png "" %}


## Swift 4.0 语法变化简介

{% asset_img grammar-changes.png "" %}

Swift 4.0语法变化还是不少的，但是值得一提的大概是以下四个：

### String
首先是String再次变成了一个Collection，Swift 1.0的时候，String是一个Collection类型，Swift 2.0将其变成了一个独有的String类型，Swift 4.0其再次变成了Collection类型
```swift
let greeting = "Hello, World!"
greeting.count
for char in greeting {
    print(char)
}
```
其中substring方法被弃用，现在我们使用集合的方式来查找子字符串，且子字符串现在是一个新类型即`Substring`
```swift
let greeting = "Hello, World!"
let comma = greeting.index(of: ",")!
let substring = greeting[..<comma]
```

### private

以前我们总抱怨Swift中的Extension访问不到private的属性或方法，我们不得不使用fileprivate来解决这个问题，在Swift 4.0中，这个问题得到了解决，我们终于可以在Extension中访问到private的属性了。

#### Swift 3.x
```swift
class Demo {
  fileprivate let id: Int

  init(id: Int) {
    self.id = id
  }
}

extension Demo: Equatable {
  static func ==(lhs: Demo, rhs: Demo) -> Bool {
    return lhs.id == rhs.id
  }
}
```
#### Swift 4.0
```swift
class Demo {
  private let id: Int

  init(id: Int) {
    self.id = id
  }
}

extension Demo: Equatable {
  static func ==(lhs: Demo, rhs: Demo) -> Bool {
    return lhs.id == rhs.id
  }
}
```

### 支持组合Class和Protocol来定义变量
```swift
let demo: (Demo & DemoProtocol)?
```

### @objc
最后也是最大的变化，`@objc`，Apple将Swift从默认与OC兼容改为了手动兼容，现在我们如果想在OC代码中调用Swift代码，我们需要手动在属性和方法前添加`@objc`关键字。如下：
```swift
@objc let demo: Demo!
@objc func doSomething()
```
苹果对此的解释是这减小了App的尺寸，举例的话即Apple自己的Music App减少了大约6%，但是我觉得，**这可能以为着Apple打算分裂Swift和OC语言了，甚至Apple打算弃用Objective-C语言了。**

### Swift 4.0 Migration 流程

{% asset_img swift.png "" %}

### 环境
#### Xcode 9
Xcode 9是Apple官方推出的IDE，自带Swift 4.0的编译环境。
#### Xcode 8.3
有些开发人员可能会进行不止一个项目的开发，可能会希望保留Xcode 8.3，这时我们也可以使用Xcode 8.3来开发Swift 4。其步骤如下：
- 从 swift.org 下载最新的 Swift 4.0 snapshot
- 运行安装程序
- 在Xcode中，`Xcode > Toolchains > Manage Toolchains…` 然后选择 snapshot

### 流程

{% asset_img migration-process.png "" %}

迁移流程如上图所示，具体可分为如下几步：
- 从Build Settings中选择Swift版本

{% asset_img select-version.png "" %}

- 使用内置工具完成迁移 `Edit > Convert > To Current Swift Syntax…`

{% asset_img convert-to-currect-version.png "" %}

- 选择要迁移的Target

{% asset_img select-target.png "" %}

- 选择你希望迁移工具的迁移行为

{% asset_img select-convertion-mode.png "" %}

- 编译你的代码

之后编译你的代码，你会看到如下图所示的警告，将其对应的方法添加`@objc`即可

{% asset_img objc-warning.png "" %}

- 在Build Settings中将`Swift 3 @objc Inference`设置为 Default

{% asset_img swift-3-objc-inference.png "" %}


## Swift 4.0 迁移过程中遇到的问题

{% asset_img questions.png "" %}

### @objc 带来的“坑”
虽然90%的`@objc`都会在编译期间被检查出来，但是仍然有一些我们无法在编译检查出来，就是运行时的错误。
```objc
if ([controller respondsToSelector:@selector(method_name)])
{
    [controller performSelector:@selector(method_name)];
}
```
上面这对代码问题在于，如果我们`method_name`没有加`@objc`，那么我们上面的if判断就会失败，并且没有警告或错误，你会发现你的功能直接失效了。
```objc
[demo setValue:value toKeyPath:path];
```
上面这对代码问题在于，我们Set keyPath的时候，如果被set的path对应的属性没有加`@objc`，那么我们调用setKeyPath就会造成App崩溃，这里同样不会有警告或错误，如果错误本身不是在主要功能中，很可能被我们忽略而造成线上Bug。

### Cocoapods Swift3， Swift4 版本混编的问题
前文我们提到了Swift这个版本的源码兼容性很高，且支持Swift3.2和Swift4.0 framework的混编，然而...

{% asset_img problems.png "" %}


我们会发现当我们将项目的Swift版本改为4.0之后，我们所有Cocoapods安装的依赖的Swift版本也变为了4.0，这导致我们完全浪费了Apple的Swift3.2和4.0 framework混编的优势，导致我们不得不等待我们所有依赖的库升级到Swift4.0之后才能升级我们自己的Target到Swift4.0，或者不得不用一个私有的Pod Module自己升级，增加了许多不必要的工作量，这里给大家分享一下我们项目上的解决方案。

#### Pod install hook
第一种是我们修改Pod install hook，来将我们希望的依赖的Swift版本手动改为Swift 3.2版本，如下面代码所示，我们将Quick改为了Swift3.2版本（Quick最为常用的测试库，从1.2.0版本开始支持Swift 4.0，之前版本为1.1.0不支持Swift4.0）。
```ruby
post_install do |installer|
  installer.pods_project.targets.each do |target|
    target.build_configurations.each do |config|
      if config.build_settings['PRODUCT_NAME'] == 'Quick'
        config.build_settings['SWIFT_VERSION'] = 3.2
      end
    end
  end
end
```
#### xcconfig文件
我们可以将Swift的版本写入xcconfig文件中，删除Build Settings里面的设置，这样也可以让我们的Pod module的Swift版本设置为3.2版本

如果我们使用的库连3.2都编译不过，或者说使用的库Swift版本低于3.0（因为Swift 3的代码不许要修改就可以用Swift3.2的编译器编译），那么我建议我们应该暂停升级Swift4.0，先将这个库替换成在持续更新的库。

## 总结
总得来说，这次版本迁移并不像Swift前几个版本那样困难，想对兼容性也比较好。本文介绍了Swift 4.0的语法变化，例如`@objc`，`private`的问题，之后讨论了迁移的基本流程，最后讨论了我遇到的一些"坑"，例如`@objc`和`Cocoapods`带来的问题。现在我们该开始期待Swift 5.0了，希望下个版本可以做到ABI稳定。

