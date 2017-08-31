> 之前写了一个关于[__unsafe_unretained](http://www.jianshu.com/p/0ca31b3e3ac0)特殊标识符讲解的翻译，其中也讲到了关于ARC情况下内存泄漏的问题，这片文章就是对之前问题的一个翻译讲解。

[点击进入StackOverFlow问答页](http://stackoverflow.com/questions/6260256/what-kind-of-leaks-does-automatic-reference-counting-in-objective-c-not-prevent)

### 一、问题

![问题](http://upload-images.jianshu.io/upload_images/711112-deebde8ef50bc518.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##### 问题翻译
在Mac和iOS平台上，未被释放的指针经常会导致内存泄漏。传统情况下，检查你的`alloc`、`copy`和`retain`来确定每个都有相对应的释放经常会变得极为重要。

伴随着Xcode 4.2到来的数据链，介绍了来自于最新一个版本的[LLVM编译器](http://llvm.org/)的自动引用计数（ARC），通过让编译器对内存进行管理，完全可以解决这个问题。这是非常酷的，并且它确实减少了不必要的、无用的开发时间，并防止很多不小心的内存泄漏，这些内存泄漏运用适当的保留或者释放会很容易修复。当你为Mac和iOS应用启用ARC时，即使自动释放池也需要进行不同的管理（因为你不应该分配自己的NSAutoreleasePools了）。

但是还有什么其他没有被阻止的内存泄漏我还需要注意么？

作为额外的，Mac OS X和iOS上的ARC，以及Mac OS X上的内存回收有什么区别？

## 二、回答

![回答](http://upload-images.jianshu.io/upload_images/711112-28abf3bcd07d9a63.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 回答翻译
您仍然需要注意的主要内存相关问题是循环引用。这种情况出现在当一个对象有一个强引用指针到另一个对象，但是目标指针也有一个强引用的指针指向最初的对象。å即使删除对这些对象的所有其他引用，他们仍然互相持有对方并且不会被释放。这也可能间接发生，通过可能在链中的最后一个对象引用回到较早对象的对象链。

这就是`__unsafe_unretained `和`__weak `所有权限定词存在的原因。前者不会保留任何它指向的对象，但是留下该对象消失和指向错误内存的可能性，而后者不保留对象，并且当其目标被释放时自动将其设置为nil。两者之中，`__weak`在支持它的平台上通常是首选的。

你可以使用这些限定符来执行像代理一类的事情，这种情况下你不希望对象保留其委托对象并且可能导致循环引用。

另外几个重要的内存相关问题是处理`Core Foundation`对象和使用`malloc()`分配像类型为`char *`的内存。ARC不管理这些类型，只管理`Objective-C`的对象，所以你仍然需要自己去处理他们。`Core Foundation`类型可能特别棘手，因为有时他们需要通过桥梁来匹配`Objective-C`对象，反之亦然。这意味着在`Core Foundation`类型和`Objective-C`之间桥接时，需要从ARC来回转移控制。一些和桥接相关的关键字已经被添加了，并且麦克·阿什（Mike Ash）在[他漫长的ARC写作中](https://www.mikeash.com/pyblog/friday-qa-2011-09-30-automatic-reference-counting.html)描述了各种桥梁案例。

除此之外，还有其他几个较不频繁但仍然是潜在的问题的案例，在[发布的规范](http://clang.llvm.org/docs/AutomaticReferenceCounting.html)中将详细介绍。

基于只要存在一个很强的指针就可以保持对象的很多新的行为，很大程度上与Mac上的`垃圾回收`非常相似。然而，技术基础是非常不同的。这种风格的内存管理依赖于我们在`Objective-C`中需要遵守的刚性保留/释放规则，而不是有一个定期运行的用来清理不再被指向的对象的`垃圾回收`进程。

ARC只需要重复我们已经做了多年的内存管理任务并将其分流到编译器，所以我们再也不用担心了。这样，你在`垃圾回收`平台上就没有停止问题或在经历的锯齿存储器配置问题了。我已经在`垃圾回收`的Mac应用程序中都经历了这两种，并且很想看到它们在`ARC`下的行为。

想看更多地关于`垃圾回收`和`ARC`，看看[Chris Lattner对`Objective-C`邮件列表的非常有趣的回应](https://lists.apple.com/archives/objc-language/2011/Jun/msg00013.html)，他列出了`ARC`对`Objective-C 2.0`垃圾回收的许多优点。我遇到了他所描述的几个`垃圾回收`问题。

## 另外

*  [简书地址](http://www.jianshu.com/u/ac41d8480d04)
*  [掘金地址](https://juejin.im/user/5730b373f38c840067d0d602)
