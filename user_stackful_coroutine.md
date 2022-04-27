# 用户态线程

## 总体设计

生成2个线程.每个线程打印一行信息之后便让出yield_thread,控制流转移到线程调度器(运行时)上,由运行时来切换到下一个线程运行,下一个线程打印一行信息之后也马上让出yield_thread进入运行时,直到所有线程都执行完毕.

整个过程包含了线程创建,运行,切换,以及简单的runtime调度线程的过程,过程中可以看到线程状态的变化.

### 线程数据结构

线程数据结构,包含线程id,执行中使用的栈,上下文信息,state来表示线程状态
```rust
struct Thread {
    id: usize,
    stack: Vec<u8>,
    ctx: ThreadContext,
    state: State,
}
```

### 上下文信息与切换过程

上下文信息主要是寄存器集合,riscv指令集规定了函数调用时**调用者保存的寄存器(caller saved)**以及**被调用者保存寄存器(callee saved)**, 我们需要被调用者保存寄存器.

#[repr(C)] 表示对这个结构体按 C 语言标准 进行内存布局，也就是从起始地址开始，按字段的定义顺序依次排列。

```rust
#[repr(C)]
struct ThreadContext {
    //riscv 被调用者保存寄存器
    // Register   ABI name        Description
    // x1         ra              return address
    // x2         sp              stack pointer
    // x8         s0,fp           saved register/frame pointer 
    // x18-x27    s2-s11          saved registers     
    // nra                        new return address

    ra: u64,        // 0*8(sp)  x1  
    sp: u64,        // 1*8(sp)  x2
    s0: u64,        // 2*8(sp)  x8
    s1: u64,        // 3*8(sp)  x9
    s2: u64,        // 4*8(sp)  x18 
    s3: u64,        // 5*8(sp)  x19
    s4: u64,        // 6*8(sp)  x20
    s5: u64,        // 7*8(sp)  x21
    s6: u64,        // 8*8(sp)  x22
    s7: u64,        // 9*8(sp)  x23
    s8: u64,        // 10*8(sp) x24
    s9: u64,        // 11*8(sp) x25
    s10: u64,       // 12*8(sp) x26
    s11: u64,       // 13*8(sp) x27

    nra: u64,       // 14*8(sp)  new return address
}
```

创建线程时,ra中保存的是runtime运行时的地址, nra保存的是要执行函数的地址.
```rust
    pub fn spawn(&mut self, f: fn()) {
        let available = self
            .threads
            .iter_mut()
            .find(|t| t.state == State::Available)
            .expect("no available thread.");

        let size = available.stack.len();
        unsafe {
            let s_ptr = available.stack.as_mut_ptr().offset(size as isize);
            let s_ptr = (s_ptr as usize & !7) as *mut u8;


            available.ctx.ra = guard as u64;
            available.ctx.nra = f as u64;
            available.ctx.sp = s_ptr.offset(-32) as u64;
        }
        available.state = State::Ready;
    }
```

```rust
#[naked]
#[no_mangle]
unsafe fn switch(old: *mut ThreadContext, new: *const ThreadContext) {
    // a0: old, a1: new
    asm!("
        sd x1, 0x00(a0)
        sd x2, 0x08(a0)
        sd x8, 0x10(a0)
        sd x9, 0x18(a0)
        sd x18, 0x20(a0)
        sd x19, 0x28(a0)
        sd x20, 0x30(a0)
        sd x21, 0x38(a0)
        sd x22, 0x40(a0)
        sd x23, 0x48(a0)
        sd x24, 0x50(a0)
        sd x25, 0x58(a0)
        sd x26, 0x60(a0)
        sd x27, 0x68(a0)
        sd x1, 0x70(a0)

        ld x1, 0x00(a1)
        ld x2, 0x08(a1)
        ld x8, 0x10(a1)
        ld x9, 0x18(a1)
        ld x18, 0x20(a1)
        ld x19, 0x28(a1)
        ld x20, 0x30(a1)
        ld x21, 0x38(a1)
        ld x22, 0x40(a1)
        ld x23, 0x48(a1)
        ld x24, 0x50(a1)
        ld x25, 0x58(a1)
        ld x26, 0x60(a1)
        ld x27, 0x68(a1)
        ld t0, 0x70(a1)
        
        jr t0
        ", options(noreturn)
    );
}
```


结合线程的汇编代码来看上下文信息.

根据riscv函数调用约定, 第一个和第二个参数分别放在a0, a1寄存器中.

unsafe fn switch(old: *mut ThreadContext, new: *const ThreadContext){}

那么a0寄存器中为旧线程的上下文地址, a1寄存器中为新线程的上下文地址


以 sd x1, 0x00(a0)为例

该汇编作用为: **把当前寄存器x1(abi名称ra)中的值, 保存到基地址为a0处偏移量为0x00处的内存中**去. 也就是ThreadContext中的第一个字段ra
```rust
接下来是按顺序把各寄存器中的值保存到ThreadContext相应字段中去
sd x2, 0x08(a0)
sd x8, 0x10(a0)
sd x9, 0x18(a0)
sd x18, 0x20(a0)
sd x19, 0x28(a0)
sd x20, 0x30(a0)
sd x21, 0x38(a0)
sd x22, 0x40(a0)
sd x23, 0x48(a0)
sd x24, 0x50(a0)
sd x25, 0x58(a0)
sd x26, 0x60(a0)
sd x27, 0x68(a0)

这里要注意的是ra被保存了2次
sd x1, 0x70(a0)

接下来是把新线程的上下文信息加载到当前寄存器中
ld x1, 0x00(a1)
ld x2, 0x08(a1)
ld x8, 0x10(a1)
ld x9, 0x18(a1)
ld x18, 0x20(a1)
ld x19, 0x28(a1)
ld x20, 0x30(a1)
ld x21, 0x38(a1)
ld x22, 0x40(a1)
ld x23, 0x48(a1)
ld x24, 0x50(a1)
ld x25, 0x58(a1)
ld x26, 0x60(a1)
ld x27, 0x68(a1)
ld t0, 0x70(a1)
```
最后跳转过去

jr t0

jr t0 实际为 jalr x0, 0(t0)       

ret = jr ra = jalr x0, 0(ra)

跳转到t0中值所指定的位置,不保存返回地址

available.ctx.nra = f as u64;  //new return address

创建线程时, nra字段中保存的是线程要执行函数的地址.这里跳转过去线程就开始执行函数了.

一个线程创建之后它的上下文信息中ra的值为t_return函数的地址. nra的值为实际需要执行函数的地址. jr t0跳转过去执行函数之后.它上下文中的ra会变成执行过程中的pc. 但是nra的值是不会发生变化的.

**当它不是初次运行被切换时候. 我们不再需要保存nra**. 所以就有了前面的sd x1, 0x70(a0). 用当前ra的值覆盖掉之前的nra. 这样ld t0, 0x70(a1)就会加载正确的pc进来执行(而不是再跳转到nra从头执行)




### 测试

```rust
pub fn stackful_coroutine_test() {
    let mut runtime = Runtime::new();
    runtime.init();
    runtime.spawn(|| {
        println!("THREAD 1 STARTING");
        let id = 1;
        for i in 0..10 {
            println!("thread: {} counter: {}", id, i);
            yield_thread();
        }
        println!("THREAD 1 FINISHED");
    });
    runtime.spawn(|| {
        println!("THREAD 2 STARTING");
        let id = 2;
        for i in 0..15 {
            println!("thread: {} counter: {}", id, i);
            yield_thread();
        }
        println!("THREAD 2 FINISHED");
    });
    runtime.run();
}
```

### 运行时创建
let mut runtime = Runtime::new();

运行时包含一个threads数组存放线程,以及current表示当前正在执行的线程的id

创建一个线程运行时,主要过程:
1.创建一个数组threads,里面存放Thread数据结构 
2.创建一个id为0的主线程,并把线程状态置为Running
3.创建多个状态为Available的线程,加入threads中
4.将current置为0.表示当前执行的是主线程

runtime.init();

将Runtime自身的地址指针赋值给之前声明的全局可变变量 static mut RUNTIME: usize = 0;


```rust
impl Runtime {
    pub fn new() -> Self {
        let base_thread = Thread {
            id: 0,
            stack: vec![0_u8; DEFAULT_STACK_SIZE],
            ctx: ThreadContext::default(),
            state: State::Running,
        };

        let mut threads = vec![base_thread];
        let mut available_threads: Vec<Thread> = (1..MAX_THREADS).map(|i| Thread::new(i)).collect();
        threads.append(&mut available_threads);

        Runtime {
            threads,
            current: 0,
        }
    }

    pub fn init(&self) {
        unsafe {
            let r_ptr: *const Runtime = self;
            RUNTIME = r_ptr as usize;
        }
    }
    ....
}
```



### 线程创建
寻找一个可用的线程控制块

填写线程上下文信息

把栈指针设置为预先开辟好的栈地址.

将线程状态设置为Ready
```rust
pub fn spawn(&mut self, f: fn()) {
    let available = self
        .threads
        .iter_mut()
        .find(|t| t.state == State::Available)
        .expect("no available thread.");

    let size = available.stack.len();

    unsafe {
        let s_ptr = available.stack.as_mut_ptr().offset(size as isize);
        let s_ptr = (s_ptr as usize & !7) as *mut u8;
        available.ctx.ra = guard as u64;
        available.ctx.nra = f as u64;
        available.ctx.sp = s_ptr.offset(-32) as u64;
    }
    available.state = State::Ready;
}
```


### 线程执行

t_yield遍历所有线程, 看看是否有线程处于 Ready 状态, Ready 表明它已准备好恢复执行.

然后我们调用 switch 来保存当前上下文（旧上下文）并将新上下文加载到 CPU 中. 新上下文要么是新任务,要么是 CPU 在现有任务上恢复工作所需的所有信息.

所有的线程都执行完毕后,会回到 runtime.run() 函数, 通过 crate::exit(0) 来退出该应用进程


```rust
impl Runtime {
...
    pub fn run(&mut self){
        while self.t_yield() {}
        
        println!("all finished");
        crate::exit(0);
    }

    fn t_return(&mut self) {
        if self.current != 0 {
            self.threads[self.current].state = State::Available;
            self.t_yield();
        }
    }
    #[inline(never)]
    fn t_yield(&mut self) -> bool {
        let mut pos = self.current;
        while self.threads[pos].state != State::Ready {
            pos += 1;
            if pos == self.threads.len() {
                pos = 0;
            }
            if pos == self.current {
                return false;
            }
        }

        if self.threads[self.current].state != State::Available {
            self.threads[self.current].state = State::Ready;
        }

        self.threads[pos].state = State::Running;
        let old_pos = self.current;
        self.current = pos;

        unsafe {
            switch1(&mut self.threads[old_pos].ctx, &self.threads[pos].ctx);
        }
        self.threads.len() > 0
    }
}
pub fn stackful_coroutine_test() {
....
    runtime.run();
}
```





### 测试结果

生成2个线程.每个线程打印一行信息之后便让出yield_thread,控制流转移到线程调度器(运行时)上,由运行时来切换到下一个线程运行,下一个线程打印一行信息之后也马上让出yield_thread进入运行时,直到所有线程都执行完毕.

```rust
>> stackful_coroutine
THREAD 1 STARTING
thread: 1 counter: 0
THREAD 2 STARTING
thread: 2 counter: 0
thread: 1 counter: 1
thread: 2 counter: 1
thread: 1 counter: 2
thread: 2 counter: 2
thread: 1 counter: 3
thread: 2 counter: 3
thread: 1 counter: 4
thread: 2 counter: 4
thread: 1 counter: 5
thread: 2 counter: 5
thread: 1 counter: 6
thread: 2 counter: 6
thread: 1 counter: 7
thread: 2 counter: 7
thread: 1 counter: 8
thread: 2 counter: 8
thread: 1 counter: 9
thread: 2 counter: 9
THREAD 1 FINISHED
thread: 2 counter: 10
thread: 2 counter: 11
thread: 2 counter: 12
thread: 2 counter: 13
thread: 2 counter: 14
THREAD 2 FINISHED
all finished
>> 
```