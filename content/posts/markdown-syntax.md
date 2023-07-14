+++
authors = ["陌浅"]
title = "Property Wrapper优化UserDefaults的使用"
date = "2023-03-11"
description = "简化Userdefaults使用上的不方便之处。"
tags = [
    "swift",
    "iOS",
    "Property Wrapper",
]
categories = [
    "iOS开发",
]
+++

## 目的
简化保存UserDefaults的写法，通过正常属性赋值取值的方式进行UserDefaults的存取。
比如，正常，保存和读取UserDefaults是这样的：
``` swift
UserDefaults.standard.setValue("aaa", forKey: "stringvalue")
UserDefaults.standard.value(forKey: "stringvalue") as? String
```
如果我想要这样做呢？
``` swift
// 保存到UserDefaults中
UserDefaults.stringValue = "aaa"
// 从UserDefaults中读取
print(UserDefaults.stringValue)
```
## 定义要存取的值
属性保存为UserDefaults的类属性，如：
``` swift
extension UserDefaults {
    static var boolValue: Bool?
    static var stringValue: String?
    static var dictValue: [String: String]?
    static var dateValue: Date?
}
```
这样的赋值方式只是保存在了内存里面，接下来要使用Property Wrapper在进行存取的时候额外做点事情，把值保存和读取在UserDefaults里面。
## 定义Property Wrapper
``` swift
// 最基本的结构
@propertyWrapper
struct UserDefault<Value> {
    var wrappedValue: Value 
}
```
现在为wrappedValue设置get和set方法，在这里面进行UserDefaults的存取。
因为取值的时候并不能保证一定能取到，所以要设置一个默认值，在无法取到值的时候使用默认值替代。
``` swift
@propertyWrapper
struct UserDefault<Value> {
    let key: String
    // 默认值
    let defaultValue: Value

    var wrappedValue: Value {
        get {
            // 在无法从UserDefaults里面取到值的时候，就取默认值
            return UserDefaults.standard.object(forKey: key) as? Value ?? defaultValue
        }
        set {
            UserDefaults.standard.set(newValue, forKey: key)
        }
    }
}
```
为特定属性添加存取特性
``` swift
extension UserDefaults {
    @UserDefault(key: "boolvalue", defaultValue: false)
    static var boolValue: Bool
    @UserDefault(key: "stingvalue", defaultValue: "default")
    static var stringValue: String
    @UserDefault(key: "dictvalue", defaultValue: [String: String]())
    static var dictValue: [String: String]
    @UserDefault(key: "datevalue", defaultValue: Date())
    static var dateValue: Date
}
```
之所以把可选值的"?"去掉，是因为UserDefaults.standard.object(forKey: key) as? Value当Value为可选值的时候，即使前面的值为nil，也不会走后面的?? defaultValue了。
## 添加对nil的支持
如果想要在设置一个属性为nil的时候，不需要设置默认值，那就要添加一个新的初始化方法，在Value为可选值的时候，可以使用该初始化方法。
判断Value是否是可选值可以判断该Value是否遵从ExpressibleByNilLiteral协议。
``` swift
extension UserDefault where Value: ExpressibleByNilLiteral {

    init(key: String) {
        self.init(key: key, defaultValue: nil)
    }
}
```
当Value值为可选值的时候，就可以不传默认值了。
当传nil的时候，删除UserDefaults中的key，可以在set中判断newValue是否为nil。我们先设置一个protocol：
``` swift
protocol AnyOptional {
    var isNil: Bool { get }
}

extension Optional: AnyOptional {
    var isNil: Bool { self == nil }
}
```
此时就可以用AnyOptional来判断是否是nil了：
``` swift
@propertyWrapper
struct UserDefault<Value> {
    let key: String
    let defaultValue: Value

    var wrappedValue: Value {
        get {
            return UserDefaults.standard.object(forKey: key) as? Value ?? defaultValue
        }
        set {
            if let optionalValue = newValue as? AnyOptional, optionalValue.isNil {
                UserDefaults.standard.removeObject(forKey: key)
            } else {
                UserDefaults.standard.set(newValue, forKey: key)
            }
        }
    }
}
```
此时，我们的属性设置可以为可选值：
``` swift
extension UserDefaults {
    @UserDefault(key: "boolvalue", defaultValue: false)
    static var boolValue: Bool
    @UserDefault(key: "stingvalue", defaultValue: "default")
    static var stringValue: String
    // 不需要传默认值
    @UserDefault(key: "dictvalue")
    static var dictValue: [String: String]?
    @UserDefault(key: "datevalue", defaultValue: Date())
    static var dateValue: Date
}
```
## 添加对自定义模型的支持
UserDefaults不支持class、struct等的类型的保存，需要先进行序列化/反序列化才能进行正常读取。所以，针对这样的情况，对这些非基本类型的保存，就使用另外的Property Wrapper进行单独处理。
比如，一个Person类的保存：
``` swift
extension UserDefaults {
    @UserDefault(key: "personvalue", defaultValue: Person(name: nil, age: nil))
    static var personValue: Person
}

class Person {
    var name: String?
    var age: Int?

    init(name: String?, age: Int?) {
        self.name = name
        self.age = age
    }
}

let person: Person = Person(name: "Ivan", age: 30)

UserDefaults.personValue = person  // Wrong！！！
```
以上操作会导致报错。所以我们需要进行额外的操作：
``` swift
@propertyWrapper
struct UserDefaultCodable<Value: Codable> {
    let key: String
    let defaultValue: Value
    var container: UserDefaults = .standard

    var wrappedValue: Value {
        get {
            if let data = container.object(forKey: key) as? Data,
               let user = try? JSONDecoder().decode(Value.self, from: data) {
                return user
            }
            return  defaultValue
        }
        set {
            if let encoded = try? JSONEncoder().encode(newValue) {
                container.set(encoded, forKey: key)
            }
        }
    }
}
```
使该类支持Codable：
``` swift
class Person: Codable {
    var name: String?
    var age: Int?

    init(name: String?, age: Int?) {
        self.name = name
        self.age = age
    }
}
```
这时候，就可以对非基础类型进行正常的存取了。
