# Rust Sized Trait与类型大小

这篇文章来自于Github的一篇关于[Rust size](https://github.com/pretzelhammer/rust-blog/blob/master/posts/sizedness-in-rust.md)的一些介绍，本文内容基本上是这文章的总结。可以话尽量看原文。
<!--more-->
# 介绍

`sideness`在Rust中是一个比较底层的概念，但是又比较重要，而且与许多Rust特性有关系。通常我们能在编译器的错误信息中看到`x doesn't have size known at compile time`，那么到底什么是`sized type`；什么是`unsized type`以及什么是`zero-sized type`。他们应该怎么使用，和一些优缺点。

这有一张表是以一个总的视角来看Rust的type。

| Phrase                                        | Shorthand for                                                |
| --------------------------------------------- | ------------------------------------------------------------ |
| sizedness                                     | property of being sized or unsized                           |
| sized type                                    | type with a known size at compile time                       |
| 1) unsized type *or* 2) DST                   | dynamically-sized type, i.e. size not known at compile time  |
| ?sized type                                   | type that may or may not be sized                            |
| unsized coercion                              | coercing a sized type into an unsized type                   |
| ZST                                           | zero-sized type, i.e. instances of the type are 0 bytes in size |
| width                                         | single unit of measurement of pointer width                  |
| 1) thin pointer *or*  2) single-width pointer | pointer that is *1 width*                                    |
| 1) fat pointer *or*  2) double-width pointer  | pointer that is *2 widths*                                   |
| 1) pointer *or* 2) reference                  | some pointer of some width, width will be clarified by context |
| slice                                         | double-width pointer to a dynamically sized view into some array |

## sizedness

到底什么叫sizedness呢，在Rust中，如果一个类型的大小能在编译器能确定下来，那就是一个sized type。确定一个类型的大小是非常重要的，因为这关系到在stack上申请多大内存，一个确定大小的类型能够pass by value and ref。如果一个type是 unsized type  or DST(dynamic Sized Type)那么它就不能在stack上申请内存，这两种情况的变量只能pass by ref。

```rust
use std::mem::size_of;

fn main() {
    // primitives
    assert_eq!(4, size_of::<i32>());
    assert_eq!(8, size_of::<f64>());

    // tuples
    assert_eq!(8, size_of::<(i32, i32)>());

    // arrays
    assert_eq!(0, size_of::<[i32; 0]>());
    assert_eq!(12, size_of::<[i32; 3]>());

    struct Point {
        x: i32,
        y: i32,
    }

    // structs
    assert_eq!(8, size_of::<Point>());

    // enums
    assert_eq!(8, size_of::<Option<i32>>());

    // get pointer width, will be
    // 4 bytes wide on 32-bit targets or
    // 8 bytes wide on 64-bit targets
    const WIDTH: usize = size_of::<&()>();

    // pointers to sized types are 1 width
    assert_eq!(WIDTH, size_of::<&i32>());
    assert_eq!(WIDTH, size_of::<&mut i32>());
    assert_eq!(WIDTH, size_of::<Box<i32>>());
    assert_eq!(WIDTH, size_of::<fn(i32) -> i32>());

    const DOUBLE_WIDTH: usize = 2 * WIDTH;

    // unsized struct
    struct Unsized {
        unsized_field: [i32],
    }

    // pointers to unsized types are 2 widths
    assert_eq!(DOUBLE_WIDTH, size_of::<&str>()); // slice
    assert_eq!(DOUBLE_WIDTH, size_of::<&[i32]>()); // slice
    assert_eq!(DOUBLE_WIDTH, size_of::<&dyn ToString>()); // trait object
    assert_eq!(DOUBLE_WIDTH, size_of::<Box<dyn ToString>>()); // trait object
    assert_eq!(DOUBLE_WIDTH, size_of::<&Unsized>()); // user-defined unsized type

    // unsized types
    size_of::<str>(); // compile error
    size_of::<[i32]>(); // compile error
    size_of::<dyn ToString>(); // compile error
    size_of::<Unsized>(); // compile error
}
```



从上面的代码可以看出来，Rust中的原生类型都有确定的大小，像struct,enum,array，以及由原生与类型与指针或者其它nested struct enum等组成的类型都能在编译期确定大小。而像slice是不能确定大小的，因为我们并不知道这个slice到底会多大，这些只有运行时才会知道。

- 一个指向动态大小视图的指针称为slice，比如&str就是*string slice*

- slice两个width大小是因为存了一个指向数据的指针与元素的数量 

- trait两个width大小是因为存一个指向数据的指针与指向虚表的指针

- unsized struct两个width大小是因为存了一个指针与这个struct 的大小

- unsized struct只能一个unsized filed而且这个必须是struct中的最后一个字段

下面有些例子可以看slice跟array的区别：

```rust
use std::mem::size_of;

const WIDTH: usize = size_of::<&()>();
const DOUBLE_WIDTH: usize = 2 * WIDTH;

fn main() {
    // data length stored in type
    // an [i32; 3] is an array of three i32s
    let nums: &[i32; 3] = &[1, 2, 3];

    // single-width pointer
    assert_eq!(WIDTH, size_of::<&[i32; 3]>());

    let mut sum = 0;

    // can iterate over nums safely
    // Rust knows it's exactly 3 elements
    for num in nums {
        sum += num;
    }

    assert_eq!(6, sum);

    // unsized coercion from [i32; 3] to [i32]
    // data length now stored in pointer
    let nums: &[i32] = &[1, 2, 3];

    // double-width pointer required to also store data length
    assert_eq!(DOUBLE_WIDTH, size_of::<&[i32]>());

    let mut sum = 0;

    // can iterate over nums safely
    // Rust knows it's exactly 3 elements
    for num in nums {
        sum += num;
    }

    assert_eq!(6, sum);
}
```

下面有些例子可以看struct 与 trait的区别：

```rust
use std::mem::size_of;

const WIDTH: usize = size_of::<&()>();
const DOUBLE_WIDTH: usize = 2 * WIDTH;

trait Trait {
    fn print(&self);
}

struct Struct;
struct Struct2;

impl Trait for Struct {
    fn print(&self) {
        println!("struct");
    }
}

impl Trait for Struct2 {
    fn print(&self) {
        println!("struct2");
    }
}

fn print_struct(s: &Struct) {
    // always prints "struct"
    // this is known at compile-time
    s.print();
    // single-width pointer
    assert_eq!(WIDTH, size_of::<&Struct>());
}

fn print_struct2(s2: &Struct2) {
    // always prints "struct2"
    // this is known at compile-time
    s2.print();
    // single-width pointer
    assert_eq!(WIDTH, size_of::<&Struct2>());
}

fn print_trait(t: &dyn Trait) {
    // print "struct" or "struct2" ?
    // this is unknown at compile-time
    t.print();
    // Rust has to check the pointer at run-time
    // to figure out whether to use Struct's
    // or Struct2's implementation of "print"
    // so the pointer has to be double-width
    assert_eq!(DOUBLE_WIDTH, size_of::<&dyn Trait>());
}

fn main() {
    // single-width pointer to data
    let s = &Struct; 
    print_struct(s); // prints "struct"
    
    // single-width pointer to data
    let s2 = &Struct2;
    print_struct2(s2); // prints "struct2"
    
    // unsized coercion from Struct to dyn Trait
    // double-width pointer to point to data AND Struct's vtable
    let t: &dyn Trait = &Struct;
    print_trait(t); // prints "struct"
    
    // unsized coercion from Struct2 to dyn Trait
    // double-width pointer to point to data AND Struct2's vtable
    let t: &dyn Trait = &Struct2;
    print_trait(t); // prints "struct2"
}
```

## Sized Trait

Sized trait是Rust一个marker trait和auto trait。auto trait是说如果一个类型满足了条件则会被自动实现。而markter trait说用来标记一个类型是某个特定的属性的。marker trait没有任何的trait item，比如函数，变量等。所有的auto trait都是marker trait，但是不是所有的marker trait都是auto trait。auto trait必须得是marker trait，这样编译器才能提供一个默认的实现 。
一个类型如果它的所有的成员都是sized的，那么它就是`Sized`。同样的trait还有`Send`与`Sync`。如果一个类型能安全地在线程间传递，那么它就是`Send`的，如果一个类型能在线程间安全的shared ref，那就是`Sync`的。但是`Sized`有点不同的是，不能对一个类型把它*去掉*：
```rust
#![feature(negative_impls)]

// this type is Sized, Send, and Sync
struct Struct;

// opt-out of Send trait
impl !Send for Struct {}

// opt-out of Sync trait
impl !Sync for Struct {}

impl !Sized for Struct {} // compile error

```
## 泛型参数中的`Sized`
```rust
// this generic function...
fn func<T>(t: T) {}

// ...desugars to...
fn func<T: Sized>(t: T) {}

// ...which we can opt-out of by explicitly setting ?Sized...
fn func<T: ?Sized>(t: T) {} // compile error

// ...which doesn't compile since t doesn't have
// a known size so we must put it behind a pointer...
fn func<T: ?Sized>(t: &T) {} // compiles
fn func<T: ?Sized>(t: Box<T>) {} // compiles
```
当我们写下一个泛型函数时，T就得到了`Sized`的trait。
- `?Sized`意思是这个类型可能是Sized或者是也许Sized的。这能让类型可以是Sized也可以的Unsized。
- `?Sized`一般用来减少参数的约束，且是Rust中唯一能减少约束的措施。

一般来说，我们还是需要放松对泛型参数的约束，因为实例化的时候， 实际类型可能是Sized也有可能是Unsized。
有如下代码：
```rust
use std::fmt::Debug;

fn debug<T: Debug>(t: T) { // T: Debug + Sized
    println!("{:?}", t);
}

fn main() {
    debug("my str"); // T = &str, &str: Debug + Sized ✔️


```
虽然这样可行，但是如果参数是个ref呢？
```rust
use std::fmt::Debug;

fn dbg<T: Debug>(t: &T) { // T: Debug + Sized
    println!("{:?}", t);
}

fn main() {
    dbg("my str"); // &T = &str, T = str, str: Debug + !Sized ❌
}
```
错误如下：
```rust
error[E0277]: the size for values of type `str` cannot be known at compilation time
 --> src/main.rs:8:9
  |
3 | fn dbg<T: Debug>(t: &T) {
  |        - required by this bound in `dbg`
...
8 |     dbg("my str");
  |         ^^^^^^^^ doesn't have a size known at compile-time
  |
  = help: the trait `std::marker::Sized` is not implemented for `str`
  = note: to learn more, visit <https://doc.rust-lang.org/book/ch19-04-advanced-types.html#dynamically-sized-types-and-the-sized-trait>
help: consider relaxing the implicit `Sized` restriction
  |
3 | fn dbg<T: Debug + ?Sized>(t: &T) {
  |   

```
这其中的类型也许有点怪，但是看表就很好理解了：

| Type   | `T`          | `&T`        |
| ------ | ------------ | ----------- |
| `&str` | `T` = `&str` | `T` = `str` |

| Type    | `Sized` |
| ------- | ------- |
| `str`   | ❌       |
| `&str`  | ✔️       |
| `&&str` | ✔️       |

所以最好对泛型参数加上一个`?Sized`的标记来放松对类型的约束。



## Unsized type

### slices

最常见的切片是str与数组的切片&[T]。切片的一个优点是许多的类型都能转换成它们。强制类型转换在Rust中很常见，特别是在函数参数中。一种类型转换是deref和unsized。一个deref转换是当`T`通过一个deref操作转换成`U`，比如`T:Deref<Target=U>`比如`STring.deref() -> str`。一个unsized转换是把T转换成U，其中T是sized type 而U是unsized type。比如`T:Unsized<U>`，[i32;3] ->[i32]。

```rust
trait Trait {
    fn method(&self) {}
}

impl Trait for str {
    // can now call "method" on
    // 1) str or
    // 2) String since String: Deref<Target = str>
}
impl<T> Trait for [T] {
    // can now call "method" on
    // 1) any &[T]
    // 2) any U where U: Deref<Target = [T]>, e.g. Vec<T>
    // 3) [T; N] for any N, since [T; N]: Unsize<[T]>
}

fn str_fun(s: &str) {}
fn slice_fun<T>(s: &[T]) {}

fn main() {
    let str_slice: &str = "str slice";
    let string: String = "string".to_owned();

    // function args
    str_fun(str_slice);
    str_fun(&string); // deref coercion

    // method calls
    str_slice.method();
    string.method(); // deref coercion

    let slice: &[i32] = &[1];
    let three_array: [i32; 3] = [1, 2, 3];
    let five_array: [i32; 5] = [1, 2, 3, 4, 5];
    let vec: Vec<i32> = vec![1];

    // function args
    slice_fun(slice);
    slice_fun(&vec); // deref coercion
    slice_fun(&three_array); // unsized coercion
    slice_fun(&five_array); // unsized coercion

    // method calls
    slice.method();
    vec.method(); // deref coercion
    three_array.method(); // unsized coercion
    five_array.method(); // unsized coercion
}
```

### Trait Object

对于一个Trait来说，是?Sized的。因为不知道是Sized type 还是unsized type会去实现它。

`Trait : ?Sized`对于`impl Trait for dyn Trait`是必要的。

我们能对每一个方法加个Self:Sized的约束，但是不能给Trait加个Sized约束。

#### Trait Object 的局限性

- 不能把一个Unsized的类型转换成Trait Object

```rust
fn generic<T: ToString>(t: T) {}
fn trait_object(t: &dyn ToString) {}

fn main() {
    generic(String::from("String")); // compiles
    generic("str"); // compiles
    trait_object(&String::from("String")); // compiles, unsized coercion
    trait_object("str"); // compile error, unsized coercion impossible
}
```

- 不能创造出多Trait的object的动态分发：因为要存两个trait的vtable要更多的内存，而正确的组合方式如下：

  ```rust
  trait Trait {
      fn method(&self) {}
  }
  
  trait Trait2 {
      fn method2(&self) {}
  }
  
  trait Trait3: Trait + Trait2 {}
  
  // auto blanket impl Trait3 for any type that also impls Trait & Trait2
  impl<T: Trait + Trait2> Trait3 for T {}
  
  // from `dyn Trait + Trait2` to `dyn Trait3` 
  fn function(t: &dyn Trait3) {
      t.method(); // compiles
      t.method2(); // compiles
  }
  ```

- Rust的trait不支持向上转换：

  ```rust
  trait Trait {
      fn method(&self) {}
  }
  
  trait Trait2 {
      fn method2(&self) {}
  }
  
  trait Trait3: Trait + Trait2 {}
  
  impl<T: Trait + Trait2> Trait3 for T {}
  
  struct Struct;
  impl Trait for Struct {}
  impl Trait2 for Struct {}
  
  fn takes_trait(t: &dyn Trait) {}
  fn takes_trait2(t: &dyn Trait2) {}
  
  fn main() {
      let t: &dyn Trait3 = &Struct;
      takes_trait(t); // compile error
      takes_trait2(t); // compile error
  }
  ```

  一个正确的做法的是去写显式的转换函数：

  ```rust
  trait Trait {}
  trait Trait2 {}
  
  trait Trait3: Trait + Trait2 {
      fn as_trait(&self) -> &dyn Trait;
      fn as_trait2(&self) -> &dyn Trait2;
  }
  
  impl<T: Trait + Trait2> Trait3 for T {
      fn as_trait(&self) -> &dyn Trait {
          self
      }
      fn as_trait2(&self) -> &dyn Trait2 {
          self
      }
  }
  
  struct Struct;
  impl Trait for Struct {}
  impl Trait2 for Struct {}
  
  fn takes_trait(t: &dyn Trait) {}
  fn takes_trait2(t: &dyn Trait2) {}
  
  fn main() {
      let t: &dyn Trait3 = &Struct;
      takes_trait(t.as_trait()); // compiles
      takes_trait2(t.as_trait2()); // compiles
  }
  ```

### User-Defined Unsized Type

```rust
struct Unsized {
    unsized_field: [i32],
}
```

定义一个unsized type的方法是，给struct一个unsize filed。这个unsized filed只能一个，且是最后一个。

如果要定义一个不知道是不是Unsized type的struct，最好是用泛型去定义这个struct。

我们常用的std::ffi:OsStr与std::path::Path就是标准库中的unsized struct。

### Zero-Sized Types

Unit Type: `()` and `{}`；每一个函数如果没有显示的return的话都会返回一个`()`。

尽管`()`是0大小类型，但是还是能给它实现一些trait：

```rust
use std::cmp::Ordering;

impl Default for () {
    fn default() {}
}

impl PartialEq for () {
    fn eq(&self, _other: &()) -> bool {
        true
    }
    fn ne(&self, _other: &()) -> bool {
        false
    }
}

impl Ord for () {
    fn cmp(&self, _other: &()) -> Ordering {
        Ordering::Equal
    }
}
```

编译器知道`()`是zero-sized type。比如：

```rust
fn main() {
    // zero capacity is all the capacity we need to "store" infinitely many ()
    let mut vec: Vec<()> = Vec::with_capacity(0);
    // causes no heap allocations or vec capacity changes
    vec.push(()); // len++
    vec.push(()); // len++
    vec.push(()); // len++
    vec.pop(); // len--
    assert_eq!(2, vec.len());
}
```

编译器就只会给Vec中的len操作，而不会在heap上去申请内存。

#### User-Defined Unit Structs

用户定义的Unit Struct:

```rust
struct Struct;
```

实际上Unit Struct比`()`有用的多，因为孤儿规则的存在，我们不能给`()`实现trait，因为它是标准库里的。这样我们可以实现一个等同的，更有意义的名字的Unit struct来给代码带来可读性。

#### Never type

Never type是第二个重要的ZST，它的代表了一个计算永远不会有结果的意思 。一些有趣的事情是：

- `!`能转换成其它任何类型
- 没法创建出一个`！`的实例。

第一点的用处在于宏与break,continue等都有`!`类型，使得可以用在返回结果里，因为能转换成任何类型

而第二点可以实现出一定会成功的函数:

```rust
fn function() -> Result<Success, !>;
```

而`fn function() -> Result<!, Error>;`是永远不会成功的函数。因为在返回的时候 ，创造不出它的实例来。

#### User-Defined Pseudo Never Type

我们可以用用户自定义的never type来实现永远不会失败的函数：

```rust
enum Void {}

// example 1
impl FromStr for String {
    type Err = Void;
    fn from_str(s: &str) -> Result<String, Self::Err> {
        Ok(String::from(s))
    }
}

// example 2
fn run_server() -> Result<Void, ConnectionError> {
    loop {
        let (request, response) = get_request()?;
        let result = request.process();
        response.send(result);
    }
}
```

