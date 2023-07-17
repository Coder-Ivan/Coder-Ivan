+++
title = "浅明分析Swift循环引用"
date = "2017-10-13"
tags = [
    "iOS",
    "swift",
    "循环引用"
]
categories = [
    "iOS开发",
]
+++

看过不少分析Swift解决循环引用的文章，分析weak和unowned的区别等等，可能是不太符合我的思路，一直感觉很模糊，在平时使用的时候对什么时候用weak，什么时候用unowned方面还是不太明确，干脆自己在这方面进行了一次整理。
### 自动引用计数（ARC）
Swift和OC一样，使用的是自动引用计数的机制来追踪和管理APP的内存。顾名思义，自动引用计数是自动进行的，并不需要我们手动去参与内存的管理——当一个实例使用完了的时候，会自动对其占用的内存进行释放。当然，ARC管理的只是引用类型，值类型的（比如结构体和枚举）不在其管理范围之内。
ARC其实就干了三件事：
- **为新创建的实例分配内存**
- **确保使用中的实例不会被销毁**
- **确保使用完的实例被正确释放，腾出占用的内存空间**

上面三板斧的实现是靠ARC维护一个计数来实现的，当初始化的时候，引用计数为1；每次有新的对这个实例的引用的时候，引用计数加1；每次对应引用被置为nil时，引用计数减1；当引用计数为0的时候，该实例被销毁，回收内存空间。

举个例子吧，假如有一个类如下：
``` swift
class Person {
    let name: String
    init(name: String) { self.name = name }
    deinit { print("\(name)被注销了") }
}
```
这个类很简单，一个name的属性，一个构造函数，一个析构函数。创建该类的新实例的时候，调用构造函数，销毁该实例的时候，调用析构函数。

下面，我们创建一个Person类的实例，如下
``` swift
Person.init(name: "Ivan")
```
我们只是创建了这个实例，在正常使用中我们是不会单纯这样做的，没有意义。我们会把这个实例赋值给某个变量来进行使用。
``` swift
let person1 = Person.init(name: "Ivan")    // 引用计数加1，现在为1
```
这时候，person1和这个Person类的新实例直接建立了一个强引用，该实例的引用计数加1。也是因为该实例有强引用，所以它所在的内存空间不会被销毁，在ARC眼中它还有利用价值。

假如这个实例也赋值给了其他变量，如下
``` swift
let person2 = person1    // 引用计数加1，现在为2
let person3 = person1    // 引用计数加1，现在为3
let person4 = Person.init(name: "Jack")    // 引用计数加1，这是个新的实例，这个实例引用计数现在为1
```
当变量对这个实例的引用被销毁，即置为nil的时候，就会减少这个实例的引用计数，当引用计数为0 的是，这个实例即被销毁，回收内存空间。
``` swift
person1 = nil    // 引用计数减1，现在为2
person2 = nil    // 引用计数减1，现在为1
person3 = nil    // 引用计数减1，现在Person类的这个实例被销毁了
```
> 但是ARC毕竟不是智能的，默认它会把所有的引用归为强引用，只要还在被其他的属性、常量、变量所使用，它是不会被释放的。但是凡事总有特殊情况，这时候就需要对ARC释放内存的方式进行提示（weak，unowned）。
### 循环引用
> 墨菲定律：如果事情有变坏的可能，不管这种可能性有多小，它总会发生。

我们再举一个例子，有下面2个相关类：
``` swift
class Person {
    let name: String
    var pet: Dog?
    init(name: String) { self.name = name }
    deinit { print("\(name)被注销了") }
}

class Dog {
    let nickName: String
    let owner: Person?
    init(species: String) { self.species = species }
    deinit { print("\(nickName)被注销了") }
}
```
发现Person类多了一个Dog属性，Dog类里面也有一个Person属性。一个人可以拥有一只宠物狗，一只狗也可以拥有一个主人；同时因为一个人也可以没有宠物，一只狗也可以是一只野狗，所以这两个变量都是可选的。

那么问题来了：假如我们同时创建了这两个类的实例并且赋值给了两个变量会怎么样？
``` swift
var ivan = Person.init(name: "ivan")
var wawa = Dog.init(nickName: "wawa")
```

![](http://upload-images.jianshu.io/upload_images/1587104-03e0582268db092e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
就像上图一样，ivan变量建立了对Person实例的强引用，wawa建立了对Dog实例的强引用。其实没什么，因为两者并没有什么关系。但是，如果我们加上下面的语句：
``` swift
ivan.pet = wawa
wawa.owner = ivan
```
那么一切都不一样了，如下图：

![](http://upload-images.jianshu.io/upload_images/1587104-0f89c80970e12b89.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
此时在之前两个强引用的基础上，多了Person实例中的pet变量对Dog实例的强引用，以及Dog实例的owner变量对Person实例的强引用。

如果这时候，我们结束了对这两个实例的使用，想要销毁它们来腾出内存空间，这时候就出问题了。
``` swift
ivan = nil
wawa = nil
```
![](http://upload-images.jianshu.io/upload_images/1587104-19ef629b885b84a9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
如上图，我们的变量到实例直接的引用已经没有了，但是这两个实例会被销毁吗？答案是否定的。因为他们直接还相互存在引用，只要还有对实例的引用，那么实例就不会轻易被销毁，内存空间也不会被正确释放，这就是因为循环引用导致的内存泄漏。
### 解决循环引用导致的内存泄漏问题
从上面的例子里面可以看到，存在一种可能，ARC会维护一种永远不会置为0的实例：如果两个实例互相持有对方的强引用，那么会互相让对方永远至少存在1的引用计数，这就造成了循环强引用。

首先，我们在平时的类关系设计的时候就会事先考虑好，尽量去避免对象实例之间的相互持有，也就避免了循环引用。

当然，在设计上无法避免这样的设定的时候，就可以对类关系之间的关系进行重新定义，把强引用改为弱引用或者无主引用。弱引用和无主引用允许发生了循环引用的两个实例之间的一个实例引用另外一个实例而不保持强引用。
##### 那么在什么时候用弱引用，什么时候用无主引用呢？
**在两个实例中，假如一个实例引用的另外一个实例具有更短的生命周期，那么就使用弱引用(weak)来引用这个实例，如果引用的实例具有相同的或者更长的生命周期的时候，那么就使用无主引用(unowned)。**

这两个在使用的时候一定要注意，假如你使用了无主引用引用了一个实例，你必须保证这个实例在引用者的生命周期内不会被销毁，如果被引用的这个实例却先一步over了，你依然访问这个无主引用，那么就会导致崩溃。弱引用不会对其引用的实例保持强引用，也就不会去阻止ARC销毁被引用的实例。

那么，如何去保证无主引用的实例不会被销毁呢？一般来说，引用的这个实例是永远存在的，不可能为nil。所以，我们可以这样区分：两个循环引用的实例所引用的属性都允许为nil的时候，可以使用弱引用来解决；但是如果其中一个属性的值不允许为nil的时候,即只要这个实例存在，就一定会引用着另外一个实例的属性，那么就可以使用无主引用来解决了。

假如出现了这种情况：两个实例所引用的属性都不允许为nil，互相引用该怎么破？假如有两个类，一个是国家Country，一个是城市City。城市肯定属于一个国家，一个国家也肯定会有一个首都城市。这就是互相引用了不为nil的属性。

如下所示：
``` swift
class Country {
    let name: String
    var capitalCity: City!
    init(name: String, capitalName: String) {
        self.name = name
        self.capitalCity = City(name: capitalName, country: self)
    }
}
 
class City {
    let name: String
    unowned let country: Country
    init(name: String, country: Country) {
        self.name = name
        self.country = country
    }
}
```
我们可以看到在Country类里面有个capitalCity属性，在City类里面有个country属性。同时，在County类的构造器里面有对City的初始化赋值到自己的capitalCity属性，在对City初始化的这个构造器里面有一个country参数引用了Country实例(self)。

构造过程没有进行完的时候如何使用这个类的实例呢？这里使用了swift两段式构造的特性，第一段构造，给每一个存储型属性指定一个初始值；第二段构造才会对属性进行进一步定制。Country类里面的capitalCity类型设置为隐式解析可选类型，`City！`表示这个可选类型属性初始化的值为nil，但是不需要进行展开。因为有初始值nil，所以顺利度过第一段构造，在name属性也被构造器赋值后，其实所有属性就已经初始化完成，这个类已经构造完成，所以在第二段构造过程中可以使用`self`作为参数，为capitalCity进行重新赋值。

通过这种方式我们可以通过一条语句同时创建Country类和City类的实例，这样就不会产生循环引用。在这里，我们一边使用了无主引用，一边使用了隐式解析，通过二段式构造的特性巧妙解决了相互引用的问题。
##### 闭包中的循环引用
因为闭包也是引用类型，所以闭包也和一个类一样，如果一个类的某个属性引用了闭包，而这个闭包中又引用了这个类实例，那么就会出现闭包引起的循环引用问题。

swift很人性化的一点就是，假如你在闭包里面是用了这类实例的某个属性或者某个方法，就一定会提示你在前面加上`self`，以提醒你注意循环引用被你一不小心就制造了出来。

swift维护有一个闭包捕获列表，列表的每一项都是由中括号括起来的一对值组成，第一个值是weak或者unowned，另外一个值是对类实例的引用或者是初始化后的变量，比如[unowned self], [weak delegate = self.delegate]等。

**当闭包和它捕获的实例相互引用并且是同时销毁的时候，将闭包里面捕获的引用定义为无主引用。如果被捕获的引用可能会变为nil的时候，将它定义为弱引用。**
