# 用户态协程


## 协程表示

Fib是我们的协程数据结构, 其包装了一个state, 只有两种状态Halted和Running.

```rust
enum State {
    Halted,
    Running,
}

struct Fib {
    state: State,
}

impl Fib {
    fn waiter<'a>(&'a mut self) -> Waiter<'a> {
        Waiter { fib: self }
    }
}

struct Waiter<'a> {
    fib: &'a mut Fib,
}
```
## 为协程实现future

在rust中future 表示为Future trait

Future trait定义了poll方法, poll方法返回Poll<T>表示协程执行完成或未完成.

poll有两个参数: self: Pin<&mut Self> 和 cx: &mut Context

self: Pin<&mut Self>表示一个被固定在某内存地址的Self引用

cx: &mut Context 包装了Waker

poll()根据Fib的状态返回Poll::Ready(())或Pending

返回值Poll(T)只有2种状态:

1.Ready(T)

2.Pending


```rust
pub trait Future {
    type Output;
    fn poll(self: Pin<&mut Self>, cx: &mut Context) -> Poll<Self::Output>;
}

pub enum Poll<T> {
    Ready(T),
    Pending,
}

impl<'a> Future for Waiter<'a> {
    type Output = ();

    fn poll(mut self: Pin<&mut Self>, _cx: &mut Context) -> Poll<Self::Output> {
        match self.fib.state {
            State::Halted => {
                self.fib.state = State::Running;
                Poll::Ready(())
            }
            State::Running => {
                self.fib.state = State::Halted;
                Poll::Pending
            }
        }
    }
}
```

## 协程执行器实现

Executor包含fibs  fibs为一个VecDeque,里面存放的是futureobject

创建Executor. 初始化一个VecDeque

push方法往Executor添加协程

首先设置协程Fib的状态为Running. 

closure(fib) 根据trait bound会生成一个future Future<Output=()> + 'static

self.fibs.push_back(Box::pin(closure(fib))) 再把一个pin住的future添加到协程队列.

```rust
struct Executor {
    fibs: VecDeque<Pin<Box<dyn Future<Output=()>>>>,
}

impl Executor {
    fn new() -> Self {
        Executor {
            fibs: VecDeque::new(),
        }
    }

    fn push<C, F>(&mut self, closure: C)
    where
        F: Future<Output=()> + 'static,
        C: FnOnce(Fib) -> F,
    {
        let fib = Fib { state: State::Running };
        self.fibs.push_back(Box::pin(closure(fib)));
    }
}
```

## 协程执行器运行

协程执行器执行协程就是从协程队列fibs中获取一个future object, 再调用其poll方法.

当poll返回Poll::Pending时表示协程未完成,将其插回队列中

当poll返回Poll::Ready(())时表示协程完成. 根据future实现不同可以得到一个返回值Ready<T>

```rust
impl Executor {
...
    fn run(&mut self) {
        let waker = create();
        let mut context = Context::from_waker(&waker);

        while let Some(mut fib) = self.fibs.pop_front() {
            match fib.as_mut().poll(&mut context) {
                Poll::Pending => {
                    self.fibs.push_back(fib);
                },
                Poll::Ready(()) => {},
            }
        }
    }
}
```

每个future会绑定一个waker. 作用是等待事件到来时将其唤醒(可以再次被poll) 这里没有体现这一点

waker是一个胖指针,前8个字节指向data trait 对象 , 第二个8字节指向vtable trait对象

其中vtable为一个虚函数表, 规定waker必须实现clone wake wake_by_ref drop函数

```rust
pub fn create() -> Waker {
    // Safety: The waker points to a vtable with functions that do nothing. Doing
    // nothing is memory-safe.
    unsafe { Waker::from_raw(RAW_WAKER) }
}

const RAW_WAKER: RawWaker = RawWaker::new(core::ptr::null(), &VTABLE);
const VTABLE: RawWakerVTable = RawWakerVTable::new(clone, wake, wake_by_ref, drop);

unsafe fn clone(_: *const ()) -> RawWaker { RAW_WAKER }
unsafe fn wake(_: *const ()) { }
unsafe fn wake_by_ref(_: *const ()) { }
unsafe fn drop(_: *const ()) {}
```


## 测试

创建协程执行器

```rust
let mut exec = Executor::new();
```

```rust
pub fn stackless_coroutine_test() {
    let mut exec = Executor::new();
    
    for instance in 1..=3 {
        exec.push(  move |mut fib| 
            async move {
            println!("{} A", instance);
            fib.waiter().await;
            println!("{} B", instance);
            fib.waiter().await;
            println!("{} C", instance);
            fib.waiter().await;
            println!("{} D", instance);
            }
        );
    }

    println!("Running");
    exec.run();
    println!("Done");
}
```

async move 将{}内的async块 的所有权以及捕获的变量转移到 mut fib中

```rust
|mut fib| async move {}
```

再将fib变量包括其所有权转移到closure中.

```rust
move |mut fib| 
```


观察push函数可以发现它为closure实现了FnOnce方法, 表示闭包调用closure时会移除所捕获的变量的所有权,只能调用一次closure方法.

fn push<C, F>  <C, F>表示2个泛型 

where 表示泛型的约束(trait bound)

```rust
fn push<C, F>(&mut self, closure: C)
where
    F: Future<Output=()> + 'static,
    C: FnOnce(Fib) -> F,
{
    let fib = Fib { state: State::Running };
    self.fibs.push_back(Box::pin(closure(fib)));
}
```

fib.waiter() 会生成一个future Future<Output=()>

其中 .await表示一直poll该future直到返回Ready.

```rust
//async move 下 async块内部
{
    println!("{} A", instance);
    fib.waiter().await;
    println!("{} B", instance);
    fib.waiter().await;
    println!("{} C", instance);
    fib.waiter().await;
    println!("{} D", instance);
}
```

协程执行器运行

```rust
exec.run();
```

## 测试结果

```rust
Running
1 A 
2 A 
3 A 
1 B 
2 B 
3 B 
1 C 
2 C 
3 C 
1 D 
2 D 
3 D 
Done
```


## pin

通过生成器的例子来看为什么需要pin

future在rust中是由生成器来实现的.

future每一次被poll相当于生成器执行一次resume. 每次resume生成器都将执行到一个状态 

GeneratorState<Self::Yield, Self::Return>

如GeneratorA, 包含Enter, Yield1, Exit状态,

其中Yield1 包含一个自引用数据结构

borrowed引用指向上面的 to_borrow: String

如之前介绍.await时提到的, 当一个.await一个future时, 闭包相关的语法会将所捕获的变量的所有权进行转移, 这时borrowed会形成一个悬垂指针.

Box::pin 为我们提供了一个safe的方法 来将对象!Unpin到**堆**上

```rust
enum GeneratorA {
    Enter,
    Yield1 {
        to_borrow: String,
        borrowed: *const String,
    },
    Exit,
}


```

```rust
enum GeneratorA {
    Enter,
    Yield1 {
        to_borrow: String,
        borrowed: *const String,
    },
    Exit,
}

impl GeneratorA {
    fn start() -> Self {
        GeneratorA::Enter
    }
}

// This tells us that the underlying pointer is not safe to move after pinning.
// In this case, only we as implementors "feel" this, however, if someone is
// relying on our Pinned pointer this will prevent them from moving it. You need
// to enable the feature flag ` #![feature(optin_builtin_traits)]` and use the
// nightly compiler to implement `!Unpin`. Normally, you would use
// `std::marker::PhantomPinned` to indicate that the struct is `!Unpin`.
impl !Unpin for GeneratorA { }

impl Generator for GeneratorA {
    type Yield = usize;
    type Return = ();
    fn resume(self: Pin<&mut Self>) -> GeneratorState<Self::Yield, Self::Return> {
        // lets us get ownership over current state
        let this = unsafe { self.get_unchecked_mut() };
            match this {
            GeneratorA::Enter => {
                let to_borrow = String::from("Hello");
                let borrowed = &to_borrow;
                let res = borrowed.len();
                *this = GeneratorA::Yield1 {to_borrow, borrowed: std::ptr::null()};

                // Trick to actually get a self reference. We can't reference
                // the `String` earlier since these references will point to the
                // location in this stack frame which will not be valid anymore
                // when this function returns.
                if let GeneratorA::Yield1 {to_borrow, borrowed} = this {
                    *borrowed = to_borrow;
                }

                GeneratorState::Yielded(res)
            }

            GeneratorA::Yield1 {borrowed, ..} => {
                let borrowed: &String = unsafe {&**borrowed};
                println!("{} world", borrowed);
                *this = GeneratorA::Exit;
                GeneratorState::Complete(())
            }
            GeneratorA::Exit => panic!("Can't advance an exited generator!"),
        }
    }
}

```

更详细的解释:

https://rust-lang.github.io/async-book/04_pinning/01_chapter.html

https://os.phil-opp.com/async-await/#pinning

https://course.rs/advance/circle-self-ref/self-referential.html