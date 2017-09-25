---
layout: post
categories: rust
---
作为Rust新人，听说Rust最特殊的就是它的borrow system 和 lifetime；在编译的时候会
遇到什么 'can't borrow error','use after moved'这类的错误，下面来记录几个例子
<!-- more -->

本文目录：

* Borrow & Move
* Copy

## borrow & move

`borrow` 和 `move` 都是 let 绑定语句实现的

`let mut k12 = klaatu;`是move，

当然，这和是否mut 没有关系， `let k12 = klaatu`也是move

borrow和 move的区别在于 klaatu的类型，如果 klaatu是& 引用 则let 语句是 borrow，否则就是一个move，下面的例子

```rust
struct Alien{
        planet: String,
        n_ten: u32
}
fn main(){

        let mut klaatu = Alien{planet: "Venus".to_string(), n_ten:15};

        klaatu.n_ten = 15;
        {
        let mut k12 = klaatu;
        k12.n_ten = 11;
        //println!("{}", k12.planet);
        }
        //println!("{} - {}", klaatu.planet, klaatu.n_ten); // compile error occur, klaatu has been moved to k12


}
```

这里 `let mut k12 = klaatu;`就是  move，在 k12作用域结束的地方'}'，Alien被释放了，所以klaatu也不能用了

而下面的borrow，则不会有这样的问题:

```rust
fn main(){

        let mut klaatu = Alien{planet: "Venus".to_string(), n_ten:15};

        klaatu.n_ten = 15;
        {
        let k12 = &mut klaatu;
        k12.n_ten = 11;
        //println!("{}", k12.planet);
        }
        println!("{} - {}", klaatu.planet, klaatu.n_ten);


}
```

## Copy
copy适用于简单的变量，一般是 primary type 保存在堆栈中的类型

```rust
fn main() {
    let mut x = 5;
    let mut y = x;

    y += 1;

    println!("x is: {}", x);
    println!("y is: {}", y);
}
```
复杂的结构体呢？如果直接在Alien上实现 Copy

```rust
#[derive(Copy)]
struct Alien{
        planet: String,
        n_ten: u32
}
```
编译错误

```
9  | #[derive(Copy)]
   |          ^^^^
10 | struct Alien{
11 |planet: String,
   |  -------------- this field does not implement `Copy`
```
编译器 提示：String类型不支持 Copy，ok，将其变成 u32

```rust
#[derive(Copy,Clone)]
struct Alien{
        planet: u32,
        n_ten: u32
}
fn main(){

        let mut klaatu = Alien{planet: 1, n_ten:15};

        klaatu.n_ten = 15;
        {
        let mut k12 = klaatu;
        k12.n_ten = 11;
        println!("copyed value:{} original value: {}", k12.n_ten, klaatu.n_ten);
        }
        println!("after k12 been freed: {} - {}", klaatu.planet, klaatu.n_ten);
}
```
可以看到   `let mut k12 = klaatu;` 触发的是Copy而非之前例子中的moved，从输出结果就可以看得出来

```
copyed value:11 original value: 15
after k12 been freed: 1 - 15
``` 
