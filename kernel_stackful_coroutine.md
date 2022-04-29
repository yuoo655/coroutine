# 内核线程


# 内核线程创建

大致过程:

生成内核线程TaskControlBlock

用Arc包装线程TCB

往调度器里添加任务

设置TrapContext

```rust
pub fn kthread_create(f: fn()) {
    //创建内核线程
    let new_tcb = TaskControlBlock::create_kthread(f);
    let kernel_stack = new_tcb.get_kernel_stack();
    let new_task = Arc::new(new_tcb);

    //往调度器加任务,与用户线程放在一起调度.
    add_task(Arc::clone(&new_task));

    let new_task_inner = new_task.inner_exclusive_access();
    let new_task_trap_cx = new_task_inner.get_trap_cx();

    //设置TrapContext
    *new_task_trap_cx = TrapContext::kernel_init_context(
        f as usize,
        kernel_stack,
    );
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

Arc::downgrade(&process) 表示将Arc转为Weak. 

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


用户线程--->内核线程的切换在这里没有体现
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
            println!("THREAD {:?} STARTING", id);
            for i in 0..10 {
                println!("thread: {} counter: {}", id, i);
            }
            println!("THREAD {:?} FINISHED", id);
            kthread_stop();
        }
    );
    kthread_create( ||
        {
            let id = 2;
            println!("THREAD {:?} STARTING", id);
            for i in 0..10 {
                println!("thread: {} counter: {}", id, i);
            }
            println!("THREAD {:?} FINISHED", id);
            kthread_stop();
        }
    );
    kthread_create( ||
        {
            let id = 3;
            println!("THREAD {:?} STARTING", id);
            for i in 0..10 {
                println!("thread: {} counter: {}", id, i);
            }
            println!("THREAD {:?} FINISHED", id);
            kthread_stop();
        }
    );
}
```

演示结果
```rust
[rustsbi] RustSBI version 0.2.0-alpha.6
.______       __    __      _______.___________.  _______..______   __
|   _  \     |  |  |  |    /       |           | /       ||   _  \ |  |
|  |_)  |    |  |  |  |   |   (----`---|  |----`|   (----`|  |_)  ||  |
|      /     |  |  |  |    \   \       |  |      \   \    |   _  < |  |
|  |\  \----.|  `--'  |.----)   |      |  |  .----)   |   |  |_)  ||  |
| _| `._____| \______/ |_______/       |__|  |_______/    |______/ |__|

[rustsbi] Implementation: RustSBI-QEMU Version 0.0.2
[rustsbi-dtb] Hart count: cluster0 with 1 cores
[rustsbi] misa: RV64ACDFIMSU
[rustsbi] mideleg: ssoft, stimer, sext (0x222)
[rustsbi] medeleg: ima, ia, bkpt, la, sa, uecall, ipage, lpage, spage (0xb1ab)
[rustsbi] pmp0: 0x10000000 ..= 0x10001fff (rwx)
[rustsbi] pmp1: 0x80000000 ..= 0x8fffffff (rwx)
[rustsbi] pmp2: 0x0 ..= 0xffffffffffffff (---)
F:\Program Files\qemu-2021-02-08\qemu-system-riscv64.exe: clint: invalid write: 00000004
[rustsbi] enter supervisor 0x80200000
[kernel] Hello, world!
last 969 Physical Frames.
.text [0x80200000, 0x8021e000)
.rodata [0x8021e000, 0x80225000)
.data [0x80225000, 0x80226000)
.bss [0x80226000, 0x80437000)
mapping .text section
mapping .rodata section
mapping .data section
mapping .bss section
mapping physical memory
mapping memory-mapped registers
remap_test passed!
/**** APPS ****
cat
cmdline_args
count_lines
early_exit
exit
fantastic_text
filetest_simple
forktest
forktest2
forktest_simple
forktree
green_threads
hello_world
huge_write
infloop
initproc
matrix
mpsc_sem
phil_din_mutex
pipetest
pipe_large_test
priv_csr
priv_inst
race_adder
race_adder_arg
race_adder_atomic
race_adder_loop
race_adder_mutex_blocking
race_adder_mutex_spin
run_pipe_test
sleep
sleep_simple
stackful_coroutine
stackless_coroutine
stack_overflow
store_fault
sync_sem
test_condvar
threads
threads_arg
until_timeout
usertests
user_shell
yield
**************/
kernel_stackful_coroutine_test
THREAD 1 STARTING
thread: 1 counter: 0
thread: 1 counter: 1
thread: 1 counter: 2
thread: 1 counter: 3
thread: 1 counter: 4
thread: 1 counter: 5
thread: 1 counter: 6
thread: 1 counter: 7
thread: 1 counter: 8
thread: 1 counter: 9
THREAD 1 FINISHED
kthread do exit
exit_kthread_and_run_next
THREAD 2 STARTING
thread: 2 counter: 0
thread: 2 counter: 1
thread: 2 counter: 2
thread: 2 counter: 3
thread: 2 counter: 4
thread: 2 counter: 5
thread: 2 counter: 6
thread: 2 counter: 7
thread: 2 counter: 8
thread: 2 counter: 9
THREAD 2 FINISHED
kthread do exit
exit_kthread_and_run_next
THREAD 3 STARTING
thread: 3 counter: 0
thread: 3 counter: 1
thread: 3 counter: 2
thread: 3 counter: 3
thread: 3 counter: 4
thread: 3 counter: 5
thread: 3 counter: 6
thread: 3 counter: 7
thread: 3 counter: 8
thread: 3 counter: 9
THREAD 3 FINISHED
kthread do exit
exit_kthread_and_run_next
entering user shell
Rust user shell
>>
```
