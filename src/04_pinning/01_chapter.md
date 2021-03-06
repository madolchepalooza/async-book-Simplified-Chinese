# Pin

为了对future进行轮询，需要将其用一个特别的，名为`Pin<T>`的类型来固定(_pin_)。如果你读了之前[`Future`的执行和Task]["Executing `Future`s and Tasks"]这一节中对[`Future`trait][the `Future` trait]的解释，应该能想起来`Pin`曾在`Future::poll`方法定义中的`self: Pin<&mut Self>`出现。但是这是什么意思？为什么要这么做？

## 为什么需要Pin

`Pin`和`Unpin`标记搭配工作。Pin能够保证实现了`!Unpin`的对象不会被移动。为了帮助理解，先来回忆下`async`/`.await`是如何工作的：

```rust,edition2018,ignore
let fut_one = /* ... */;
let fut_two = /* ... */;
async move {
    fut_one.await;
    fut_two.await;
}
```

从底层来看，其创建了一个实现了`Future`的匿名类型，并提供了如下的`poll`方法：

```rust,ignore
// The `Future` type generated by our `async { ... }` block
struct AsyncFuture {
    fut_one: FutOne,
    fut_two: FutTwo,
    state: State,
}

// List of states our `async` block can be in
enum State {
    AwaitingFutOne,
    AwaitingFutTwo,
    Done,
}

impl Future for AsyncFuture {
    type Output = ();

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<()> {
        loop {
            match self.state {
                State::AwaitingFutOne => match self.fut_one.poll(..) {
                    Poll::Ready(()) => self.state = State::AwaitingFutTwo,
                    Poll::Pending => return Poll::Pending,
                }
                State::AwaitingFutTwo => match self.fut_two.poll(..) {
                    Poll::Ready(()) => self.state = State::Done,
                    Poll::Pending => return Poll::Pending,
                }
                State::Done => return Poll::Ready(()),
            }
        }
    }
}
```

`poll`首次被调用时，会先对`fut_one`进行轮询。若`fut_one`未能完成，则会返回`AsyncFuture::poll`。每次future调用`poll`时都会找到上次执行到的地方，此过程接连进行，直到future成功完成。

但要是有一个使用了引用的`async`块，又会怎样呢？

比如说：

```rust,edition2018,ignore
async {
    let mut x = [0; 128];
    let read_into_buf_fut = read_into_buf(&mut x);
    read_into_buf_fut.await;
    println!("{:?}", x);
}
```

这个会编译成怎样的结构？

```rust,ignore
struct ReadIntoBuf<'a> {
    buf: &'a mut [u8], // 指向下面的 `x`
}

struct AsyncFuture {
    x: [u8; 128],
    read_into_buf_fut: ReadIntoBuf<'what_lifetime?>,
}
```

在此处，`ReadIntoBuf`future中保存了一个对我们定义的结构体的字段，`x`，的引用。若`AsyncFuture`的所有权发生转移，`x`(在内存中)的位置同样会转移，使得`read_into_buf_fut.buf`中保存的指针无效。

将future固定(_pin_)至内存的特定位置可以解决此问题，也因此可以安全地在`async`块中创建引用了。

## Pin详解

来用一个相对简单的例子来解释pin的作用吧。现在问题可以归结为 如何处理 Rust 中的 自引用类型(_self-referential type_) 中的引用。

例子如下：

```rust, ignore
use std::pin::Pin;

#[derive(Debug)]
struct Test {
    a: String,
    b: *const String,
}

impl Test {
    fn new(txt: &str) -> Self {
        Test {
            a: String::from(txt),
            b: std::ptr::null(),
        }
    }

    fn init(&mut self) {
        let self_ref: *const String = &self.a;
        self.b = self_ref;
    }

    fn a(&self) -> &str {
        &self.a
    }

    fn b(&self) -> &String {
        unsafe {&*(self.b)}
    }
}
```

`Test`提供了用于获取字段`a`和`b`引用的方法。`b`是`a`的一个引用，由于Rust的借用规则不允许我们定义其生命周期，只好把它作为指针存储。现在就有了一个自引用结构体(_self-referential struct_)。

如果不转移数据的所有权，例子还是没什么问题的：

```rust
fn main() {
    let mut test1 = Test::new("test1");
    test1.init();
    let mut test2 = Test::new("test2");
    test2.init();

    println!("a: {}, b: {}", test1.a(), test1.b());
    println!("a: {}, b: {}", test2.a(), test2.b());

}
# use std::pin::Pin;
# #[derive(Debug)]
# struct Test {
#     a: String,
#     b: *const String,
# }
#
# impl Test {
#     fn new(txt: &str) -> Self {
#         Test {
#             a: String::from(txt),
#             b: std::ptr::null(),
#         }
#     }
#
#     // We need an `init` method to actually set our self-reference
#     fn init(&mut self) {
#         let self_ref: *const String = &self.a;
#         self.b = self_ref;
#     }
#
#     fn a(&self) -> &str {
#         &self.a
#     }
#
#     fn b(&self) -> &String {
#         unsafe {&*(self.b)}
#     }
# }
```
能够得到预料之中的结果：

```rust, ignore
a: test1, b: test1
a: test2, b: test2
```

但要是交换了`test1`和`test2`而因此使所有权发生了转移，就不大一样了：

```rust
fn main() {
    let mut test1 = Test::new("test1");
    test1.init();
    let mut test2 = Test::new("test2");
    test2.init();

    println!("a: {}, b: {}", test1.a(), test1.b());
    std::mem::swap(&mut test1, &mut test2);
    println!("a: {}, b: {}", test2.a(), test2.b());

}
# use std::pin::Pin;
# #[derive(Debug)]
# struct Test {
#     a: String,
#     b: *const String,
# }
#
# impl Test {
#     fn new(txt: &str) -> Self {
#         Test {
#             a: String::from(txt),
#             b: std::ptr::null(),
#         }
#     }
#
#     fn init(&mut self) {
#         let self_ref: *const String = &self.a;
#         self.b = self_ref;
#     }
#
#     fn a(&self) -> &str {
#         &self.a
#     }
#
#     fn b(&self) -> &String {
#         unsafe {&*(self.b)}
#     }
# }
```

从直觉上来说，理应得到debug两次打印了`test1`的内容：

```rust, ignore
a: test1, b: test1
a: test1, b: test1
```

但其实是这样的：

```rust, ignore
a: test1, b: test1
a: test1, b: test2
```

`test1`中指向`test2.b`的指针还指向原来(`test2.b`)的位置。结构体不再是自引用的，而是保存了一个指向不同对象的字段的指针。也就是说，`test2.b`和`test2`的生命周期也不再绑定在一起了。

还是不信的话，这个肯定能说服你了：

```rust
fn main() {
    let mut test1 = Test::new("test1");
    test1.init();
    let mut test2 = Test::new("test2");
    test2.init();

    println!("a: {}, b: {}", test1.a(), test1.b());
    std::mem::swap(&mut test1, &mut test2);
    test1.a = "I've totally changed now!".to_string();
    println!("a: {}, b: {}", test2.a(), test2.b());

}
# use std::pin::Pin;
# #[derive(Debug)]
# struct Test {
#     a: String,
#     b: *const String,
# }
#
# impl Test {
#     fn new(txt: &str) -> Self {
#         Test {
#             a: String::from(txt),
#             b: std::ptr::null(),
#         }
#     }
#
#     fn init(&mut self) {
#         let self_ref: *const String = &self.a;
#         self.b = self_ref;
#     }
#
#     fn a(&self) -> &str {
#         &self.a
#     }
#
#     fn b(&self) -> &String {
#         unsafe {&*(self.b)}
#     }
# }
```

下图能够将此过程可视化：

**Fig 1: Before and after swap**
![swap_problem](../assets/swap_problem.jpg)

这张图同样地，更明显地展现了此过程中的种种未定义行为或问题。

## Pin实践

现在来看看pin和`Pin`类型是怎么解决问题的。

`Pin`类型能够对指针类型进行包装，保证其指向的值不会移动。比如，`Pin<&mut T>`，`Pin<&T>`，`Pin<Box<T>>`对于满足`T: !Unpin`条件的`T`，均能保证`T`不被移动。

大部分类型的移动都不存在问题，因为它们实现了名为`Unpin`的trait。指向`Unpin`类型的指针 能够自由置于`Pin`中 或 从`Pin`中取出。比如说，`u8`是`Unpin`类型的，因此`Pin<&mut u8>`与普通的`&mut u8`使用方式并无二致。

但是，如果类型经`!Unpin`标记，就不再能进行移动了。`async`/`await`创建的future就是例子之一。

### 固定(_pin_)到栈上

再次回到例子中，现在可以用`Pin`来解决问题了。我们的例子在使用了一个经固定的指针后是这样的：

```rust, ignore
use std::pin::Pin;
use std::marker::PhantomPinned;

#[derive(Debug)]
struct Test {
    a: String,
    b: *const String,
    _marker: PhantomPinned,
}


impl Test {
    fn new(txt: &str) -> Self {
        Test {
            a: String::from(txt),
            b: std::ptr::null(),
            _marker: PhantomPinned, // This makes our type `!Unpin`
        }
    }
    fn init<'a>(self: Pin<&'a mut Self>) {
        let self_ptr: *const String = &self.a;
        let this = unsafe { self.get_unchecked_mut() };
        this.b = self_ptr;
    }

    fn a<'a>(self: Pin<&'a Self>) -> &'a str {
        &self.get_ref().a
    }

    fn b<'a>(self: Pin<&'a Self>) -> &'a String {
        unsafe { &*(self.b) }
    }
}
```

若对象实现了`!Unpin`，则将其固定至栈的行为总是`unsafe`的，可以用[`pin_utils`][pin_utils] 来避免编写`unsafe`代码。

在下面的代码中，我们将`test1`和`test2`两个对象固定至栈：

```rust
pub fn main() {
    // 在 `test1` 初始化前对其进行移动是安全的 
    let mut test1 = Test::new("test1");
    // 通过覆盖 `test1` 使其不再可达
    let mut test1 = unsafe { Pin::new_unchecked(&mut test1) };
    Test::init(test1.as_mut());

    let mut test2 = Test::new("test2");
    let mut test2 = unsafe { Pin::new_unchecked(&mut test2) };
    Test::init(test2.as_mut());

    println!("a: {}, b: {}", Test::a(test1.as_ref()), Test::b(test1.as_ref()));
    println!("a: {}, b: {}", Test::a(test2.as_ref()), Test::b(test2.as_ref()));
}
# use std::pin::Pin;
# use std::marker::PhantomPinned;
#
# #[derive(Debug)]
# struct Test {
#     a: String,
#     b: *const String,
#     _marker: PhantomPinned,
# }
#
#
# impl Test {
#     fn new(txt: &str) -> Self {
#         Test {
#             a: String::from(txt),
#             b: std::ptr::null(),
#             // This makes our type `!Unpin`
#             _marker: PhantomPinned,
#         }
#     }
#     fn init<'a>(self: Pin<&'a mut Self>) {
#         let self_ptr: *const String = &self.a;
#         let this = unsafe { self.get_unchecked_mut() };
#         this.b = self_ptr;
#     }
#
#     fn a<'a>(self: Pin<&'a Self>) -> &'a str {
#         &self.get_ref().a
#     }
#
#     fn b<'a>(self: Pin<&'a Self>) -> &'a String {
#         unsafe { &*(self.b) }
#     }
# }
```

现在如果再试图移动数据，就会得到一个编译错误：

```rust, compile_fail
pub fn main() {
    let mut test1 = Test::new("test1");
    let mut test1 = unsafe { Pin::new_unchecked(&mut test1) };
    Test::init(test1.as_mut());

    let mut test2 = Test::new("test2");
    let mut test2 = unsafe { Pin::new_unchecked(&mut test2) };
    Test::init(test2.as_mut());

    println!("a: {}, b: {}", Test::a(test1.as_ref()), Test::b(test1.as_ref()));
    std::mem::swap(test1.get_mut(), test2.get_mut());
    println!("a: {}, b: {}", Test::a(test2.as_ref()), Test::b(test2.as_ref()));
}
# use std::pin::Pin;
# use std::marker::PhantomPinned;
#
# #[derive(Debug)]
# struct Test {
#     a: String,
#     b: *const String,
#     _marker: PhantomPinned,
# }
#
#
# impl Test {
#     fn new(txt: &str) -> Self {
#         Test {
#             a: String::from(txt),
#             b: std::ptr::null(),
#             _marker: PhantomPinned, // This makes our type `!Unpin`
#         }
#     }
#     fn init<'a>(self: Pin<&'a mut Self>) {
#         let self_ptr: *const String = &self.a;
#         let this = unsafe { self.get_unchecked_mut() };
#         this.b = self_ptr;
#     }
#
#     fn a<'a>(self: Pin<&'a Self>) -> &'a str {
#         &self.get_ref().a
#     }
#
#     fn b<'a>(self: Pin<&'a Self>) -> &'a String {
#         unsafe { &*(self.b) }
#     }
# }
```

类型系统会阻止我们移动此数据。

> 需要注意的是，栈固定(_stack pinning_)依赖于我们编写`unsafe`代码时做出的保证。尽管我们知道`&'a mut T`指向的目标被以生命周期`'a`进行固定，却不能确定`&'a mut T`的数据在`'a`结束后是否被移动。如果被移动了，就违背了`Pin`的约定。
>
> 有一个很容易犯的错误，就是忘记覆盖原变量，而因此可以在释放`Pin`后移动`a mut T`的数据，就像下面这样(违背了`Pin`约定)：
>
> ```rust
> fn main() {
> 	let mut test1 = Test::new("test1");
> 	let mut test1_pin = unsafe { Pin::new_unchecked(&mut test1) };
>     
> 	Test::init(test1_pin.as_mut());
> 	drop(test1_pin);
> 	println!(r#"test1.b points to "test1": {:?}..."#, test1.b);
>     
> 	let mut test2 = Test::new("test2");
> 	mem::swap(&mut test1, &mut test2);
> 	println!("... and now it points nowhere: {:?}", test1.b);
> }
> # use std::pin::Pin;
> # use std::marker::PhantomPinned;
> # use std::mem;
> #
> # #[derive(Debug)]
> # struct Test {
> #     a: String,
> #     b: *const String,
> #     _marker: PhantomPinned,
> # }
> #
> #
> # impl Test {
> #     fn new(txt: &str) -> Self {
> #         Test {
> #             a: String::from(txt),
> #             b: std::ptr::null(),
> #             // This makes our type `!Unpin`
> #             _marker: PhantomPinned,
> #         }
> #     }
> #     fn init<'a>(self: Pin<&'a mut Self>) {
> #         let self_ptr: *const String = &self.a;
> #         let this = unsafe { self.get_unchecked_mut() };
> #         this.b = self_ptr;
> #     }
> #
> #     fn a<'a>(self: Pin<&'a Self>) -> &'a str {
> #         &self.get_ref().a
> #     }
> #
> #     fn b<'a>(self: Pin<&'a Self>) -> &'a String {
> #         unsafe { &*(self.b) }
> #     }
> # }
> ```

### 固定(_pin_)到堆上

将一个`!Unpin`类型固定至堆上会为该数据赋予一个稳定的地址，也就是说，一旦数据被固定就不再能移动了。与固定到栈相比，固定到堆会使目标在整个生命周期中都被固定住。

```rust, edition2018
use std::pin::Pin;
use std::marker::PhantomPinned;

#[derive(Debug)]
struct Test {
    a: String,
    b: *const String,
    _marker: PhantomPinned,
}

impl Test {
    fn new(txt: &str) -> Pin<Box<Self>> {
        let t = Test {
            a: String::from(txt),
            b: std::ptr::null(),
            _marker: PhantomPinned,
        };
        let mut boxed = Box::pin(t);
        let self_ptr: *const String = &boxed.as_ref().a;
        unsafe { boxed.as_mut().get_unchecked_mut().b = self_ptr };

        boxed
    }

    fn a<'a>(self: Pin<&'a Self>) -> &'a str {
        &self.get_ref().a
    }

    fn b<'a>(self: Pin<&'a Self>) -> &'a String {
        unsafe { &*(self.b) }
    }
}

pub fn main() {
    let mut test1 = Test::new("test1");
    let mut test2 = Test::new("test2");

    println!("a: {}, b: {}",test1.as_ref().a(), test1.as_ref().b());
    println!("a: {}, b: {}",test2.as_ref().a(), test2.as_ref().b());
}
```

一些函数需要其future为`Unpin`。为了在这些函数中使用`Future`或`Stream`这种非`Unpin`的类型，就必须先使用`Box::pin`(能创建一个`Pin<Box<T>>`)或`pin_utils::pin_mut!`宏(能创建一个`Pin<&mut T>`)来固定值。`Pin<Box<Fut>>`和`Pin<&mut Fut>`都能作为future使用，且都实现了`Unpin`。

举个例子：

```rust,edition2018,ignore
use pin_utils::pin_mut; // `pin_utils` crate可从crates.io获得

// 一个接受 `Future` 参数并实现了 Unpin 的函数
fn execute_unpin_future(x: impl Future<Output = ()> + Unpin) { /* ... */ }

let fut = async { /* ... */ };
execute_unpin_future(fut); // Error: `fut` does not implement `Unpin` trait

// 通过 `Box` 固定:
let fut = async { /* ... */ };
let fut = Box::pin(fut);
execute_unpin_future(fut); // OK

// 通过 `pin_mut!` 固定:
let fut = async { /* ... */ };
pin_mut!(fut);
execute_unpin_future(fut); // OK
```

## 总结

1. 如果有`T: Unpin`(默认情况下就是这样)，那么`Pin<'a, T>`与`&'a mut T>`完全等价。换句话说：`Unpin`意味着在值被固定(_pin_)时可被移动，因而`Pin`不会对此类型有任何影响。
2. 如果有`T: !Unpin`，那么获取`&mut T`必须在unsafe块中进行。
3. 标准库中大多数类型都实现了`Unpin`，而对于Rust中可能遇到的大多数类型同样如此。由`async`/`.await`生成的`Future`是个例外。
4. 可在Rust的nightly版本中通过feature flag为某个类型添加`!Unpin` bound，或者在stable版本中为类型添加`std::marker::PhantomPinned`
5. 可将数据固定至堆或栈
6. 将`!Unpin`对象固定至栈需要在`unsafe`块中进行
7. 将`!Unpin`对象固定至堆不需要`unsafe`，可使用`Box::pin`来进行。
8. 对于被固定的`T: !Unpin`类型的数据，必须要维持其不可变性，即维持其内存内容不会被无效化或被修改，_从被固定时开始，直到被释放(drop)时_。这是`Pin`约定的重点。



["Executing `Future`s and Tasks"]: ../02_execution/01_chapter.md
[the `Future` trait]: ../02_execution/02_future.md
[pin_utils]: https://docs.rs/pin-utils/