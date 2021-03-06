# 常用的 Trait
 这里列举一些常用的`Trait`，积累在这里，以免随时查阅复习。

[trait 常用](https://rustwiki.org/zh-CN/rust-by-example/conversion.html)

Rust 使用`trait` 解决类型之间的转换问题，最一般的转换会用到 From 和 into 两个 trait, 也有特殊的情况比如用 String 转换到别的类型，以及把别的类型转换到 String 时。

## Deref && DerefMut
实现 `Deref trait`允许我们重载 解引用运算符（dereference operator）*（与乘法运算符或通配符相区别）。

- 看这个文档比较清晰[通过 Deref trait 将智能指针当作常规引用处理](https://kaisery.github.io/trpl-zh-cn/ch15-02-deref.html)
- [Deref 设计者给它最神奇之处在于：强制隐式转换](https://rustcc.gitbooks.io/rustprimer/content/intoborrow/deref.html)

```rust
use std::ops::Deref;

struct MyBox<T>(T);

impl<T> MyBox<T> {
  fn new(x: T) -> MyBox<T> {
    MyBox(x)
  }
}

impl<T> Deref for MyBox<T> {
  type Target = T;

  fn deref(&self) -> &T {
    &self.0
  }
}

fn hello(name: &str) {
  println!("Hello, {}!", name);
}

fn main() {
  let m = MyBox::new(String::from("Rust"));
  hello(&m); // 利用了 Deref 强制转换
}
```

Rust 在发现类型和 Trait 实现满足三种情况时会进行 Deref 强制转换：
- 当 `T: Deref<Target=U>` 时从`&T` 到`&U`
- 当 `T: DerefMut<Target=U>` 时从`&mut T` 到 `&mut U`
- 当 `T: Deref<Target=U>` 时从`&mut T` 到`&U`

## From/Into && TryFrom/TryInto

- [from/into](https://rustcc.gitbooks.io/rustprimer/content/intoborrow/into.html)
- [示例](https://rustwiki.org/zh-CN/rust-by-example/conversion/from_into.html)

Into trait 就是把 From trait 倒过来而已。也就是说，如果你为你的类型实现了 From，那么同时你也就免费获得了 Into。
使用 Into trait 通常要求指明要转换到的类型，因为编译器大多数时候不能推断它。不过考虑到我们免费获得了 Into，这点代价不值一提。

```rust
// From
let string = "hello".to_string();
let other_string = String::from("hello");
assert_eq!(string, other_string);

// Into
fn is_hello<T: Into<Vec<u8>>>(s: T) {
  let bytes = b"hello".to_vec();
  assert_eq!(bytes, s.into());
}

let s = "hello".to_string();
is_hello(s);
```

下面拿一个实际生产中字符串作为函数参数的例子来说明。
```rust
struct Person {
  name: String,
}

impl Person {
  fn new<S: Into<String>>(name: S) -> Person {
    Person{ name: name.into() }
  }
}

fn main() {
  let person = Person::new("Herman");
  let person = Person::new("Herman".to_string());
}
```
参数类型为 S， 是一个泛型参数，表示可以接受不同的类型。S: Into<String> 表示 S 类型必须实现了 Into<String>（约束）。而 &str 类型，符合这个要求。因此 &str 类型可以直接传进来。

而 String 本身也是实现了 Into<String> 的。当然也可以直接传进来。

然后，下面 name: name.into() 这里也挺神秘的。它的作用是将 name 转换成 String 类型的另一个对象。当 name 是 &str 时，它会转换成 String 对象，会做一次字符串的拷贝（内存的申请、复制）。而当 name 本身是 String 类型时，name.into() 不会做任何转换，代价为零（有没有恍然大悟）。

```rust
use std::convert::From;

#[derive(Debug)]
struct Number {
  value: i32,
}

impl From<i32> for Number {
  fn from(item: i32) -> Self {
    Number { value: item }
  }
}

fn main() {
  let num = Number::from(30);
  println!("My number is {:?}", num);

  let int = 5;
  let num: Number = int.into();
  println!("My number is {:?}", num);
}
```

## ToString/FromStr

- [例子学习](https://rustwiki.org/zh-CN/rust-by-example/conversion/string.html)
- [fromstr](https://runebook.dev/zh-CN/docs/rust/std/str/trait.fromstr)

```rust
use std::str::FromStr;
use std::num::ParseIntError;

#[derive(Debug, PartialEq)]
struct Point {
  x: i32,
  y: i32
}

impl FromStr for Point {
  type Err = ParseIntError;

fn from_str(s: &str) -> Result<Self, Self::Err> {
    let coords: Vec<&str> = s.trim_matches(|p| p == '(' || p == ')')
                              .split(',')
                              .collect();
    let x_fromstr = coords[0].parse::<i32>()?;
    let y_fromstr = coords[1].parse::<i32>()?;

    Ok(Point{ x: x_fromstr, y: y_fromstr})
  }
}

fn main() {
  let p = Point::from_str("(1,2)");
  assert_eq!(p.unwrap(), Point{x: 1, y: 2});
}
```

## AsRef && AsMut
[AsRef 和 AsMut](https://wiki.jikexueyuan.com/project/rust-primer/intoborrow/asref.html)

`std::convert`下面，还有另外两个`Trait`，`AsRef/AsMut`，它们功能是配合泛型，在执行引用操作的时候，进行自动类型转换。这能够使一些场景的代码实现得清晰漂亮，大家方便开发。

`AsRef`提供了一个方法`.as_ref()`。

对于一个类型为`T`的对象`foo`，如果`T`实现了`AsRef<U>`，那么，`foo`可执行`.as_ref()`操作，即`foo.as_ref()`。操作的结果，我们得到了一个类型为`&U`的新引用。

注：

1. 与`Into<T>`不同的是，`AsRef<T>`只是类型转换，`foo`对象本身没有被消耗；
2. `T: AsRef<U>`中的`T`，可以接受 资源拥有者（owned）类型，共享引用（shared referrence）类型 ，可变引用（mutable referrence）类型。

下面举个简单的例子：
```rust
fn is_hello<T: AsRef<str>>(s: T) {
  assert_eq!("hello", s.as_ref());
}

fn main() {
  let s = "hello";
  is_hello(s);

  let s = "hello".to_string();
  is_hello(s); 
}
```
**因为 String 和 &str 都实现了 AsRef<str>**

`AsRef`特性是一个转换特性。它用来在泛型中把一些值转换为引用。像这样：
```rust
let s = "hello".to_string();

fn foo<T: AsRef<str>>(s: T) {
  let slice = s.as_ref();
}
```

在 stackoverflow 上面有一个示例：

我们需要一个用户成为版主，而且版主有不同的权限，我们可能这样来设计
```rust
// 用户信息
struct User {
  email: String,
  age: u8,
}

enum Privilege {
  // imagine different moderator privileges here
  // 想象一下论坛版主有不同的权限在这里
  Super,
  Admin,
}

// 论坛版主
struct Moderator {
  user: User,
  privileges: Vec<Privilege>,
}
```

 现在有点浪费空间, 而且比较笨的一种方式来获取 User 的值.

 ```rust
 #[derive(Default)]
 struct User {
   email: String,
   age: u8,
 }

enum Privilege {
  // imagine different moderator privileges here
  // 想象一下论坛版主有不同的权限在这里
  Super,
  Admin,
}

#[derive(Default)]
struct Moderator {
  user: User,
  privileges: Vec<Privilege>,
}

fn takes_user(user: &User) {}

fn main() {
  let user = User::default();
  let moderator = Moderator::default();

  takes_user(&user);
  takes_user(&moderator.user); // 很笨的一种方式
}
 ```

 如果任何期望用`&User`的地方我们也可以通过`&moderator` 调用， 用`AsRef`我们就可以做到，实现如下：
 ```rust
 #[derive(Default)]
 struct User {
   email: string,
   age: u8,
 }

// obviously 显而易见的
impl AsRef<User> for User {
  fn as_ref(&self) -> &User {
    self
  }
}

enum Privilege {
  // imagine different moderator privileges here
  // 想象一下论坛版主有不同的权限在这里
  Super,
  Admin,
}

#[derive(Default)]
struct Moderator {
  user: User,
  privileges: Vec<Privilege>,
}

// 因为版主也只是普通的用户而已
// since moderators are just regular users
impl AsRef<User> for Moderator {
  fn as_ref(&self) -> &User {
    &self.user
  }
}

fn takes_user<U: AsRef<User>>(user: U) {}

fn usage_asref() {
  let user = User::default();
  let moderator = Moderator::default();

  takes_user(&user);
  takes_user(&moderator); // yay 哇
}

// test module
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_asref() {
        usage_asref();
    }
}
 ```
现在我们可以通过`&Moderator`到任何函数期望&User 的参数的地方，我们只需要一点代码的重构即可。而且我们也扩张到其他类型，比如`Admin, PowerUser, SubscribedUser`等，他们只需要实现`AsRef<User>`即可工作。

 从`&Moderator`到`&User`能正常工作的原因是因为实现了：`impl AsRef<User> for &Moderator`

 ```rust
 impl<T: ?Sized, U: ?Sized> AsRef<U> for &T
 where
    T: AsRef<U>,
{
  fn as_ref(&self) -> &U {
    <T as AsRef<U>>::as_ref(*self)
  }
}
 ```
 > Which basically just says if we have some`impl AsRef<U> for T`we also automatically get`impl AsRef<U> for &T`for all `T` for free.

 这就是意味着我们有`impl AsRef<U> for T` 我们也自动免费获得`impl AsRef<U> for &T`

### AsMut
`AsMut<T>`提供了一个方法`.as_mut()`。它是`AsRef<T>`的可变（mutable）引用版本。

对于一个类型为`T`的对象`foo`，如果`T`实现了`AsMut<U>`，那么，`foo`可执行`.as_mut()`操作，即`foo.as_mut()`。操作的结果，我们得到了一个类型为`&mut U`的可变（mutable）引用。

注：在转换的过程中，`foo`会被可变（mutable）借用。

## Sized trait

Sized 在Rust中是一个比较特殊的 trait ，该Sized trait默认是自动实现的。一个类型是否是Sized，要看它在编译期内的size是否已知且固定不变。比如，u8 的大小是 1 byte。

有些类型在编译期无法确定大小。

一个 slice的[T]的size是未知的，因为在编译期不知道到底会有多少个T存在。

一个trait的size是未知的，因为不知道实现这个trait的结构是什么。

把unsized的类型放到指针或者Box里面，就变成了sized了，通过指针找到源头，然后顺着源头找到其他的数据。

所有的类型参数，如fn foo<T>(){}中的T，默认都是实现了Sized的了（自动实现），这就限制了传参数的数据了，

## ?Sized trait

`?Sized`就表示**UnSized类型**。

`fn foo<T:?Sized>(){}`现在可以接受**UnSized**的数据类型了。

`?Sized`是一个非常特殊的用法，其他的`trait bound`，都是用来缩小数据的类型范围的，这个是用来**扩大**类型范围的。也就是说

### 示例
例如 HashMap有一个用了 Borrow 的 get 方法：
```rust
pub fn get<Q: ?Sized>(&self, k: &Q) -> Option<&V>
    where
        K: Borrow<Q>,
        Q: Hash + Eq,
    {
        self.base.get(k)
    }
```