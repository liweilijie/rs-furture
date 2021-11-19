# AsRef
`AsRef`特性是一个转换特性。它用来在泛型中把一些值转换为引用。像这样：
```rust
let s = "hello".to_string();

fn foo<T: AsRef<str>>(s: T) {
  let slice = s.as_ref();
}
```

认真把这篇文章读懂就明白如何使用了[AsRef](https://stackauth.com/auth/oauth2/github?code=4c5e231aa08b991657a5&state=%7B%22sid%22%3A1%2C%22st%22%3A%2259%3A3%3Abbc%2C16%3A49260bdc2d6801b5%2C10%3A1637315039%2C16%3A0f54a230ff1d70be%2C77be37a5c35126a2544a39f79b6287ccb5e26dd5770297c638876edf26dde864%22%2C%22cid%22%3A%2201b478c0264a1fbd7183%22%2C%22k%22%3A%22GitHub%22%2C%22ses%22%3A%22e064074472954fc9b9b7d6ac432fe9ed%22%7D)