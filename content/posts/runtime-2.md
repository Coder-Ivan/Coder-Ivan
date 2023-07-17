+++
title = "iOS必备技能之Runtime（二）"
date = "2016-05-16"
tags = [
    "iOS",
    "runtime",
]
categories = [
    "iOS开发",
]
+++

![Apple.jpg](http://upload-images.jianshu.io/upload_images/1587104-3f5ffd530527cc8c.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
> 当向一个对象发送消息，如果方法被实现了，则直接在底层使用消息机制调动该方法，如果方法没有被实现，则响应链最前端是[《iOS必备技能之Runtime（一）》](http://www.jianshu.com/p/332698be594c)中提到的**动态方法解决方案**，如果方法被动态添加，那么这个消息会被对象接收；如果消息不能被接收，则响应链会寻找有没有实现**消息转发**的方法，让别的类去接收这个消息；如果消息转发也找不到对应的方法实现，那么程序才会**报错**（unrecognized selector sent to instance）。

## 四、消息转发
消息转发有两种，一种是对消息可定制的，一种是不可定制的。响应链优先响应不可定制的消息转发，如果没有实现就去响应可定制的消息转发。
#### 简单消息转发（不可定制）
简单的消息转发对转发的消息不可以修改，怎么发过来的怎么转走。
``` objectivec
- (id)forwardingTargetForSelector:(SEL)aSelector
```
这个方法赋予实现这个方法的类一个“传球”的能力，如果该类没有实现这个方法，那么`forwardingTargetForSelector:`返回一个其他类的对象，让其他类里面的方法代为实现。如下示例，该类没有实现method方法，但是转发给OtherClass来实现这个方法。
``` objectivec
- (id)forwardingTargetForSelector:(SEL)aSelector {
    if (aSelector == @selector(method)) {
       return [[OtherClass alloc] init];
    }
    return [super forwardingTargetForSelector:aSelector];
}
```
> 这个方法只能让我们把消息转发到另一个能处理这个消息的对象，但是无法处理消息的内容，比如参数和返回值。

#### 完整消息转发（可定制）
当前面两种方法分别返回NO和nil时，完整的消息转发`forwardInvocation:`就是保证程序不报“unrecognized selector sent to instance”错误的最后一关了。在完整消息转发里面，`forwardInvocation:`会对消息进行相应，对象会创建一个NSInvocation对象，把与尚未处理原始消息和参数一起封装起来。
``` objectivec
- (void)forwardInvocation:(NSInvocation *)anInvocation
```
为了理解转发的意图和范围，想象这样一个情景：假使在一个对象里你希望能够响应`negotiate `方法，并且其他几个不同类的对象也能够响应这个方法（这几个类实现了`negotiate `方法），最先想到的方法应该是直接发送消息到这几个类里面。
进一步思考，假使你的这个对象对`negotiate `方法的相应的实现恰恰在其他的类里面，一种实现方法就是让这个对象的类去继承那个类的方法，这样你就可以直接在在这个类里面调用`negotiate `方法了。但是，既然他们被分为不同的类，不属于同一继承体系，这也就意味着大部分情况下你往往不能够这么做。
虽然你不能继承`negotiate `方法，但是你可以通过把消息直接发送给其他类的对象的方式把这个方法“借”过来。如下：
``` objectivec
- (id)negotiate
{
    if ( [someOtherObject respondsTo:@selector(negotiate)] )
        return [someOtherObject negotiate];
    return self;
}
```
但是上面这种处理方式显得有些不灵活，特别是当你想把不止一个消息传递给其他对象的时候——你必须为每一个想要借过来的方法提供和上面类似的实现。另外，如果这些消息本来就是基于runtime的，会随着新的方法和类的改变而改变其实现，那么这种处理方式就变得捉襟见肘了。
`forwardInvocation:`就能很轻便地解决这个问题，它是这样工作的：当一个对象因为没有消息中对应方法名的方法而不能去响应消息的时候，runtime系统通知这个对象发送`forwardInvocation:`消息。每一个对象都从NSObject中继承得到`forwardInvocation:`方法，但是在NSObject的这个方法中只是简单调用了 `doesNotRecognizeSelector:`，这是个abstract方法（类似于C++的纯虚函数），当子类没有实现这个方法的时候，外部调用这个方法就会抛出异常。
> 只有当消息的接收者没能调用任何一个该类已经存在的方法的时候，`forwardInvocation:`方法才能够被调用来处理消息。比如，你想；要你的对象吧`negotiate`方法转发到其他类的对象，那么它本身就不能有`negotiate`方法。如果有这个方法，那么该类中就不会调用`forwardInvocation:`方法。

为了能够转发消息，所有的`forwardInvocation:`方法必须要做下面两件事：
- 决定消息要去哪儿
- 带着原始的参数向目标进行发送

消息可以用`invokeWithTarget:`来进行发送：

``` objectivec
- (void)forwardInvocation:(NSInvocation *)anInvocation
{
    if ([someOtherObject respondsToSelector:
            [anInvocation selector]])
        [anInvocation invokeWithTarget:someOtherObject];
    else
        [super forwardInvocation:anInvocation];
}
```
runtime系统首先会调用`methodSignatureForSelector:`方法来获得方法签名，方法签名记录了方法的参数和返回值的信息。
``` objectivec
- (NSMethodSignature*)methodSignatureForSelector:(SEL)selector
{
    NSMethodSignature* signature = [super methodSignatureForSelector:selector];
    if (!signature) {
       signature = [target methodSignatureForSelector:selector];
    }
    return signature;
}
```
> 转发的消息的返回值是要返回到原始的发送者的，所有的可以返回的类型都能被传递，包括id类型、结构体类型、双精度的浮点型数据等。

一个`forwardInvocation:`方法可以作为所有未被识别的消息的“分配中心”，把消息打包发送到不同的目标。或者说它可以是一个“换乘站”，把所有的消息发送到同一个目标。它可以把消息进行转化，或者只是简单地“吞掉”它，这样就会没有回应也没有报错。它还可以把几个消息集成到一个响应中。总结起来就是一句话，这个方法能做什么取决去它的实现。

#### 转发和多继承的异同
虽然Objectiv-C不支持多继承，但是使用转发来模仿继承，可以让Objective-C实现一部分多继承的特性。如下图所示，一个对象通过转发来响应消息就像是从其它类里面“借“或者说”继承“一个方法的实现一样。

![forwarding.gif](http://upload-images.jianshu.io/upload_images/1587104-a6eb9131c0f9c5cb.gif?imageMogr2/auto-orient/strip)
在上图中，Warrior类把`negotiate`方法转发到Diplomat类的实例中，就像是Warrior类在实现`negotiate`方法一样，它会对`negotiate`方法做出响应。
所以，转发和多继承有很多相似的特点，但是，它们有以下根本的不同之处：
1. 继承是把多种功能集中到了单个的对象中，它使类趋向于巨大化、全能化。相反地，转发是把职责进行了分化，它把一个问题分成了若干小的问题分配给不同的小对象，通过转发进行关联。
2. `respondsToSelector: `、`instanceRespondToSelector`和` isKindOfClass: `方法只看继承树，不看转发链，比如`[aWarrior respondsToSelector:@selector(negotiate)]`在这里为假，即使它能够接收`negotiate`的消息并且进行响应。如果使用了协议，那么`conformsToProtocol:`方法也在此列。但是你可以通过重写这些方法让他在转发中发挥和继承中一样的作用：

``` objectivec
- (BOOL)respondsToSelector:(SEL)aSelector
{
    if ( [super respondsToSelector:aSelector] )
        return YES;
    else {
        /* Here, test whether the aSelector message can     *
         * be forwarded to another object and whether that  *
         * object can respond to it. Return YES if it can.  */
    }
    return NO;
}
```
> 这个技巧只适合在没有其他可以使用的方法的情况下使用，它并不可以取代继承的作用。

#### 代理对象
转发不只只是模仿多继承，它尽可能地产生了更轻量化的对象去实现尽可能多的原本属于很多相关对象的特性。代理对象作为其他类的替身，把消息如实传递过去。代理对象更关注的是在向其他类的对象(远程对象)抓发消息这个过程的细节，比如确保在连接远程对象的过程中每个参数被正确传达等。但是它在这个过程中并不是把远程对象做了副本，而是创建了一个对应于远程对象的本地地址，通过这个地址，远程对象在其他应用可以收到这些转发的消息。
还有一种情况，比如你有一个对象需要处理大批量的数据——创建一个复杂的图片或者从本地磁盘中读取文件等，这种情况下使用代理对象也是很合适的。创建一个这样的对象是很耗时的，所以我们希望只有在它确实需要或者在系统资源临时闲置的时候才回去创建它，但是又要保证存在这个对象的占位符使得其他对象涉及到这个对象的时候能够正常运行。
在这种情况下，你可以在最开始的时候不要创建一个全功能的对象，而是创建一个代理对象给它。这个代理对象主要就是为即将创建的大型对象做一个占位，或是在时机成熟的时候，进行消息转发。当代理对象的`forwardInvocation：`方法第一次被调用的时候，它要确认代理的对象是不是存在了，如果还没有存在，就去创建它。在这个对象的其他功能被需要之前，代理对象和这个对象在意义上其实就是一样的。



---
参考：[《Objective-C Runtime Programing Guide》](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Introduction/Introduction.html)

---
链接：
[《iOS必备技能之Runtime（一）》](http://www.jianshu.com/p/332698be594c)