---
title: Rust引用和借用
description: Rust引用和借用
pubDate: 01 27 2026
image: /image/image2.jpg
categories:
  - Rust
tags:
  - Rust
---

# 概述

## 函数传递常见问题
在上篇文章我们说到，凡是用到Box的变量中赋值的时候会出现所有权的转移，称为移动

但是每次这样移动未免太过麻烦，并且在函数传递的时候更不方便

比如下面这个例子

format!是重新创建一个格式化后的String类型变量
``` rust
fn main(){
	let m1 = String::from("Hello");
	let m2 = Srting::from("world");
	greet(m1,m2);   #L2
	let s = format!("{} {}",m1,m2);  #L3  //Error: m1 and m2 are moved
}

fn greet(g1: Srting,g2: String){
	println!("{} {}!",g1,g2);  #L1
}
```

![alt text](<../../../public/image/Rust2/Pasted image 20260126173319.png>)
这个例子是在函数传递的时候发生了移动，导致原来的变量无法访问，如果访问编译器会报错

## 解决方式

### 第一种（返回值的方式）
let (m1_again,m2_again) = greet(m1,m2);是把greet的两个返回值分别返回给m1_again,m2_again

把所有权再移动回main函数中的变量
所有权转移： m1 -> g1 -> m1_again
m2 -> g2 -> m2_again
``` rust
fn main(){
	let m1 = String::from("Hello");
	let m2 = String::from("world"); #L1
	let (m1_again,m2_again) = greet(m1,m2);
	let s = format!("{} {}",m1_again,m2_again); #L2
}

fn greet(g1: String,g2: Srting) -> (String, String){
	println!("{} {}", g1, g2);
	(g1, g2)
}
```

![alt text](<../../../public/image/Rust2/Pasted image 20260126174033.png>)

### 第二种（引用的方式）

由第一种可以看出，返回值的方式同样复杂，为了解决这种不方便，rust有一种类型叫做引用


# 引用
引用：引用是没有 “所有权” 的指针

也就是使用引用类型的变量不会移动所有权
``` rust
fn main(){

	let m1 = String::from("Hello");
	let m1 = String::from("world"); #L1
	greet(m1,m2); #L3
	let s = format!("{} {}", m1 m2);
}

fn greet(){g1: &Srting,g2: &String}{
	#L2
	println!("{} {}", g1, g2);
}
```

![alt text](<../../../public/image/Rust2/Pasted image 20260126174926.png>)

## 解引用指针以访问数据

解引用运算符 ： *

这是我们常理解的正常解引用，也就是显示解引用
``` rust
let mut x: Box<i32> = Box::new(1);
let a: i32 = *x;
*x += 1;

let r1: &Box<i32> = &x;
let b: i32 = **r1;

let r2: &i32 = &*x;
let c: i32 = *r2;  #L1
```

![alt text](<../../../public/image/Rust2/Pasted image 20260126180353.png>)

而在Rust中，很多情况都是隐式的解引用

``` rust
let x: Box<i32> = Box::new(-1);
let x_abs1 = i32::abs(*x);   //显式解引用
let x_abs2 = x.abs();  //隐式解引用
assert_eq!(x_abs1,x_abs2);

let r: &Box<i32> = &x;
let r_abs1 = i32::abs(**r); //显式解引用,两次
let r_abs2 = r.abs()  //隐式解引用，两次
assert_eq!(r_abs1,r_abs2)

let s = String::from("Hello");
let s_len1 = str::len(&s); //显式解引用
let s_len2 = s.len(); //隐式解引用
assert_eq!(s_len1,s_len2)
```

我们不难发现，Rust中有一个规律
- 一般   类型::方法(变量)   这种需要显式解引用（或显式引用）
- 而      变量.方法()   这种一般是隐式解引用（或隐式引用），并且支持多层


## 引用带来的问题

别名：通过不同的变量访问同一数据
别名数据：可被多个变量访问的一块数据

1. 一个指针变量，如果把这个变量的值传给另一个指针变量叫创建了他的一个别名（也就是所谓的传递地址）
2. 一般来说引用一个变量也“相当于”创建了他的一个别名（但Rust中引用与别名不完全相同）

试想如果我释放了引用数据，然后再通过原变量访问，就出现了UAF

或者我通过引用改变了Heap内存的值，而原变量却不知道，可能会出现与预期不相符的结果

或引用和原变量同时修改，可能会导致数据竞争，产生漏洞


## Rust为解决引用带来的不安全性的方案

Rust为解决这些可能产生未定义行为的代码，引入了一个原则：别名和可变性不可以同时存在

先看下面这个例子

补充：
Vec的底层结构
- `ptr`：指向堆上分配的内存起始地址的指针
- `len`：当前已存储的元素数量
- `cap`：分配的内存总共能容纳的元素数量
push 的逻辑：
- 如果 `len < cap`：直接在`ptr + len`的内存位置写入新元素，`len += 1`，**指针和内存区域完全不变**；
- 如果 `len >= cap`：触发扩容（reallocation）—— 分配一块更大的内存（通常是原容量的 2 倍，小容量时可能按固定值增长），把原有元素拷贝到新内存，释放旧内存，然后在新内存尾部写入新元素，此时`ptr`指向新内存地址。
let mut v： Vec\<i32> = vec!\[1, 2, 3];
是初始创建了包含3个元素1,2,3,容量为3的一个vec


``` rust
let mut v: Vec<i32> = vec![1, 2, 3];
let num: &i32 = &v[2]; #L1
v.push(4); #L2
println!("Third element is {}", *num); #L3 error
```

![alt text](<../../../public/image/Rust2/Pasted image 20260126183346.png>)
这个例子由于push之后v的容量超过了原来的容量，需要另外开辟一个空间并拷贝且增加容量，那么num这个别名就访问到了非法内存，在Rust中编译器会报错

因为这违反了rust的原则：别名与可变不可以同时存在
在这个例子中也就是，在num存在的时候，v不能调用push方法，v没有写的权限W

同时为了安全考虑，Rust的引用被设计之初就不是一种等同于别名的存在。

## 引用不等于别名
一个很好的解释说，Rust中的引用是临时创建的别名

Box(有所有权的指针)：不能别名（上文中的第一条），但可被引用（上文中的第二条），若将一个Box变量赋值给另一个Box变量（把地址传给另一个变量，别名操作），只会发生移动，也就是所有权的转移

不能让多个Box同时拥有一块数据

引用(无所有权的指针)：旨在临时创建别名，把Box的地址传给一个引用变量，这个引用可以间接访问到Box指向的Heap内存

由于println！会自动解引用
``` rust
fn main(){
	let x = Box::new(1);
	let y = x;
	println!("y: {}",y);
	
	let r1 = &y;
	let r2 = &y;
	println!("r1: {r1},r2: {r2}");
  }
```

# Rust中的权限
## Rust通过借用检测器确保引用的安全性

变量（数据所在地址）对其数据（地址的内容）有三种权限：
	读（R）：数据可以被复制到另一个位置
	写（W）：数据可以被修改
	拥有（O）：数据可以被移动或释放

这些权限在运行时并不存在，仅在编译器内部存在

默认情况下，变量对其数据具有读/拥有权限（RO）。
如果一个变量被注解为let mut,那么他还具有写权限（W）。

关键：引用可以临时移除这些权限，所以引用有时也被叫做借用，把权限暂时借用走了


补充：对于数组或者vec来说，引用其中的一个元素，这个元素地址的权限以及v都会受到影响，也就是&v\[2]的权限和v的权限都变了

对于不可变引用（共享引用）：引用只会让原变量失去W和O权限，也就是修改和释放权限

比如这里的&v\[2]他就是一个不可变引用

![alt text](<../../../public/image/Rust2/Pasted image 20260126192331.png>)

如何理解变量（数据所在地址）对其数据（地址的内容）这个限定
x_ref（一个地址）对地址的内容具有修改的权限而\*x_ref（一个数0）则没有
![alt text](<../../../public/image/Rust2/Pasted image 20260126195039.png>)

## 权限是定义在位置上的
权限是定义在位置上的，不仅仅是单个变量
位置上任何可以防在赋值语句左侧的东西

比如*((\*a)\[0].1),他就是一个位置，他具有权限这个定义

## 为什么失去特定权限

因为有些权限是互斥的

怎么理解

下面这个例子，当num有读权限R的时候，v就必然不能有O权限，因为num在访问v指向的Heap内存的时候，要保证v不能释放所指向的Heap内存，否则会出现未定义的行为

同时为了防止数据竞争，当num有读权限R的时候，v就必然不能有W权限，防止num读的时候，v恰好修改Heap数据而造成的数据竞争，出现未定义的行为

![alt text](<../../../public/image/Rust2/Pasted image 20260126195850.png>)


比如在num使用的时候,就必然不能修改v指向的内存，下面这个例子就会报错，在num还存在的时候（还使用的时候）企图通过v来修改Heap内存

![alt text](<../../../public/image/Rust2/Pasted image 20260126200513.png>)

## 不可变引用和可变引用

以上都是说的不可变引用

不可变引用可以存在多个，因为他们不违反：多个Box不能同时指向同一个Heap内存

不可变引用（共享引用）：只读的

如果只有不可变引用，我们要想修改一个Box类型指向的Heap内存，只能通过移动所有权的方式来用另一个变量修改

因此我们还需要有可变引用

---

可变引用（独占引用）：在不移动数据的情况下，临时提供可变访问

**同一作用域，特定数据只能有一个可变的引用**：

可变引用：可变引用提供对数据 “唯一的”(同一个作用域) 且 “非拥有的”访问
按道理来说可变引用也可以存在多个，因为他们不违反：多个Box不能同时指向同一个Heap内存，

但是可变引用由于另一个原则，只能是唯一的

我们说过权限是互斥的，当一个可变引用在修改Heap内存的时候，其他变量，不管是原变量还是另一个原变量的引用都不能再修改Heap内存了，因为如果是同时进行的话就会出现数据竞争

也就是说，访问同一个Heap内存，只能有一个变量的权限有W,这也对应着，对于Box来说，别名和不可变性不能同时存在 

同时可变引用也会让原变量失去R权限和O权限，O权限和不可变引用一样，而失去R权限还是为了防止数据竞争，在使用可变引用来修改Heap内存的时候，不能通过原变量来访问Heap内存，因为如果可变引用修改的时候原变量来访问，就会出现数据竞争，出现未定义对行为

---

下面是可变引用的例子
![alt text](<../../../public/image/Rust2/Pasted image 20260126202054.png>)

很多时候，大括号可以帮我们解决一些编译不通过的问题，通过手动限制变量的作用域：

``` rust
let mut s = String::from("hello"); 
 {  
   let r1 = &mut s; 
 } // r1 在这里离开了作用域，所以我们完全可以创建一个新的引用 
  let r2 = &mut s;
```

## 不可变引用和可变引用对原变量权限的影响

由上面的例子，我们大致可以推断出这样一个原则

不可变引用可以存在多个，他会让原变量失去WO权限，而引用本身获得R权限

可变引用只能存在一个，他会让原变量失去RWO权限，而引用本身获得RW权限

由以上还可以推断出
- 引用不能用来释放Heap,只有权限还给原变量的时候，才能通过原变量释放Heap
- 原变量和引用一个有R另一个就没有W,一个有W另一个就不能有R


## 可变引用和不可变引用不能同时存在

下面的代码会导致一个错误：
``` rust
let mut s = String::from("hello");

let r1 = &s; // 没问题
let r2 = &s; // 没问题
let r3 = &mut s; // 大问题

println!("{}, {}, and {}", r1, r2, r3);
```

错误如下：

``` bash
error[E0502]: cannot borrow `s` as mutable because it is also borrowed as immutable
        // 无法借用可变 `s` 因为它已经被借用了不可变
 --> src/main.rs:6:14
  |
4 |     let r1 = &s; // 没问题
  |              -- immutable borrow occurs here 不可变借用发生在这里
5 |     let r2 = &s; // 没问题
6 |     let r3 = &mut s; // 大问题
  |              ^^^^^^ mutable borrow occurs here 可变借用发生在这里
7 |
8 |     println!("{}, {}, and {}", r1, r2, r3);
  |                                -- immutable borrow later used here 不可变借用在这里使用

```

其实这个也很好理解，正在借用不可变引用的用户，肯定不希望他借用的东西，被另外一个人莫名其妙改变了。多个不可变借用被允许是因为没有人会去试图修改数据，每个人都只读这一份数据而不做修改，因此不用担心数据被污染。

> 注意，引用 `r1`,`r2`,`r3` 的作用域从创建开始，一直持续到它最后一次使用的地方 `println!(....)`，这个跟变量的作用域有所不同，变量的作用域从创建持续到某一个花括号 `}`

Rust 的编译器一直在优化，早期的时候，引用的作用域跟变量作用域是一致的，这对日常使用带来了很大的困扰，你必须非常小心的去安排可变、不可变变量的借用，免得无法通过编译，例如以下代码：

``` rust
fn main() {
   let mut s = String::from("hello");

    let r1 = &s;
    let r2 = &s;
    println!("{} and {}", r1, r2);
    // 新编译器中，r1,r2作用域在这里结束

    let r3 = &mut s;
    println!("{}", r3);
} // 老编译器中，r1、r2、r3作用域在这里结束
  // 新编译器中，r3作用域在这里结束
```


在老版本的编译器中（Rust 1.31 前），将会报错，因为 `r1` 和 `r2` 的作用域在花括号 `}` 处结束，那么 `r3` 的借用就会触发 **无法同时借用可变和不可变** 的规则。

但是在新的编译器中，该代码将顺利通过，因为 **引用作用域的结束位置从花括号变成最后一次使用的位置**，因此 `r1` 借用和 `r2` 借用在 `println!` 后，就结束了，此时 `r3` 可以顺利借用到可变引用。


## 可变引用临时降级为只读引用

还是同样的道理，如果先创建可变引用再创建不可变引用的话
可变引用会降级

可以这么理解，要创建不可变引用，首先要保证创建的是不可变引用，所以只有R权限，而他的R和可变引用的W互斥，因为如果同时存在可能能发生数据竞争，所以可变引用被临时降级，当不可变引用灭亡时，可变引用重新获得W权限

![alt text](<../../../public/image/Rust2/Pasted image 20260126211316.png>)

曾经的疑惑
为什么下面一串代码会报错
``` rust
fn main() {
	let mut v: Vec<i32> = vec![1,2,3];
	let num : &mut i32 = &mut v[2];
	let num2 : & i32 = &v[2];
	println!("{} {}", num, num2);
}
```

其实就是Rust借用检测器关注的是借用的来源，如果这么写就相当于有一个可变引用和一个不可变引用在同一作用域，并且他们的根为v\[2]

而原来的写法相当于一个可变引用根为v\[2],一个不可变引用根为num，这叫做派生借用，也叫子借用

![alt text](<../../../public/image/Rust2/Pasted image 20260126214801.png>)
![alt text](<../../../public/image/Rust2/Pasted image 20260126214952.png>)
![alt text](<../../../public/image/Rust2/Pasted image 20260126215005.png>)

# 悬垂引用问题
悬垂引用也叫做悬垂指针，意思为指针指向某个值后，这个值被释放掉了，而指针仍然存在，其指向的内存可能不存在任何值或已被其它变量重新使用。在 Rust 中编译器可以确保引用永远也不会变成悬垂状态：当你获取数据的引用后，编译器可以确保数据不会在引用结束前被释放，要想释放数据，必须先停止其引用的使用。

让我们尝试创建一个悬垂引用，Rust 会抛出一个编译时错误：
`
``` rust
fn main() {
    let reference_to_nothing = dangle();
}

fn dangle() -> &String {
    let s = String::from("hello");

    &s
}
```

这里是错误：

``` bash
error[E0106]: missing lifetime specifier
 --> src/main.rs:5:16
  |
5 | fn dangle() -> &String {
  |                ^ expected named lifetime parameter
  |
  = help: this function's return type contains a borrowed value, but there is no value for it to be borrowed from
help: consider using the `'static` lifetime
  |
5 | fn dangle() -> &'static String {
  |                ~~~~~~~~

```

错误信息引用了一个我们还未介绍的功能：[生命周期(lifetimes)](https://course.rs/basic/lifetime.html)。不过，即使你不理解生命周期，也可以通过错误信息知道这段代码错误的关键信息：

``` bash
fn dangle() -> &String { // dangle 返回一个字符串的引用

    let s = String::from("hello"); // s 是一个新字符串

    &s // 返回字符串 s 的引用
} // 这里 s 离开作用域并被丢弃。其内存被释放。
  // 危险！
```

仔细看看 `dangle` 代码的每一步到底发生了什么：

``` rust
fn dangle() -> &String { // dangle 返回一个字符串的引用

    let s = String::from("hello"); // s 是一个新字符串

    &s // 返回字符串 s 的引用
} // 这里 s 离开作用域并被丢弃。其内存被释放。
  // 危险！
```

因为 `s` 是在 `dangle` 函数内创建的，当 `dangle` 的代码执行完毕后，`s` 将被释放，但是此时我们又尝试去返回它的引用。这意味着这个引用会指向一个无效的 `String`，这可不对！

其中一个很好的解决方法是直接返回 `String`：

``` rust
fn no_dangle() -> String {
    let s = String::from("hello");

    s
}

```

这样就没有任何错误了，最终 `String` 的 **所有权被转移给外面的调用者**。

# 借用规则总结

总的来说，借用规则如下：

- 同一时刻，你只能拥有要么一个可变引用，要么任意多个不可变引用，可变引用和不可变引用不能同时存在
- 引用必须总是有效的


# F权限流动权限
最后再补充一下

流动权限F：在表达式使用输入引用或返回输出引用时需要

F权限在函数体内不会发生变化

如果一个引用被允许在特定表达式中使用（即流动），那么它就具有F权限

我的理解是，流动权限是决定你能不能在不同表达式之间“流动”的一种权限，如果可以那么就具有流动权限，如果不可以就不具有流动权限

第一行的引用是输入引用（也就是函数某一参数的引用）需要流动权限，而他也具有流动权限
第二行返回输出引用，也需要流动权限，而他也具有流动权限
![alt text](<../../../public/image/Rust2/Pasted image 20260126222118.png>)


这个例子就会报错，因为Rust检查输出引用的返回时只看函数的签名，而这个例子中Rust只看这个函数签名，不知道&String返回的是引用自谁的引用，可能是strings的，也可能是default的，由于这种不确定性，可能会导致未定义的行为，比如下面的例子可能造成UAF,drop为释放Box指向的Heap内存

可见&string\[0]和default的输出引用都不具有流动权限

![alt text](<../../../public/image/Rust2/Pasted image 20260126222357.png>)
![alt text](<../../../public/image/Rust2/Pasted image 20260126222427.png>)


这个例子就是我们上面说的悬挂引用的例子
s_ref输出引用显然不具有流动权限
![alt text](<../../../public/image/Rust2/Pasted image 20260126223009.png>)