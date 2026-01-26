---
title: Rust所有权
description: Rust所有权
pubDate: 01 11 2025
image: /image/image1.jpg
categories:
  - Rust
tags:
  - Rust
---

rust为了防止出现未定义的行为引入了所有权机制，极大的保障了内存的安全


# 概述
## 未定义行为
未定义行为：当执行一段代码时，结果不可预测且未被语言指定的情况

比如数组越界访问，访问释放的堆内存，释放两次堆内存等

## Rust的目标
基础目标：确保程序永远不会有未定义的行为
次要目标：在编译的时候而不是运行的时候防止未定义行为


# 所有权
## 局部变量在stack中
``` rust
fn main(){
	let n = 5;    #L1
	let y = plus_one(n);    #L3
	println!("The value of y is: {y}");
}

fn plus_one(x: i32) -> i32{
	x + 1        #L2
}

```
![!\[\[Pasted image 20260126105539.png\]\]](<../../../public/image/Rust1/Pasted image 20260126105539.png>)
## Box存活在Heap中

如果不使用Box,也就是在Stack上分配，会浪费很多空间
``` rust
fn main(){
	let a = [0;1_000_000];  #L1
	let b = a;     #L2
}
```
![alt text](<../../../public/image/Rust1/Pasted image 20260126105752.png>)

如果使用Box在Heap上分配，Stack上存放的只是指针，指向同一个Heap地址，其他语言一般通过这两个指针都可以访问这个Heap内存，但在Rust中，由于所有权的概念，同时最多只能有一个指针能访问指向的Heap内存。

一开始我们说a具有所有权，到后来let b = a; 我们说，a把所有权转移给了b,此时只能通过b来访问Heap,如果企图通过a来访问，rust编辑器会报错。

``` rust
let a = Box::new([0;1_000_000]);  #L1
let b = a;   #L2
```

![alt text](<../../../public/image/Rust1/Pasted image 20260126110422.png>)


需要注意的是这种情况
``` rust
let a = Box::new(15);
let b = a;
let c = Box::new(15);
```

![alt text](<../../../public/image/Rust1/Pasted image 20260126110610.png>)

# Rust内存管理策略
## Rust不允许手动内存管理

Stack Frame由Rust自动管理：当调用一个函数时，Rust为调用的函数分配一个Stack Frame。当调用结束时，Rust释放该Stack Frame

假设有一段代码,这段代码会由于手动释放了b所在的Stack内存，然后rust在函数调用结束后又自动释放了b所在的Stack内存，这就出现了Double Free,为了防止这样的未定义行为，Rust不允许手动内存管理

``` rust
let b = Box::new([0;100]);
free(b);
assert!(b[0] == 0);
```
![[Pasted image 20260126113208.png]]

## Box的真正拥有者来管理对应Box内存的释放
Rust会自动释放Box的Heap内存

Box内存释放原则：如果一个变量拥有（所有）一个Box,当Rust释放该变量的Stack Frame时，Rust会释放该Box的Heap内存。

而使用Box的集合有：Vec,String, &HashMap等
``` rust
fn main(){
	let first = String::from("Ferris");  #L1
	let full = add_suffix(first); #L4
}

fn add_suffix(mut name: String) -> String{
	#L2
	name.push_str("Jr.");  #L3
	name
}
```

![alt text](<../../../public/image/Rust1/Pasted image 20260126114156.png>)
所有权的转移：first  ->   name   ->   full

## 移动
移动：如果变量x将Heap内存的所有权给了另一个变量y,也就是发生了所有权的转移，这就叫移动，所有权转移后x将不再能访问原来的Heap内存。

## 克隆
而与移动相对应的就是克隆

避免数据移动的一种方法是使用.clone()方法进行克隆
``` rust
fn main(){
	let first = String::from("Ferris");
	let fist_clone = first.clone(); #L1
	let full = add_suffix(first_clone); #L2
	println!("{full},originally {first}"); 
}

fn add_suffix(mut name: String) -> String{
	name.push_str("Jr.");
	name
}
```

![alt text](<../../../public/image/Rust1/Pasted image 20260126115343.png>)