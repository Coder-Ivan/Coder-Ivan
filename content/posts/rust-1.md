+++
title = "Rust学习语法篇(一)"
date = "2023-09-01"
tags = [
    "Rust",
    "基础",
]
categories = [
    "Rust学习之路",
]
+++

# 循环
带标签的循环：
``` rust
fn main() {
    let mut count = 0;
    'counting_up: loop {
        println!("count = {}", count);
        let mut remaining = 10;
        loop {
            println!("remaining = {}", remaining);
            if remaining == 9 {
                break;
            }
            if count == 2 {
                break 'counting_up;
            }
            remaining -= 1;
        }
        count += 1;
    }
    println!("End count = {}", count);
}
```
loop的一个用例是重试可能会失败的操作，比如检查线程是否完成了任务。因此，需要将操作的结果从返回中传递给其他代码。为此，可以在用于停止循环的`break`表达式添加想要返回的值。
``` rust
fn main() {
    let mut counter = 0;
    let result = loop {
        counter += 1;
        if counter == 10 {
            break counter * 2;
        }
    };
    println!("The result is {}", result);
}
```
while循环：
``` rust
fn main() {
    let mut number = 3;
    while number != 0 {
        println!("{}!", number);
        number -= 1;
    }
    println!("LIFTOFF!!!");
}

```
for循环：
``` rust
fn main() {
    let a = [10, 20, 30, 40, 50];
    for element in a {
        println!("the value is: {}", element);
    }
	// 反转区间
	for number in (1..4).rev() {
        println!("{}!", number);
    }
}
```
# 所有权
Rust永远不会创建数据的”深拷贝“。
默认情况下，引用是不可变的，我们无法通过引用修改内容。加上mut后的可变引用可以修改内容。
在同一时间，只能有一个对某一特定数据的可变引用，尝试创建2个可变引用的代码会失败。这样的限制在编译的时候就避免了数据竞争。
> 发生数据竞争的条件：
> 1. 两个或更多指针同时访问同一数据。
> 2. 至少有一个指针呗用来写数据。
> 3. 没有同步数据访问的机制。

同样，也不能在拥有对一个数据的引用的时候，再加一个可变引用。

总结：
1. 在任意给定的时间，要么只能有一个可变引用，要么只能有多个不可变引用。
2. 引用必须总是有效的。