# Sized trait

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

## 示例
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