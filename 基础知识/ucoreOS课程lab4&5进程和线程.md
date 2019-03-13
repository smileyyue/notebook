# 进程和线程

[TOC]

## 进程

### 基础概念

进程的定义：进程是一个具有一定**独立功能**的程序在一个**数据集合**上的一次**动态执行**过程

进程的组成：进程包含了一个正在执行程序的所有状态信息，这些状态信息构成进程控制块。

- 代码
- 数据
- 状态寄存器（CR0,IP）
- 通用寄存器（AX,BX,CX）
- 进程占用的系统资源（打开的文件，分配到内存）

进程的特点：

- 动态性：可动态创建、结束
- 并发性：可被操作系统独立调度并占CPU运行
- 独立性：不同进程互不影响
- 制约性：因共享数据、资源、进程空间同步问题而产生的互相制约

程序：静态可执行文件

进程：包括程序、数据（占用的资源）、进程控制块

### 进程控制块(PCB, Process Control Block)

定义：进程控制块是操作系统管理控制进程运行所用的集合

特点：一个进程对应一个PCB，创建时同时创建PCB，结束时回收PCB，PCB保存了进程的基本情况和运行状态

进程控制块的内容：

- 进程标识信息：标识该进程块，PID等
- 处理机(CPU)现场保存：从进程地址空间中抽出一些关键信息存到PCB中（例如：PC,SP,PID,UID，其他寄存器，调度优先级，打开文件列表）
- 进程控制信息：
  - 调度和状态信息：调度进程和处理机使用情况
  - 进程间通信信息：进程间通信相关的各种标识
  - 存储管理信息：指向进程管理存储空间的数据结构
  - 进程所用资源：进程使用的系统资源，如打开的文件等
  - 有关数据结构连接信息：各种链表，进程队列

PCB的组织方式：可以用多种不同数据结构组织。

- 链表：同一状态PCB链接成一个链表，如：就绪链表、阻塞链表
- 索引表：同一状态PCB归入一个索引表，由索引指向PCB，如：就绪索引表、阻塞索引表

### 进程状态

#### 进程生命周期划分

不同操作系统划分方式不同，一般包括：进程创建，进程执行，进程等待，进程抢占，进程唤醒，进程结束

进程创建：

- 进程被创建的情况：
  - 系统初始化时
  - 用户请求创建
  - 正在执行的进程执行了创建进程的系统调用（系统提供创建进程的API）

进程执行：进程创建结束后把进程放到就绪队列，根据CPU**调度算法**，调入进程执行。

进程阻塞：

- 进程被阻塞的情况：
  - 请求并等待的系统服务无法马上完成
  - 启动某种操作无法马上完成
  - 需要的数据（资源）没有到达
- 只有进程自身才能知道合适需要等待某种时间的发生，别的进程或操作系统不知道他被阻塞

进程抢占：

- 进程被抢占的情况：
  - 高优先级进程就绪
  - 进程当前执行时间用完

进程唤醒：

- 进程被唤醒的情况：
  - 阻塞进程的资源满足
  - 阻塞进程等待的事件到达
- 进程只能被别的进程或者操作系统唤醒，自己不知道合适被唤醒

进程退出：

- 进程结束情况：
  - 正常退出（自愿退出）return 0
  - 错误推出（自愿退出）return -1
  - 致命错误（强制性）fatal error
  - 被其他进程kill（强制性）sigalpipe:ctrl-c

#### 三状态进程模型

三个主要状态：运行状态（Running）、就绪状态（Ready）、等待状态（或称阻塞状态，Blocked）

两个次要状态：创建状态（New）、结束状态（Exit），这两个状态只进一次

状态变迁：（格式：状态变迁：状态变迁原因）

- New->Ready：创建后进入
- Ready->Running：调度后执行
- Running->Ready：时间片用完
- Running->Blocked：等待资源或者事件
- Blocked->Ready：资源或事件到达
- Running->Exit：执行完毕

#### 进程挂起

前面讲的都是和CPU相关的进程状态，而进程挂起则是与存储相关的进程状态。

进程挂起：处在挂起状态的进程映像在磁盘上，目的是减少进程占用的内存。

外存中的进程状态：就绪挂起和等待挂起

- 等待挂起状态（Blocked-suspend）：进程在外存并等待某事件出现
- 就绪挂起状态（Ready-suspend）：进程在外存中，但只要调入内存就能运行，进入这个状态的原因：内存空间不够、进程优先级不够高。

状态变迁：（格式：状态变迁：状态变迁原因）

- 挂起（Suspend）：把一个进程从内存转到外存
  - 等待到等待挂起：没有进程处于就绪状态，或者就绪的进程需要更多内存空间。
  - 就绪到就绪挂起：由高优先级等待（系统认为会很快就绪的）进程和低优先级就绪进程，为了给高优先级进程足够内存空间，将低优先级进程挂起。
  - 运行到就绪挂起：对抢先式分时系统，有高优先级等待挂起进程因事件出现而进入内存但空间不足，就会把当前运行的进程变为就绪挂起。
  - 等待挂起到就绪挂起：事件出现
- 激活：把一个进程从外存转到内存
  - 就绪挂起到就绪：没有就绪进程或该就绪挂起进程优先级高于其他内存中就绪进程
  - 等待挂起到等待：前面进程释放大量空间，且该等待挂起进程优先级较高

状态队列：

- 操作系统维护一组队列，表示进程中所有进程的当前状态
- 不同队列表示不同状态：就绪队列、各种等待状态、僵尸队列（收尾即将退出的进程），可能有多个等待队列（比如根据设备分，每个设备一个队列）
- 根据进程状态不同将PCB加入相应队列

## 线程

### 基础概念

线程：进程内部的一类实体，满足并发性、共享地址空间的特性。

定义：线程是进程的一部分，描述指令流执行状态，是进程中**指令执行流**的最小单元，是CPU**调度的基本单位**。

> 即加入线程后，指令执行流的最小单位和CPU调度的基本单位从进程变成了线程。
>
> 执行流从PCB变成了TCB。
>
> TCB：线程控制块，Thread Control Block
>
> 资源仍旧由PCB管理，但执行流分成多个TCB管理，即代码、数据、资源（打开的文件等）还是共享的，由PCB管理，寄存器、堆栈不再共享，而是分别由各自的TCB管理，每个TCB有自己的寄存器、堆栈。
>
> 线程=进程-共享资源
>
> 一个线程崩溃会导致进程所有线程崩溃

进程和线程的关系

- 进程是资源分配单位，线程是处理机（CPU）调度单位。
- 进程拥有完整的资源，线程只有指令流执行必要的资源，如寄存器和栈。
- 线程有就绪、等待、运行三种基本状态和状态间的转换关系。

线程减少了并发执行的事件空间开销：线程创建终止时间短，各线程切换时间短，各线程间通信方便。

### 线程的三种实现方式

#### 用户线程

在用户空间实现，由函数库实现。例：POSIX Pthreads， Mach C-threads， Solaris threads。

用户线程通过一组用户级的线程库函数来完成线程的管理，包括线程的创建、终止、同步和调度，操作系统还是只管PCB，不知道线程的存在。

特征：

- 不依赖操作系统的内核，可用于不支持线程的多进程操作系统
- 同一进程中线程切换速度快（不经过操作系统，没有用户态内核态的特权级变化）
- 用户可以自己写线程调度算法
- 一个线程阻塞，则整个进程阻塞
- 不支持线程抢占，除非当前线程主动放弃，否则其他线程无法抢占（没操作系统去调度）
- 只能按进程分配CPU时间片，线程越多每个线程时间片越少。

#### 内核线程

在内核中实现，用户线程出现后效果好，然后被加入操作系统，成为其中一个模块。Windows，Solaris，Linux。

由内核通过系统调用实现的线程机制，由内核完成线程的创建、终止和管理。

特征：

- 内核维护PCB和TCB
- 线程执行系统调用被阻塞不影响其他进程
- 线程的创建、终止、切换开销较大
- 以线程为单位分配CPU时间，多线程进程可获得更多CPU时间

#### 轻量级线程（轻权线程）

在内核中实现，支持用户线程。Solaris（LightWeight Process）

内核支持的用户线程，一个进程可有多个轻权进程，每个轻权进程由一个单独的内核线程来支持，内核线程和轻权进程可绑定或者解绑，通过用户程序决定调度策略（决定方式是让哪个轻权进程运行就将其绑定到内核线程上）。内核线程构成一个内核线程池，等待绑定或解绑。

效果不理想，机制过于复杂，优点没实现。

## 进程控制

进程切换（又称上下文切换）：暂停当前Running进程到其他状态，调度其他进程从Ready到Running。

切换要求：切换前保存这个进程上下文，切换后恢复下一个进程上下文，切换速度要快（汇编实现）。

进程生命周期信息：寄存器(PC,SP,...)，CPU状态，内存地址空间（各进程所用地址空间不重叠，所以大部分不用保存，仅保存一些管理结构体信息）。

### PCB数据结构

实际linux、windows的PCB数据结构很复杂，里面东西很多，ucore的比较简化。

ucore的PCB结构为`struct proc_struct`：

``` c
struct proc_struct {
    enum proc_state state;                      // Process state
    int pid;                                    // Process ID
    int runs;                                   // the running times of Proces
    uintptr_t kstack;                           // Process kernel stack
    volatile bool need_resched;                 // bool value: need to be rescheduled to release CPU?
    struct proc_struct *parent;                 // the parent process
    struct mm_struct *mm;                       // Process's memory management field
    struct context context;                     // Switch here to run process
    struct trapframe *tf;                       // Trap frame for current interrupt
    uintptr_t cr3;                              // CR3 register: the base addr of Page Directroy Table(PDT)
    uint32_t flags;                             // Process flag
    char name[PROC_NAME_LEN + 1];               // Process name
    list_entry_t list_link;                     // Process link list 
    list_entry_t hash_link;                     // Process hash list
};
```

标识信息：

- `char name[PROC_NAME_LEN + 1]`：可执行文件进程名
- `int pid`：进程id，另有tid：线程id，gid：组id
- `int runs`：
- `struct proc_struct *parent`：父进程指针

状态信息：

- `uint32_t flags`：状态寄存器相关信息
- `uintptr_t cr3`：进程一级页表起始地址
- `enum proc_state state`：进程状态
- `volatile bool need_resched`：是否允许调度

占用资源信息：

- `uintptr_t kstack`：占用的内核堆栈
- `struct mm_struct *mm`：占用的存储资源

保存现场信息：

- `struct trapframe *tf`：中断保护现场，调用栈数据结构
- `struct context context`：上下文现场

状态队列指针：描述当前PCB位于哪个队列当中

- `list_entry_t list_link`：进程链表
- `list_entry_t hash_link`：进程哈希表，链表长的话加一级哈希队列，每个队列里哈希值相同的在组成相应的自己的队列

### 进程/线程切换流程

不论哪种流程，都会进到sched.c::schedule()函数中

流程举例：

1. 开始调度：proc.c::cpu_idle()->sched.c::schedule()
2. 清除调度标志：schedule()
3. 查找就绪进程：schedule()
4. 修改进程状态：schedule()->proc.c::proc_run()
5. 进程切换：switch_to()

switch_to()实现：process/switch.S，汇编实现，不同CPU，对应具体实现都不同。

### 进程创建

#### 系统调用函数介绍

- Windows API：`CreateProcess()`

- Unix：`fork/exec`

  - fork()把一个进程复制成两个进程：`parent(old PID), child(new PID)`，父进程返回子进程ID，子进程返回0。fork后PCB也复制一份，该进程的数据也都会复制（变量在复制后继续执行时在复制值的基础上继续运算），占用的资源则共享（都打开某个文件）。

  - exec()用新进程来重写当前进程，PID不变。exec后PCB被重载成新程序的。

    举例：

    ``` c
    int main(){
        int pid;
        pid = fork();	//在这之前只有一个进程，这句之后变成两个进程并且程序指针都在这里往下执行
        if(pid == 0){	//上一行fork后变成两个进程给，根据fork返回值不同判断在哪个进程里
            //子进程执行的代码
            exec(filename, arg1, arg2, ...); //用一个新的可执行文件filename替代当前进程
        }
        else {
            //父进程执行的代码
        }
    }
    ```

  - ucore do_fork中需要关注的部分：copy_mm, copy_thread

#### 空闲进程(idle process)的创建

系统刚创建时和系统中所有进程都执行完毕后，会一直执行空闲进程。

ucore中空闲进程(idleproc)的创建：

1. idleproc：proc_init()
2. 分配idleproc的资源：alloc_proc()->kmalloc()
3. 初始化idleproc的PCB：  alloc_proc()
4. 初始化idleproc：proc_init()

#### 内核线程的创建

ucore中内核线程创建也是用do_fork函数（给不同的参数决定创建进程还是线程）

创建流程：

1. initproc()：proc_init()
2. 初始化trapframe：kernel_thread()tf -> do_fork() -> copy_thread()，只拷贝线程所需信息，而非整个PCB
3. 初始化PCB,initproc：alloc_proc()
4. 初始化内核堆栈：setup_stack()
5. 内存共享：copy_stack()
6. 把initproc放到就绪队列中
7. 调度唤醒initproc

#### fork的开销与减少开销

fork开销：

- 对子进程分配内存空间
- 复制父进程内存和CPU寄存器到子进程

99%的情况里，调用fork后就调用了exec

- fork中复制内存没用
- 子进程可能关掉打开的文件和链接
- 所以可以结合fork和exec，省掉复制父进程这一步的开销，让子进程几乎立即调用exec()

windows：提供一个系统调用，包含fork和exec的功能，实现创建和加载，但不复制

unix：

- 早期使用vfork()，创建进程不再创建一个同样的内存映像，有时称为轻量级fork()
- 现在使用写时复制技术（COW，Copy on Write）技术，进程创建时不复制，后面要用时才延迟过来进行复制，不用就不复制了。

### 进程加载

进程加载：用户程序通过exec()，完成新可执行文件的加载，创建一个新进程。

exec主要工作：识别可执行文件格式和内容，然后加载文件内容到PCB，重写寄存器、堆栈、代码段等。

### 进程等待和退出

父进程等待子进程结束：wait()系统调用。

- 子进程通过exit()向父进程返回一个值
- 父进程通过wait()接受并处理返回值（子进程会等待父进程接收并处理后才退出）
- 正常情况：子进程存活时，父进程调用wait()，然后父进程进入等待状态，等到子进程调用exit()并返回一个结果时，唤醒父进程，并将exit()返回值作为wait()的返回值返回。
- 有僵尸子进程（子进程早于父进程wait()之前调用了exit()）等待时：子进程等待父进程wait()接收并处理返回值，此时wait()立即返回exit()的值，如果有多个僵尸子进程的话则返回其中一个的值。
- 无子进程存活时：wait()立即返回。

进程的有序终止：exit()系统调用

- 进程结束执行时调用exit()，完成进程资源的回收
- 功能：
  - 返回一个值给父进程
  - 关闭打开的文件等占用资源
  - 释放内存
  - 释放大部分进程相关的内核数据结构
  - 检查父进程是否存活，存活：保留结果，进入僵尸(zombie/defunct)状态，不存活：释放所有数据结构，进程结束
  - 清理所有等待的僵尸进程
- 进程终止是最终的垃圾收集（资源回收）,回收全部资源。

### 其他进程控制系统调用(Unix)

优先级控制：nice()指定进程的初始优先级，Unix系统中进程优先级会随执行时间而衰减。

进程调试支持：ptrace()运行一个进程控制另一个进程执行，设置断点和查看寄存器等。

定时：sleep()可以让进程在定时器的等待队列中等待。

## 实验部分

### 进程管理数据结构

#### PCB\TCB控制块：struct proc_struct

``` c
struct proc_struct {
    enum proc_state state;                      // Process state
    int pid;                                    // Process ID
    int runs;                                   // the running times of Proces
    uintptr_t kstack;                           // Process kernel stack
    volatile bool need_resched;                 // bool value: need to be rescheduled to release CPU?
    struct proc_struct *parent;                 // the parent process
    struct mm_struct *mm;                       // Process's memory management field
    struct context context;                     // Switch here to run process
    struct trapframe *tf;                       // Trap frame for current interrupt
    uintptr_t cr3;                              // CR3 register: the base addr of Page Directroy Table(PDT)
    uint32_t flags;                             // Process flag
    char name[PROC_NAME_LEN + 1];               // Process name
    list_entry_t list_link;                     // Process link list 
    list_entry_t hash_link;                     // Process hash list
};
```

ucoreOS中TCB和PCB用的都是这一个数据结构，lab4内核创建的都是线程，内核线程。

- tf：中断帧的指针，总是指向内核栈的某个位置：当进程从用户空间跳到内核空间时，中断帧记录了进程在被中断前的状态。当内核需要跳回用户空间时，需要调整中断帧以恢复让进程继续执行的各寄存器值。除此之外，uCore内核允许嵌套中断。因此为了保证嵌套中断发生时tf 总是能够指向当前的trapframe，uCore 在内核栈上维护了 tf 的链，可以参考trap.c::trap函数做进一步的了解。
- cr3: cr3 保存页表的物理地址，目的就是进程切换的时候方便直接使用 lcr3实现页表切换，避免每次都根据 mm 来计算 cr3。mm数据结构是用来实现用户空间的虚存管理的，但是内核线程没有用户空间，它执行的只是内核中的一小段代码（通常是一小段函数），所以它没有mm 结构，也就是NULL。当某个进程是一个普通用户态进程的时候，PCB 中的 cr3 就是 mm 中页表（pgdir）的物理地址；而当它是内核线程的时候，cr3 等于boot_cr3。而boot_cr3指向了uCore启动时建立好的饿内核虚拟空间的页目录表首地址。
- kstack: 每个线程都有一个内核栈，并且位于内核地址空间的不同位置。对于内核线程，该栈就是运行时的程序使用的栈；而对于普通进程，该栈是发生特权级改变的时候使保存被打断的硬件信息用的栈。uCore在创建进程时分配了 2 个连续的物理页（参见memlayout.h中KSTACKSIZE的定义）作为内核栈的空间。这个栈很小，所以内核中的代码应该尽可能的紧凑，并且避免在栈上分配大的数据结构，以免栈溢出，导致系统崩溃。kstack记录了分配给该进程/线程的内核栈的位置。主要作用有以下几点。首先，当内核准备从一个进程切换到另一个的时候，需要根据kstack 的值正确的设置好 tss （可以回顾一下在实验一中讲述的 tss 在中断处理过程中的作用），以便在进程切换以后再发生中断时能够使用正确的栈。其次，内核栈位于内核地址空间，并且是不共享的（每个线程都拥有自己的内核栈），因此不受到 mm 的管理，当进程退出的时候，内核能够根据 kstack 的值快速定位栈的位置并进行回收。uCore 的这种内核栈的设计借鉴的是 linux 的方法（但由于内存管理实现的差异，它实现的远不如 linux 的灵活），它使得每个线程的内核栈在不同的位置，这样从某种程度上方便调试，但同时也使得内核对栈溢出变得十分不敏感，因为一旦发生溢出，它极可能污染内核中其它的数据使得内核崩溃。如果能够通过页表，将所有进程的内核栈映射到固定的地址上去，能够避免这种问题，但又会使得进程切换过程中对栈的修改变得相当繁琐。感兴趣的同学可以参考 linux kernel 的代码对此进行尝试。

#### 保存上下文信息：struct context

```c
// Saved registers for kernel context switches.
// Don't need to save all the %fs etc. segment registers,
// because they are constant across kernel contexts.
// Save all the regular registers so we don't need to care
// which are caller save, but not the return register %eax.
// (Not saving %eax just simplifies the switching code.)
// The layout of context must match code in switch.S.
struct context {
    uint32_t eip;
    uint32_t esp;
    uint32_t ebx;
    uint32_t ecx;
    uint32_t edx;
    uint32_t esi;
    uint32_t edi;
    uint32_t ebp;
};
```

#### 与context对应的switch.S中switch_to函数

switch_to函数实现了上下文切换、利用堆栈保护、恢复进程上下文的功能。

``` asm
.text
.globl switch_to
switch_to:                      # switch_to(from, to)

    # save from's registers
    movl 4(%esp), %eax          # eax points to from
    popl 0(%eax)                # save eip !popl
    movl %esp, 4(%eax)
    movl %ebx, 8(%eax)
    movl %ecx, 12(%eax)
    movl %edx, 16(%eax)
    movl %esi, 20(%eax)
    movl %edi, 24(%eax)
    movl %ebp, 28(%eax)

    # restore to's registers
    movl 4(%esp), %eax          # not 8(%esp): popped return address already
                                # eax now points to to
    movl 28(%eax), %ebp
    movl 24(%eax), %edi
    movl 20(%eax), %esi
    movl 16(%eax), %edx
    movl 12(%eax), %ecx
    movl 8(%eax), %ebx
    movl 4(%eax), %esp

    pushl 0(%eax)               # push eip

    ret
```

#### 保存所有进程地址的哈希链表：hash_list

``` c
// hash list for process set based on pid
static list_entry_t hash_list[HASH_LIST_SIZE]
```

#### [任务状态段TSS](https://blog.csdn.net/longintchar/article/details/51485179)

参见lab1特权级转换一节。

在一个多任务环境中，当任务发生切换时，必须保存现场（比如通用寄存器，段寄存器，栈指针等）。为了保存被切换任务的状态，并且在下次执行它时恢复现场，每个任务都应当有一片内存区域，专门用于保存现场信息，这就是任务状态段（Task State Segment）。

在创建一个任务的时候，我们要为这个任务创建TSS并填写其中的一些字段。

1. 前一任务链接（TSS Back Link）：TSS内偏移0处是前一个任务的TSS描述符的选择子。 
   当Call指令、中断或者异常造成任务切换，处理器会把旧任务的TSS选择子复制到新任务的TSS的Back Link字段中，并且设置新任务的NT（EFLAGS的bit14）为1，以表明新任务嵌套于旧任务中。关于这点我们会在第15章学习。在创建一个任务的时候，这个字段可以填写0.
2. SS0，SS1，SS2和ESP0，ESP1，ESP2分别是0,1,2特权级堆栈的选择子和栈顶指针。这些内容应当由任务的创建者填写，且属于填写后一般不变的静态字段。
3. CR3和分页有关
4. 偏移为0x20~0x5C的区域是处理器各个寄存器的快照部分，用于任务切换时保存现场。在一个多任务环境中，每次创建一个任务，内核至少要填写EIP,EFLAGS,ESP,CS,DS,SS,ES,FS和GS。当任务首次执行时，处理器从这些寄存器中加载初始执行环境，从CS：EIP处开始执行任务的第一条指令。
5. LDT选择子是当前任务的LDT选择子，由内核填写，以指向当前任务的LDT。该信息由处理器在任务切换时使用，在任务运行期间保持不变
6. T（Debug Trap）位用于软件调试。在多任务环境中，如果T=1，则每次切换到该任务的时候，会引发一个调试异常中断（int 1）
7. I/O位图基地址用来决定当前的任务是否可以访问特定的硬件端口。

TSS的使用是为了解决调用门中特权级变换时堆栈发生的变化，每一个任务是最多可能在四个特权级间转移，所以每个任务实际上需要4个堆栈，从上图中可以看出，从偏移4到偏移27的3 个SS和3个ESP，当发生堆栈切换时，被调用者（内层，高特权级）的堆栈的SS和ESP就是从这里取得的。因为是从低特权级向高特权级转换，故TSS中没有最外层（最低特权级）的堆栈信息.

### ucoreOS系统进程的执行流程

#### 初始化时的执行流程

``` mermaid
graph LR
kern_init--> A[proc_init]
A --> X[init idleproc]
A --> |创建内核线程initproc作为实例| B[kern_thread]
A --> |通过pid查hash表hash32找到进程指针|find_proc
B --> |设置trapframe为内核进程的|C[do_fork]

```



- kern_init()：内核初始化，内存、进程、系统等各方面的初始化。
- proc_init()：初始化了一个空闲线程idleproc和一个演示用的内核线程initproc，并通过hash32()函数由pid查到initproc的控制块指针。
- idleproc：空闲进程，idleproc内核线程的工作就是不停地查询（schedule()），看是否有其他内核线程可以执行了，如果有，马上让调度器选择那个内核线程执行（请参考cpu_idle函数的实现）。所以idleproc内核线程是在ucore操作系统没有其他内核线程可执行的情况下才会被调用。
- kern_thread():设置trapframe为内核进程的，内核进程共享资源，所以设置为内核的CS、DS等即可，而不受copy当前进程的tf
- do_fork()：建立线程，复制当前进程的PCB信息，建立新线程。

#### 创建进程：do_fork()执行流程

``` mermaid
graph LR
A[do_fork] --> 判断进程数是否超过最大进程数MAX_PROCESS
A --> alloc_proc
A --> setup_kstack
A --> copy_mm
A --> copy_thread
A --> local_intr_save屏蔽中断
A --> hash_proc
A --> 设置进程为就绪状态
A --> local_intr_restore打开中断
```

- alloc_proc():分配并初始化PCB，各成员置零或设为默认值
- setup_kstack():申请KSTACKPAGE大小的页面，设置为内核栈，ucoreOSv中分配2页
- copy_mm():复制mm,复制用户进程的内存空间和设置相关内存管理（如页表等）信息，如果是内核进程则不申请，而是共用。COW机制中是在复制时设置写保护，当写入时发生缺页中断，在中断中复制修改的页，减少开销。
- copy_thread()设置进程在内核（将来也包括用户态）正常运行和调度所需的中断帧和执行上下文
- hash_proc():把设置好的进程控制块根据pid映射hash值，加入hash_list和proc_list两个全局进程链表中

#### 调度进程：schedule()

调度器会在特定的调度点上执行调度，完成进程切换。在lab4中，这个调度点就一处，即在cpu_idle函数中，此函数如果发现当前进程（也就是idleproc）的need_resched置为1（在初始化idleproc的进程控制块时就置为1了），则调用schedule函数，完成进程调度和进程切换。

进程调度的过程其实比较简单，就是在进程控制块链表中查找到一个“合适”的内核线程，所谓“合适”就是指内核线程处于“PROC_RUNNABLE”状态。在接下来的switch_to函数(在后续有详细分析，有一定难度，需深入了解一下)完成具体的进程切换过程。

#### 执行进程：proc_run()执行流程

```c
// proc_run - make process "proc" running on cpu
// NOTE: before call switch_to, should load  base addr of "proc"'s new PDT
void
proc_run(struct proc_struct *proc) {
    if (proc != current) {
        bool intr_flag;
        struct proc_struct *prev = current, *next = proc;
        local_intr_save(intr_flag);
        {
            current = proc;
            load_esp0(next->kstack + KSTACKSIZE);
            lcr3(next->cr3);
            switch_to(&(prev->context), &(next->context));
        }
        local_intr_restore(intr_flag);
    }
}
```

函数的执行过程是：

1. 让 current 指向 next 内核线程 initproc；
2. 设置任务状态段 ts 中特权态 0 下的栈顶指针 esp0 为 next 内核线程 initproc 的内核栈的栈顶，即 next->kstack + KSTACKSIZE ；
3. 设置 CR3 寄存器的值为 next 内核线程 initproc 的页目录表起始地址 next->cr3，这实际上是完成进程间的页表切换；
4. 由 switch_to函数完成具体的两个线程的执行现场切换，即切换各个寄存器，当 switch_to 函数执行完“ret”指令后，就切换到 initproc 执行了。

### 用户进程管理

#### 加载ELF格式二进制代码并创建用户进程

一般操作系统做法：ELF文件保存在文件系统中，执行时，将ELF加载到内存，然后创建用户进程。

ELF文件头格式：

``` c
#define ELF_MAGIC   0x464C457FU         // "\x7FELF" in little endian

/* file header */
struct elfhdr {
    uint32_t e_magic;     // must equal ELF_MAGIC
    uint8_t e_elf[12];
    uint16_t e_type;      // 1=relocatable, 2=executable, 3=shared object, 4=core image
    uint16_t e_machine;   // 3=x86, 4=68K, etc.
    uint32_t e_version;   // file version, always 1
    uint32_t e_entry;     // entry point if executable
    uint32_t e_phoff;     // file position of program header or 0
    uint32_t e_shoff;     // file position of section header or 0
    uint32_t e_flags;     // architecture-specific flags, usually 0
    uint16_t e_ehsize;    // size of this elf header
    uint16_t e_phentsize; // size of an entry in program header
    uint16_t e_phnum;     // number of entries in program header or 0
    uint16_t e_shentsize; // size of an entry in section header
    uint16_t e_shnum;     // number of entries in section header or 0
    uint16_t e_shstrndx;  // section number that contains section name strings
};

/* program section header */
struct proghdr {
    uint32_t p_type;   // loadable code or data, dynamic linking info,etc.
    uint32_t p_offset; // file offset of segment
    uint32_t p_va;     // virtual address to map segment
    uint32_t p_pa;     // physical address, not used
    uint32_t p_filesz; // size of segment in file
    uint32_t p_memsz;  // size of segment in memory (bigger if contains bss）
    uint32_t p_flags;  // read/write/execute bits
    uint32_t p_align;  // required alignment, invariably hardware page size
};
```

ELF访问实例：

``` c
 	//(3.1) get the file header of the bianry program (ELF format)
    struct elfhdr *elf = (struct elfhdr *)binary;
    //(3.2) get the entry of the program section headers of the bianry program (ELF format)
    struct proghdr *ph = (struct proghdr *)(binary + elf->e_phoff);
    //(3.3) This program is valid?
    if (elf->e_magic != ELF_MAGIC) {
        ret = -E_INVAL_ELF;
        goto bad_elf_cleanup_pgdir;
    }
```



lab5中ucoreOS做法：操作系统和ELF文件加载到内存中（ELF文件跟在操作系统后面，或者通过烧录器指定一个固定地址），操作系统运行后直接创建用户进程。这也是无文件系统的操作系统的统用做法。lab8中ucoreOS将包含文件系统，可以用一般操作系统的做法实现。

创建用户进程的方式：内核复制一个内核线程作为用户进程的壳，int 0x80软中断切换特权级到用户态，将内存中的ELF程序加到这个用户进程的壳中。这个过程使用函数do_execve()实现。

#### 进程控制函数

- do_exit：释放进程自身所占内存空间和相关内存管理（如页表等）信息所占空间，唤醒父进程，好让父进程收了自己，让调度器切换到其他进程
- do_execve：先回收自身所占用的内存空间，然后调用load_icode，用新的程序覆盖内存空间，形成一个执行新程序的进程。入参四个，1. 程序名，2. 程序名长度， 3. 二进制文件首地址 4. 二进制文件长度。后两个参数传给load_icode。
- load_icode：被do_execve调用，完成加载放在内存中的执行程序到进程空间，这涉及到对页表等的修改，分配用户栈。即将内存中其他位置的代码段数据段，搬到系统给该进程分配的用户空间中。两个入参：第一个入参是ELF代码所在的首地址，第二个入参是代码长度。
- do_yield：让调度器执行一次选择新进程的过程，即重新调度进程，current->need_resched = 1。
- do_wait：父进程等待子进程，并在得到子进程的退出消息后，彻底回收子进程所占的资源（比如子进程的内核栈和进程控制块）
- do_kill：给一个进程设置PF_EXITING标志（“kill”信息，即要它死掉），这样在trap函数中，将根据此标志，让进程退出

#### 用户程序的编译

ld命令，把hello应用程序的执行码obj/\_\_user_hello.out连接在了ucore kernel的末尾。且ld命令会在kernel中会把\_\_user_hello.out的位置和大小记录在全局变量\_binary_obj\_\__user_hello_out_start和\_binary_obj___user_hello_out_size中，这样这个hello用户程序就能够和ucore内核一起被 bootloader 加载到内存里中，并且通过这两个全局变量定位hello用户程序执行码的起始位置和大小。而到了与文件系统相关的实验后，ucore会提供一个简单的文件系统，那时所有的用户程序就都不再用这种方法进行加载了，而可以用大家熟悉的文件方式进行加载了。

在tools/user.ld描述了用户程序的用户虚拟空间的执行入口虚拟地址：

```
SECTIONS {
    /* Load programs at this address: "." means the current address */
    . = 0x800020;
```

在tools/kernel.ld描述了操作系统的内核虚拟空间的起始入口虚拟地址：

```
SECTIONS {
    /* Load the kernel at this address: "." means the current address */
    . = 0xC0100000;
```

.ld文件时自己设置的，ld工具根据ld文件进行链接。详见lab2笔记ld工具一节

#### 用户态程序执行流程

``` mermaid
graph LR
kern_init --> A[proc_init]
A --> idleproc
A --> B[initproc:init_main]
B --> C[kernel_thread]
C --> user_main
```

ucoreOS lab5系统初始化时通过proc_init创建了两个内核线程idleproc和initproc，initproc的入口函数为init_main()在这个函数中执行了kernel_thread(user_main, NULL, 0);

即第二个内核线程initproc通过把hello应用程序执行码覆盖到initproc的用户虚拟内存空间来创建了第一各用户进程user_main()

user_main()中通过宏定义找到了hello程序的起始位置和代码并加载。

``` c
#define KERNEL_EXECVE(x) ({                                             \
            extern unsigned char _binary_obj___user_##x##_out_start[],  \
                _binary_obj___user_##x##_out_size[];                    \
            __KERNEL_EXECVE(#x, _binary_obj___user_##x##_out_start,     \
                            _binary_obj___user_##x##_out_size);         \
        })

#define __KERNEL_EXECVE2(x, xstart, xsize) ({                           \
            extern unsigned char xstart[], xsize[];                     \
            __KERNEL_EXECVE(#x, xstart, (size_t)xsize);                 \
        })

#define KERNEL_EXECVE2(x, xstart, xsize)        __KERNEL_EXECVE2(x, xstart, xsize)

// user_main - kernel thread used to exec a user program
static int
user_main(void *arg) {
#ifdef TEST
    KERNEL_EXECVE2(TEST, TESTSTART, TESTSIZE);
#else
    KERNEL_EXECVE(exit);	// 实际执行了这个
#endif
    panic("user_main execve failed.\n");
}
```

KERNEL_EXECVE(hello);展开：（怀疑exit是写错了）

``` c
({ extern unsigned char binaryobj___user_hello_out_start[], binaryobj___user_hello_out_size[]; ({ cprintf("kernel_execve: pid = %d, name = "%s".\n", current->pid, "hello"); kernel_execve("hello", binaryobj___user_hello_out_start, (size_t)(binaryobj___user_hello_out_size)); }); })
```

这个宏最终是调用kernel_execve函数来调用SYS_exec系统调用，由于ld在链接hello应用程序执行码时定义了两全局变量：

- _binary_obj___user_hello_out_start：hello执行码的起始位置
- _binary_obj___user_hello_out_size中：hello执行码的大小

kernel_execve把这两个变量作为SYS_exec系统调用的参数，让ucore来创建此用户进程。当ucore收到此系统调用后，将依次调用如下函数

```
vector128(vectors.S)--\>
\_\_alltraps(trapentry.S)--\>trap(trap.c)--\>trap\_dispatch(trap.c)--
--\>syscall(syscall.c)--\>sys\_exec（syscall.c）--\>do\_execve(proc.c)
```

更多详情以及do_execve,load_icode主要工作详见[gitbook](https://objectkuan.gitbooks.io/ucore-docs/content/lab5/lab5_3_2_create_user_process.html)

所有用户态程序都是从initcode.S开始执行的，所有应用程序的其实用户态执行地址都是`_start`，该程序调整了EBP和ESP，然后调用umain()函数。

``` asm
.text
.globl _start
_start:
    # set ebp for backtrace
    movl $0x0, %ebp

    # move down the esp register
    # since it may cause page fault in backtrace
    subl $0x20, %esp

    # call user-program function
    call umain
1:  jmp 1b
```

umain()函数调用了main函数，并在main函数执行完之后执行exit回收资源。

``` c
// 通过int main()声明了其他文件中的main函数
```

#### 用户进程退出

1. 用户程序执行完后，执行系统所调用exit(error_code)，没错误则exit(0)

2. exit()切到内核态执行系统调用do_exit()

3. do_exit()回收大部分内存资源：

   1. 回收内存：mm计数减一，减一后如果等于0，则说明该控件没有复用，执行回收操作
      1. exit_mmap()回收mm->vma，释放页表占用的空间，并清空页目录项、页表项
      2. put_pgdir()释放页目录所占内存
      3. mm_destory()释放vma结构所占内存和mm结构所占内存
   2. 设置进程状态为僵尸状态(PROC_ZOMBIE)
   3. 唤醒父进程
   4. 如果该进程还有子进程，则将子进程的父进程指针改为initproc并将子进程指针插入initproc的子进程链表，即将当前进程的子进程挂靠在initproc下。如果子进程中有僵尸进程，唤醒initproc回收之。
   5. schedule()重新调度、选新进程执行

   > 到此为止用户进程已进入僵尸状态，不可调度，并已回收大部分资源，剩下资源等待父进程回收
   >
   > 父进程回收的前提条件：父进程创建子进程后执行wait()或者wait_pid(pid)函数，等待子进程结束通知，wait()等待任意子进程结束，wait_pid(pid)等待指定pid的子进程结束通知。

4. 父进程被唤醒后do_wait()中执行的操作：

   1. 执行wait_pid()则查看退出状态的子进程pid是否时指定的，不是则继续等，是则执行下一步
   2. 执行wait()或执行wait_pid()后pid检查正确：查看子进程状态，不是PROC_ZOMBIE则设置自身状态为PROC_SLEEPING，睡眠原因为WT_CHILD（即等待子进程退出），调用sechdule()重新调度，并跳回4.1重新执行，子进程是PROC_ZOMBIE则执行4.3
   3. 子进程状态时PROC_ZOMBIE则将子进程PCB从hash_list和proc_list删除，并释放子进程的内核堆栈和PCB。

5. 至此子进程彻底结束

#### kill进程

kill()–->SYS_kill-->do_kill()-->proc->flags |= PF_EXITING, -->wakeup_proc()-->do_wait()-->do_exit()

