+++
title = "iOS必备技能之Runtime（一）"
date = "2016-05-10"
tags = [
    "iOS",
    "runtime",
]
categories = [
    "iOS开发",
]
+++

> Runtime 是一个比较底层的C语言的API，可以翻译为“运行时”。作为使用运行时机制的OC语言的底层，它在程序运行时把OC语言转换成了runtime的C语言代码。学习并理解runtime是OC学习历程中的不可或缺的一大块儿。

## 一、消息机制
> 调用方法的本质就是发送消息。

发送消息常见的有四个方法：
- `objc_msgSend` 向一个类的实例发送消息，返回id类型数据。**（这也是最常用的一个发送消息的方法）**
- `objc_msgSend_stret` 向一个类的实例发送消息，返回结构体类型数据。
- `objc_msgSendSuper` 向一个类的实例的父类发送消息，返回id类型数据。
- `objc_msgSendSuper_stret` 向一个类的实例的父类发送消息，返回结构体类型的数据。

> 在OC语言中，方法的真正实现是在程序运行的时候绑定的，假如一个方法只有声明，没有实现，调用后在编译阶段是不会出错的，真正报错是在运行的时候。

```
[receiver message]
```
以上方法在运行时会被转化为
``` objectivec
//receiver是方法的调用者，selector是方法名
objc_msgSend(receiver, selector)
//如果有参数
objc_msgSend(receiver, selector, arg1, arg2, ...)
```
#### 发送消息的原理
objc_msgSend为了完成动态绑定，进行了以下三步：
1. 首先它要先根据方法名找到方法的具体实现程序，因为多态性，同一个方法在不同的类里面可以有不同的实现，所以查找主要依靠寻找receiver所在的类。
2. 传递参数，调用该方法的实现程序。
3. 把该程序的返回值作为方法自己的返回值。

``` objectivec
//runtime中对类的定义
struct objc_class {
    Class isa  OBJC_ISA_AVAILABILITY;
#if !__OBJC2__
    Class super_class                                        OBJC2_UNAVAILABLE;
    const char *name                                         OBJC2_UNAVAILABLE;
    long version                                             OBJC2_UNAVAILABLE;
    long info                                                OBJC2_UNAVAILABLE;
    long instance_size                                       OBJC2_UNAVAILABLE;
    struct objc_ivar_list *ivars                             OBJC2_UNAVAILABLE;
    struct objc_method_list **methodLists                    OBJC2_UNAVAILABLE;
    struct objc_cache *cache                                 OBJC2_UNAVAILABLE;
    struct objc_protocol_list *protocols                     OBJC2_UNAVAILABLE;
#endif
} OBJC2_UNAVAILABLE;

//runtime中对实例的定义
struct objc_object {
    Class isa  OBJC_ISA_AVAILABILITY;
};
```
如上runtime中对类的定义，每一个类都有指向父类的指针(`super_class`)和一个方法调度表（`objc_method_list **methodLists`:根据方法名SEL查找该方法的具体实现的地址IMP），当向一个对象发送消息的时候，该对象通过isa指针找到该对象的类(实际上，实例的定义里面也只有这个指针，没有别的了)，在类的调度表查找该方法名，当找不到的时候，通过指向父类的指针找到该类的父类，然后在该类的父类中继续查找该方法名，这样递归查找一直到NSObject类为止(NSProxy类除外，它不属于NSObject子类)。如果查找到该方法名，根据调度表找到该方法的实现的地址进行调用。如下图所示

![ Messaging Framework](http://upload-images.jianshu.io/upload_images/1587104-25f8eca901527249.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

为了加速发送消息的进程，runtime系统会把使用过的方法名和对应的内存地址缓存起来，每个类都有一个单独的缓存空间，其中包含**自己类的方法和继承自父类的方法**。在查找调度表之前，runtime系统会首先在缓存中进行查找。

#### 使用隐藏的参数
当objc_msgSend找到方法的实现程序时，它调用这个程序并传递所有方法的参数给它，这其中还包含两个隐藏的参数：
- 消息的接收对象
- 调用方法的方法名（selector)

这两个参数虽然没有在方法中进行定义，但是你可以很方便地使用它们。消息的接收对象通过**self**来引用，方法名通过**_cmd**来引用。
``` objectivec
- strange
{
    id  target = getTheReceiver();
    SEL method = getTheMethod();
 
    if ( target == self || method == _cmd )
        return nil;
    return [target performSelector:method];
}
```

#### 获取方法的地址
避免动态绑定的唯一方法就是直接获得方法的地址然后把它当做函数一样来调用。当一个方法被连续多次执行，而你又不想每次都用消息机制造成额外的开支，这种办法就是一个合适的使用时机。
下面的例子展示了如何节省开支多次调用setFilled:方法
``` objectivec
void (*setter)(id, SEL, BOOL);
int i;
 
setter = (void (*)(id, SEL, BOOL))[target
    methodForSelector:@selector(setFilled:)];
for ( i = 0 ; i < 1000 ; i++ )
    setter(targetList[i], @selector(setFilled:), YES);
```
通过`methodForSelector:`方法，你可以请求得到指向实现该方法的程序的指针，然后通过这个指针调用该程序。值的注意的是，参数和返回值要正确声明，而且参数中id和SEL要进行显式声明。

---
## 二、动态方法
假如你想动态地为方法提供实现，OC使用`@dynamic`实现了这个特性。
``` objectivec
@dynamic propertyName;
```
这样就会通知编译器和这个属性相关的方法将会动态提供。你可以通过方法`resolveInstanceMethod:`和`resolveClassMethod:`分别为类方法和实例方法动态地提供实现。
> 一个OC的方法其实就是由C语言的函数再加上**至少**两个参数(self和_cmd)组成的。  

你可以把一个函数通过`class_addMethod`作为方法添加到一个类中去。给定以下一个函数：
``` objectivec
void dynamicMethodIMP(id self, SEL _cmd) {
    // implementation ....
}
```
你可以通过`resolveInstanceMethod:`这个方法把上面的函数以方法名(`resolveThisMethodDynamically`)动态地添加到一个类(MyClass)里面。具体实现方式如下：
``` objectivec
@implementation MyClass
+ (BOOL)resolveInstanceMethod:(SEL)aSEL
{
    if (aSEL == @selector(resolveThisMethodDynamically)) {
          class_addMethod([self class], aSEL, (IMP) dynamicMethodIMP, "v@:");
          return YES;
    }
    return [super resolveInstanceMethod:aSEL];
}
@end
```
这其中，`class_addMethod`这个方法有四个参数，第一个是要添加方法的类，第二个是要添加的方法名，第三个是这个方法的实现函数的指针（值的注意的是，这个函数必须显式地把`self`和`_cmd`这两个参数写出来），第四个是方法的参数数组，在这里它是用的类型编码的方式进行表示的，因为方法一定含有`self`和`_cmd`这两个参数，所以字符数组的第二个和第三个字符一定是"@:",第一个字符代表返回值，这里为空用“v”来表示。相关知识点请见下文。

---
## 三、类型编码
为了使runtime系统更加简洁，编译器把每个方法的返回值和参数的类型都分别使用一个字符来编码，然后再把它们关联到方法选择器(selector)上。因为这种编码方案在其它环境中也很实用，所以我们可以很方便地使用`@encode()`编译器指令来自定义类似的编码。
``` objectivec
char *buf1 = @encode(int **);
char *buf2 = @encode(struct key);
char *buf3 = @encode(Rectangle);
```
> 一般来说，不管是基本类型，还是指针，或者结构体，或者联合体，甚至可以是类名，只要这个类型能够作为C语言中`sizeof()`的参数，那么它就能被进行编码。

下表便是已经定义了的类型编码，使用`@encode()`编译器指令自定义编码的时候一定要避开这些字符。

![](http://upload-images.jianshu.io/upload_images/1587104-19dda2f70f36ff4f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![Objective-C type encodings](http://upload-images.jianshu.io/upload_images/1587104-751071ca207683ca.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

*特别注意，OC不支持`long double`类型，因此`@encode(long double)`会返回字符“d"，意义为`double`。*
结构体的类型编码是按照结构体内部的类型的顺序来表示的，比如
``` objectivec
typedef struct example {
    id   anObject;
    char *aString;
    int  anInt;
} Example;
```
会被编码为：
``` objectivec
{example=@*i}
```
由第一章内容可以得知，类的实例的定义是一个只包含isa指针的结构体，所以`[NSObject class]`会被编码为
``` objectivec
{ NSObject=# }
```
具体应用方面，上一章`class_addMethod`最后一个参数就是使用的类型编码来表示的函数返回值和参数的类型。

---

参考：[《Objective-C Runtime Programing Guide》](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Introduction/Introduction.html)

---
链接：
[iOS必备技能之Runtime（二）](http://www.jianshu.com/p/1e5cc20ed552)

_文章会不定期进行增添和更新，欢迎订阅和收藏！_