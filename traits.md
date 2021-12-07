# 常用的 Trait
 这里列举一些常用的`Trait`，积累在这里，以免随时查阅复习。

## Deref && DerefMut
实现 `Deref trait`允许我们重载 解引用运算符（dereference operator）*（与乘法运算符或通配符相区别）。

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

## AsRef
`AsRef`特性是一个转换特性。它用来在泛型中把一些值转换为引用。像这样：
```rust
let s = "hello".to_string();

fn foo<T: AsRef<str>>(s: T) {
  let slice = s.as_ref();
}
```

认真把这篇文章读懂就明白如何使用了[AsRef](https://stackauth.com/auth/oauth2/github?code=4c5e231aa08b991657a5&state=%7B%22sid%22%3A1%2C%22st%22%3A%2259%3A3%3Abbc%2C16%3A49260bdc2d6801b5%2C10%3A1637315039%2C16%3A0f54a230ff1d70be%2C77be37a5c35126a2544a39f79b6287ccb5e26dd5770297c638876edf26dde864%22%2C%22cid%22%3A%2201b478c0264a1fbd7183%22%2C%22k%22%3A%22GitHub%22%2C%22ses%22%3A%22e064074472954fc9b9b7d6ac432fe9ed%22%7D)


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