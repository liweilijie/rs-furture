# 基础篇

## ownership

Rust通过单一所有权来限制任意引用的行为。

所有权和Move语义Rust给出了如下规则：

- 一个值只能被一个变量所拥有，这个变量被称为所有者。(Each value in Rust has a variable that's called its owner.)
- 一个值同一时刻只能有一个所有者(There can only be one owner at a time)。
- 当所有者离开作用域，其拥有的值被丢弃(When the owner goes out of scoe, the value will be dropped)。

实现了Copy语义的类型有哪些：
- 原生类型，包括函数、不可变引用和裸指针实现了Copy;
- 数组和元组，如果其内部的数据结构实现了Copy,那么它们也实现了Copy;
- 可变引用没有实现Copy;
- 非固定大小的数据结构，没有实现Copy;

## Borrow
其实，在Rust中，"借用"和"引用"是一个概念，只不过在其他语言中引用的意义和Rust不同，所以Rust提出新的概念叫"借用"来便于区分。在Rust所有的引用只是借用了"临时所有权"，它并不破坏值的单一所有权约束。

Rust没有传引用的概念，Rust所有的参数传递都是传值。

- 在一个作用域内，仅允许一个活跃的可变引用。
- 在一个作用域内，活跃的可变引用(写)和只读引用(读)是互斥的，不能同时存在。


Rc是利用了Rust的Box::leak()机制来使内存系统撕开了一道口子。
Rc是一个只读的引用计数器。要修改数据需要使用RefCell来Rust编译器的静态检查，允许我们在运行时，对某个只读数据进行可变借用。这就涉及Rust另一个比较独特且有点难懂的概念：**内部可变性**(interior mutability)。
在编译器的眼里，值是只读的，但是在运行时，这个值可以得到可变借用，从而修改内部的数据。这就是RefCell的用武之地。

因为引用计数的智能指针只有`Rc`和`Arc`，所以需要用Rc或者Arc将其RefCell包裹。

```rust
use std::cell::RefCell;
fn main() {
    let data = RefCell::new(1);
    {
        // 获得RefCell内部数据的可变借用
        let mut v = data.borrow_mut();
        *v += 1;
    } // 这里需要缩小可变借用的作用域，因为不能同时有活跃的可变借用和不可变借用
    println!("data: {:?}", data.borrow());
}
```

利用RefCell实现一个DAG的可修改的代码：
请先熟悉一下`Option`之中的`as_ref()`方法。是`Converts from &Option<T> to Option<T>`
```rust
let text: Option<String> = Some("hello".to_string());
let text_length: Option<usize> = text.as_ref().map(|s|s.len());
println!("still can print text: {:?}", text);
```
`Option.map()`: Maps an Option<T> to Option<U> by applying a function to a contained value. 建议好好看文档，读英文文档，很容易理解。

再看具体的实现
```rust
use std::cell::RefCell;
use std::rc::Rc;

#[derive(Debug)]
struct Node {
    id: usize,
    // 一个节点可能被多个其它节点指向，所以我们使用Rc<T>来表述
    // 一个节点可能没有下游节点，所以我们用Option<Rc<T>>来表述它
    // 一个节点内部值我们需要可变引用修改，所以使用 Rc<RefCell<T>> 让节点可以被修改
    downstream: Option<Rc<RefCell<Node>>>;
    // 另外如果为了线程安全，Rc需要换成Arc，另外RefCell不是多线程安全的,需要换成Mutex,RwLock
    // Rc<RefCell<T>> 替换为 Arc<Mutex<T>>或者Arc<RwLock<T>>
}

impl Node {
    pub fn new(id: usize) -> Self { // 新建一个Node
        Self { id, downstream: None, }
    }

    // 设置Node的downstream.
    pub fn update_downstream(&mut self, downstream: Rc<RefCell<Node>>) {
        self.downstream = Some(downstream);
    }

    // clone一份Node里的downstream.
    pub fn get_downstream(&self) -> Option<Rc<RefCell<Node>>> {
        self.downstream.as_ref().map(|v|v.clone())
    }
}

fn main() {
    let mut node1 = Node::new(1);
    let mut node2 = Node::new(2);
    let mut node3 = Node::new(3);
    let node4 = Node::new(4);

    node3.update_downstream(Rc::new(RefCell::new(node4)));
    node1.update_downstream(Rc::new(RefCell::new(node3)));
    node2.update_downstream(node1.get_downstream().unwrap());
    println!("node1: {:?}, node2: {:?}", node1, node2);

    let node5 = Node::new(5);
    let node3 = node1.get_downstream().unwrap();
    // 获取可变引用，来修改downstream
    node3.borrow_mut().downstream = Some(Rc::new(RefCell::new(node5)));

    println!("node1: {:?}, node2: {:?}", node1, node2);
}
```

![](./images/Rc.RefCell.png)

## lifetimes

如果一个值的生命周期贯穿整个进程的生命周期，那么我们就称这种生命周期为静态生命周期。

当值拥有静态生命周期，其引用也具有静态生命周期。我们在表述这种引用的时候，可以用`'static`来表示。 比如`&'static str`代表这是一个具有静态生命周期的字符串引用。

1. 所有引用类型的参数都有独立的生命周期`'a`,`'b`等。
2. 如果只有一个引用型输入，它的生命周期会赋给所有输出。
3. 如果有多个引用类型的参数，其中一个是**self**,那么它的生命周期会赋给所有输出。

来尝试实现一个`strtok()`字符串分割函数，把字符串按照分割符delimiter切出一个token并返回，然后将传入的字符串引用指向后续的token。
由于传入的s需要可变引用，所以它是一个指向字符串引用的可变引用 `&mut &str`:
然后`&mut &str`添加生命周期之后变成了`&'b mut &'a str`, 它将导致返回的`&str`无法选择一个合适的生命周期。

返回值和谁的生命周期有关？是指向字符串引用的可变引用`&mut`,还是字符串引用`&str`本身. 当然是字符串引用本身.
`pub fn strtok(s: &mut &str, delimiter: char) -> &str {` 肯定会报错.
改成这样:
`pub fn strtok<'b, 'a>(s: &'b mut &'a str, delimiter: char) -> &'a str {`
因为返回值的生命周期跟字符串有关,我们只为这部分的约束添加标注就可以了,剩下的标注交给编译器自动添加,所以代码简化成:
`pub fn strtok<'a>(s: &mut &'a str, delimiter: char) -> &'a str {`


```rust
pub fn strtok<'a>(s: &mut &'a str, delimiter: char) -> &'a str {
    if let Some(i) = s.find(delimiter) {
        let prefix = &s[..i];
        // 由于delimiter可以是utf8, 所以我们需要获得其utf8长度
        // 直接使用len返回的是字节长度，会有问题
        let suffix = &s[(i + delimiter.len_utf8())..];
        *s = suffix;
        prefix
    } else { // 如果没找到，返回整个字符串，把原字符串指针s指向空串
        let prefix = *s;
        *s = ""; // TODO: ???
        prefix
    }
}

fn main() {
    let s = "hello world".to_owned();
    let mut s1 = s.as_str();
    let hello = strtok(&mut s1, ' ');
    println!("hello is: {}, s1: {}, s: {}", hello, s1, s);
}
```

函数和结构体的生命周期相似的.比如下面的例子, Employee的name和title是两个字符串引用, Employee的生命周期不能大于它们,否则会访问失效的内存,因而我们需要妥善标注:
```rust
struct Employee<'a, 'b> {
    name: &'a str,
    title: &'b str,
    age: u8,
}
```

## 全局变量
我们讲过const和static都可以用于声明全局变量,但是注意除非使用unsafe, **static**无法作为mut使用.因为这意味着它可能在多个线程下被修改,所以不安全.static全局定义的变量在静态区,如果你的确想用可写的全局变量,可以用`Mutex<T>`, 然而初始化它很麻烦,所以用一个`lazy_static`库来解决.

```rust
use lazy_static::lazy_static;
use std::collections::HashMap;
use std::sync::{Arc, Mutex};
struct Color {
    r,g,b u8
}

lazy_static! {
    static ref HASHMAP: Arc<Mutex<HashMap<u32, &'static str>>> = {
        let mut m = HashMap::new();
        m.insert(0, "foo");
        m.insert(1, "bar");
        m.insert(2, "baz");
        Arc::new(Mutex::new(m))
    };

    static ref COLORS_MAP: HashMap<&'static str, Color> = {
        let mut map = HashMap::new();
        map.insert("amber", Color {r: 255, g: 191, b: 0});
        map.insert("zinnwaldite brown", Color{r: 44, g:22, b:8});
        map
    };
}

fn main() {
    let mut map = HASHMAP.lock().unwarp();
    map.insert(3, "waz");

    println!("map: {:?}", map);
}
```