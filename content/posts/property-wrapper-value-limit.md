+++
title = "使用Property Wrapper进行数值约束"
date = "2023-05-16"
tags = [
    "swift",
    "iOS",
    "Property Wrapper",
]
categories = [
    "iOS开发",
]
+++

一直以来对数值大小进行约束都是在textField的代理里面进行的，在输入过程中或者输入结束的代理里面，对输入的值进行判定和修改。最近发现了更优雅的做法，就是使用Property Wrapper封装约束逻辑，在set方法里面对值进行赋值的时候就对值进行修改。
## Property Wrapper使用方法
对于Property Wrapper类型，必须：
1. 使用@propertyWrapper进行定义
2. 必须有wrappedValue属性
``` swift
@propertyWrapper
struct Wrapper<T> {
   var wrappedValue: T
}
```
这时候我们就可以通过`@Wrapper`修饰一个属性，这个属性绑定的就是wrappedValue。
``` swift
@Wrapper var x = 0
```
## 构建
我们要实现这样的功能：如果一个数值在一个区间内，就返回这个数值本身，如果小于这个区间，就返回这个区间的最小值，如果大于这个区间，就返回这个区间的最大值。
这个思路可以通过`Swift.max(Swift.min(self, maxValue), minValue)`来实现。
可以比较的类型都是Comparable类型的，所以我们创建一个协议并添加默认实现：
``` swift
protocol FPValueLimit: Comparable {
    func limit(from minValue: Self, to maxValue: Self) -> Self
}

extension FPValueLimit {
    func limit(from minValue: Self, to maxValue: Self) -> Self {
        Swift.max(Swift.min(self, maxValue), minValue)
    }
}
```
并为常用的数值类型添加此协议：
``` swift
extension Int: FPValueLimit {}
extension Int8: FPValueLimit {}
extension Int16: FPValueLimit {}
extension Int32: FPValueLimit {}
extension Int64: FPValueLimit {}
extension Float: FPValueLimit {}
extension Double: FPValueLimit {}
```
这时候，其实这些类型已经可以通过`limit(from: minValue, to: maxValue)`方法实现了数值约束了。但是这不是最终目的。
添加Property Wrapper类型`ValueLimit`:
``` swift
/// 限制一个值在一个固定的区间
/// 如果超过这个区间，则取这个值的最大/最小值
@propertyWrapper
struct ValueLimit<T> where T: FPValueLimit {
    private var value: T?
    var minValue: T
    var maxValue: T
    var wrappedValue: T? {
        get { value }
        set {
            guard let aValue = newValue else { return }
            value = aValue.limit(from: minValue, to: maxValue)
        }
    }

    init(min: T, max:T) {
        self.minValue = min
        self.maxValue = max
    }
}
```
通过在wrappedValue的set方法，我们可以为添加这个前缀的属性在set的时候添加数值约束。
比如在填单的时候填写重量：
``` swift
@ValueLimit(min: FPRule.minWeight, max:FPRule.maxWeight)
var weight: Int? {
    didSet {
        guard let weight = weight, weight > 0 else { return }
        weightTextField.stringValue = weight.format.toWeightDisplay()
    }
}
```
在didSet方法执行的时候，赋值的weight已经经过了数值约束，在最大和最小值之中了。在UI控件textField还没有碰到数值之前，数值就已经做好了一切准备。
有时候数值限制还需要添加警示信息：
``` swift
/// 如果一个值超出固定的区间，弹出警告信息
@propertyWrapper
struct ValueLimitAlert<T> where T: Comparable {
    private var value: T?
    var minValue: T
    var maxValue: T
    var alertMessage: String
    var wrappedValue: T? {
        get { value }
        set {
            guard let newValue = newValue else { return }
            if newValue > maxValue || newValue < minValue {
                FPHUD.shared.showToast(text: alertMessage)
            }
        }
    }

    init(min: T, max:T, alert: String) {
        self.minValue = min
        self.maxValue = max
        self.alertMessage = alert
    }
}
```
这样在超出范围的时候，就会弹出对应的警示信息了。
