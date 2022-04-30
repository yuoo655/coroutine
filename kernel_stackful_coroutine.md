# 内核线程


# 内核线程创建

大致过程:

生成内核线程TaskControlBlock

用Arc包装线程TCB

往调度器里添加任务

```rust
pub fn kthread_create(f: fn()) {
    //创建内核线程
    let new_tcb = TaskControlBlock::create_kthread(f);
    let kernel_stack = new_tcb.get_kernel_stack();
    let new_task = Arc::new(new_tcb);

    //往调度器加任务,与用户线程放在一起调度.
    add_task(Arc::clone(&new_task));
}
```

# 生成内核线程数据结构

线程TaskControlBlock大致需要包含以下信息

1.执行过程中用的栈

2.线程上下文信息

3.线程的状态

4.所属进程(对于内核线程来说这个不需要, 但是下面还是设置了一遍)

所以生成TaskControlBlock大致过程为:

1.为线程设置一个栈

2.初始化一个上下文信息

3.设置线程状态

4.设置所属进程

Arc::downgrade(&process) 表示将Arc转为Weak弱引用计数.

rCore-Tutorial中的相关解释:

注意我们在维护父子进程关系的时候大量用到了引用计数 Arc/Weak 。进程控制块的本体是被放到内核堆上面的，对于它的一切访问都是通过智能指针 Arc/Weak 来进行的，这样是便于建立父子进程的双向链接关系（避免仅基于 Arc 形成环状链接关系）。当且仅当智能指针 Arc 的引用计数变为 0 的时候，进程控制块以及被绑定到它上面的各类资源才会被回收。子进程的进程控制块并不会被直接放到父进程控制块中，因为子进程完全有可能在父进程退出后仍然存在。

```rust
impl TaskControlBlock {
...
    pub fn create_kthread(f: fn()) -> Self{
        use crate::mm::{KERNEL_SPACE, PhysPageNum, VirtAddr, PhysAddr};

        //创建栈
        let kernelstack = crate::task::id::KStack::new();
        let kstack_top = kernelstack.top();

        //初始化线程上下文信息
        let mut context = TaskContext::kthread_init();

        //获取线程上下文所在的物理页帧号, 我们的内核中va = pa 所以可以不用查页表直接得到物理页帧号
        let context_addr = &context as *const TaskContext as usize;
        let pa = PhysAddr::from(context_addr);
        let context_ppn = pa.floor();

        //设置ra为线程要执行函数的地址
        //设置sp为上面生成好的栈的栈顶
        context.ra = f as usize;
        context.sp = kstack_top;
        
        //设置线程所属进程
        let process = ProcessControlBlock::kernel_process();
        let process = Arc::downgrade(&process);

        Self {
            process,
            kstack : kstack_top,
            inner: unsafe {
                UPSafeCell::new(TaskControlBlockInner {
                    res: None,
                    trap_cx_ppn: context_ppn,
                    task_cx: context,
                    task_status: TaskStatus::Ready,
                    exit_code: None,
                })
            },
        }
    }
}
```


# 生成TaskControlBlock所需的ProcessControlBlock

除了memory_set为生成KERNEL_SPACE的一个拷贝. 其他过程和普通用户进程生成ProcessControlBlock一样


注: linux中的内核线程中将页表设置为NULL. 方便了后续在context_switch时候分情况做不同预处理, 最后用一段共同的switch汇编来完成切换. 

```rust
pub fn kernel_process() -> Arc<Self>{
    let memory_set = MemorySet::kernel_copy();
    let process = Arc::new(
        ProcessControlBlock {
            pid: super::pid_alloc(),
            inner: unsafe {
                UPSafeCell::new(
                    ProcessControlBlockInner {
                    is_zombie: false,
                    memory_set: memory_set,
                    parent: None,
                    children: Vec::new(),
                    exit_code: 0,
                    fd_table: Vec::new(),
                    signals: SignalFlags::empty(),
                    tasks: Vec::new(),
                    task_res_allocator: RecycleAllocator::new(),
                    mutex_list: Vec::new(),
                    semaphore_list: Vec::new(),
                    condvar_list: Vec::new(),
                })
           },
    });
    process
}
pub fn kernel_copy() -> Self {
    let areas = KERNEL_SPACE.exclusive_access().areas.clone();
    Self {
        page_table: PageTable::from_token(kernel_token()),
        areas: areas,
    }
}
pub fn kernel_token() -> usize {
    KERNEL_SPACE.exclusive_access().token()
}
```

# 内核线程退出

内核线程不会自己退出.这里让它退出的方法是,把切换到调度循环中去.且不再保存这个线程的上下文)

```rust
pub fn kthread_stop(){
    do_exit();
}
#[no_mangle]
pub fn do_exit(){
    exit_kthread_and_run_next(0);
    panic!("Unreachable in sys_exit!");
}

pub fn exit_kthread_and_run_next(exit_code: i32) {
    // we do not have to save task context
    let mut _unused = TaskContext::zero_init();
    schedule(&mut _unused as *mut _);
}
```


# 测试
将内核线程与用户程序一起添加到调度队列中.再开始运行.

这里能体现的切换:

内核线程--->内核线程

内核线程--->用户线程

用户线程--->用户线程

用户线程--->内核线程

创建了3个内核线程

kernel thread1 会打印几行信息之后便退出

kernel thread2/3 会打印一行信息之后就会主动切换到下一个线程,可能是内核线程也可能是用户进程.

在shell运行起来的时候,可以输入要运行的用户程序,也可以不断按回车来切换到内核线程.

```rust
pub fn rust_main() -> ! {
    clear_bss();
    println!("[kernel] Hello, world!");
    mm::init();
    mm::remap_test();
    trap::init();
    trap::enable_timer_interrupt();
    timer::set_next_trigger();
    fs::list_apps();
    coroutine::kernel_stackful_coroutine_test();
    task::add_initproc();
    task::run_tasks();
    panic!("Unreachable in rust_main!");
}

pub fn kernel_stackful_coroutine_test() {
    println!("kernel_stackful_coroutine_test");
    kthread_create( ||
        {
            let id = 1;
            println!("kernel thread {:?} STARTING", id);
            for i in 0..10 {
                println!("kernel thread: {} counter: {}", id, i);
            }
            println!("kernel thread {:?} FINISHED", id);
            kthread_stop();
        }
    );
    kthread_create( ||
        {
            let id = 2;
            println!("kernel thread {:?} STARTING", 2);
            for i in 0..10 {
                println!("kernel thread: {} counter: {}", 2, i);
                kthread_yield();
            }
            println!("kernel thread {:?} FINISHED", 2);
            kthread_stop();
        }
    );
    kthread_create( ||
        {
            let id = 3;
            println!("kernel thread {:?} STARTING", 3);
            for i in 0..10 {
                println!("kernel thread: {} counter: {}", 3, i);
                kthread_yield();
            }
            println!("kernel thread {:?} FINISHED", 3);
            kthread_stop();
        }
    );
}
```

演示结果
```rust
**************/
kernel_stackful_coroutine_test
kthread_create
kstack_alloc  kstack_bottom: 0xffffffffffffd000, kstack_top: 0xfffffffffffff000
context ppn :PPN:0x80234
kstack_drop  kstack_bottom: PA:0xffffffffffd000
kthread trap context addr:0x80207c7a
kthread cx: TrapContext { x: [0, 0, 80240000, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0], sstatus: Sstatus { bits: 100 }, sepc: 80207c7a, kernel_satp: 8000000000080436, kernel_sp: 80240000, trap_handler: 80205eee }
kthread_create
kstack_alloc  kstack_bottom: 0xffffffffffffa000, kstack_top: 0xffffffffffffc000
context ppn :PPN:0x80234
kstack_drop  kstack_bottom: PA:0xffffffffffa000
kthread trap context addr:0x80207918
kthread cx: TrapContext { x: [0, 0, 80240000, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0], sstatus: Sstatus { bits: 100 }, sepc: 80207918, kernel_satp: 8000000000080436, kernel_sp: 80240000, trap_handler: 80205eee }
kstack_alloc  kstack_bottom: 0xffffffffffff7000, kstack_top: 0xffffffffffff9000
kernel stack top: 0xffffffffffff9000
kstack_drop  kstack_bottom: PA:0xffffffffff7000
app_init_context :0x10000
kernel thread 1 STARTING
kernel thread: 1 counter: 0
kernel thread: 1 counter: 1
kernel thread: 1 counter: 2
kernel thread: 1 counter: 3
kernel thread: 1 counter: 4
kernel thread: 1 counter: 5
kernel thread: 1 counter: 6
kernel thread: 1 counter: 7
kernel thread: 1 counter: 8
kernel thread: 1 counter: 9
kernel thread 1 FINISHED
kthread do exit
exit_kthread_and_run_next
kernel thread 2 STARTING
kernel thread: 2 counter: 0
kernel thread 3 STARTING
kernel thread: 3 counter: 0
kernel thread: 2 counter: 1
kernel thread: 3 counter: 1
kstack_alloc  kstack_bottom: 0xffffffffffff1000, kstack_top: 0xffffffffffff3000
kernel stack top: 0xffffffffffff3000
kstack_drop  kstack_bottom: PA:0xffffffffff1000
kernel thread: 2 counter: 2
kernel thread: 3 counter: 2
app_init_context :0x10000
kernel thread: 2 counter: 3
kernel thread: 3 counter: 3
Rust user shell
>> kernel thread: 2 counter: 4
kernel thread: 3 counter: 4

>> kernel thread: 2 counter: 5
kernel thread: 3 counter: 5

>> kernel thread: 2 counter: 6
kernel thread: 3 counter: 6

>> kernel thread: 2 counter: 7
kernel thread: 3 counter: 7

>> kernel thread: 2 counter: 8
kernel thread: 3 counter: 8

>> kernel thread: 2 counter: 9
kernel thread: 3 counter: 9

>> kernel thread 2 FINISHED
kthread do exit
exit_kthread_and_run_next
kernel thread 3 FINISHED
kthread do exit
exit_kthread_and_run_next

>>
>>
>>
>>
```
