# ucoreOS课程lab1学习笔记
[TOC]

# 学到的知识点
- gcc编译参数
- gdb命令参数即调试方法
- qemu使用
- makefile编写方式及function.mk高级用法
- AT&T汇编与内联汇编
- BIOS启动流程
- bootloador工作流程、主要任务
- 保护模式、段机制、函数调用方式与相应的堆栈操作、中断与异常的处理、特权级

# 尚不太懂的点
- 地址空间：逻辑、线性、物理空间关系，能理清但太绕
- ELF结构体中各变量的用途
- pmm_init初始化的具体作用、段机制对程序运行的影响、地址空间问题
- 特权级切换方式

# 程序执行流程：
1. boot/bootasm.S文件中`start`关中断,清段寄存器,使能A20,加载GDT表,进入保护模式
2. kern/init/init.c文件中`kern_init()`依次初始化控制台、打印内核信息、函数调用测试及信息打印（练习5）,页机制初始化,中断控制初始化,idt初始化（练习6）,时钟初始化,使能中断,内核用户态变换测试（挑战）。
3. 死循环等待。

>注：
>- bootasm.S中加载的GDT表是个空表，kernel中pmm.c会重新加载一个包含内核代码、数据、用户代码、数据、中断服务段(TSS)五个段的GDT表。
>- `grade_backtrace()`函数没有用，是'练习5'的题目，只是不断地嵌套调用，然后最里面打印栈中的esp、ebp数据
>- `pmm_init()`初始化现在只是进行段初始化，还没有加入页机制。
>- 除了`grade_backtrace()、pmm_init()、idt_init()、lab1_switch_test()`之外的初始化都是使用inb、outb等函数做的硬件操作。
>- GDT表结构可以看kern/mm/mmu.h中`struct segdesc`，IDT表结构可以看同一文件中`struct gatedesc`，本文件另有许多两个结构体的设置宏定义。

# 中断处理流程
1. 产生中断后CPU跳转到vectors.S文件中`__vectors[]`数组的位置，根据中断号执行相应的汇编指令，将`error_code`（非异常没有）和`trap_no`压栈，然后跳转`trap()`函数；
2. `trap()`函数直接跳转`trap_dispatch()`根据中断号`trap_no`执行相应操作然后退出。

> 中断号`trap_no`压栈后和之前压入的`esp`、`ds`等数据作为跳转到`trap()`函数后的传参，所有压栈参数组成`struct trapframe`结构体。

# 函数堆栈
每次进行函数调用时都会将之前的栈底指针、参数、地址等压入函数堆栈，然后调用新的函数；
> 压栈的参数主要是ax等寄存器中的数据，之后还会压入下一个函数的传入参数；

## 具体流程：
1. 参数入栈；
2. 返回地址入栈；
3. ebp入栈，esp赋给ebp；
> ebp是栈底指针，esp是栈顶指针，eip为下一条指令地址。  
> 函数调用发生后，之前函数的栈底指针被存入堆栈，之前函数的栈顶指针的位置附给ebp作为新的栈底，调用结束返回的时候从ebp的位置读出数据赋给ebp就可以得到老的栈底指针位置，而esp则在读出ebp的位置作为栈顶指针。

## 函数堆栈结构如下表所示：
栈底方向 | 高位地址
---|---
... | 
参数3 |
参数2 |
参数1 |
返回地址 |
上一层ebp | 当前ebp指向这里
局部变量 |  低位地址
null | 当前esp指向这里

# 段机制
保护模式下可以使用段机制管理内存，使用段基址和段界限两个参数把内存分为大小不等的内存块，这些内存块称为段(segment)。
## 段描述符表
1. **段基址**和**段界限**还有一些其他属性（**段属性**）存放在**段描述符**中；
2. **段描述符**存放在段描述符表也叫**全局描述符表**(GDT)中，每个全局描述符大小为8Byte；
3. **全局描述符表**的首地址存放在**全局描述符表寄存器**(GDTR)中；

```
graph LR
全局描述符表寄存器(GDTR) --> |指向起始地址|A[全局描述符表GDT]
A --> |存有最多2^13个|B[全局描述符]
B --> |包含|段基址
B --> |包含|段界限
B --> |包含|段属性
```

## 段机制发展流程
1. 最早使用16bit地址总线，寄存器就是16bit的，不用段机制；
2. x86使用20bit地址总线，16bit不够了，就加了给寄存器，即cs:ip，cs<<4+ip即可寻址；
3. 后来32bit系统出现，cs:ip刚好够用，但是这样的话使用32bit寻址，不能记录其他属性或描述符信息，所以另建一个8Byte段描述符表，两个16bit寄存器不再直接寻址，而是指向段描述符表，得到需要的各种信息再寻址，这就是段机制。

## 寻址方式
1. 根据**逻辑地址**的高16位**段选择子**在全局描述符表中索引需要的**段描述符**；
2. 从段描述符中提取**段基址**与逻辑地址中的段内偏移量相加即得**物理地址**；
>- 只有段机制的**逻辑地址**由48bit组成，高16bit为**段选择子**，后32bit为**段内偏移量**；
>- 段选择子高13位为段描述符索引（所以最多只能索引2^13=8192个段，低三位分别为GDT\LDT选择和特权级选择（0~3），算上两个表（GDT和LDT）可以索引2^14个段）。  
>- 段选择子不能是0，全局描述符表的第一项是空的，不能被CPU调用。
>- 虽说是把内存分成一个个段，但是段界限可以直接设置为满4G（事实上lab1程序中4段界限均是0xffffffff）。

## pmm.c中GDT初始化
pmm.c中定义了全局变量`struct segdesc gdt[]`用来存储全局描述符表，并使用`SEG()`宏定义直接初始化了`SEG_NULL`等6个段描述符，地址分别为0x00,0x08,0x10,0x18,0x20,0x28，即13bit段选择子分别为1,2,3,4,5,6。
``` C++
static struct segdesc gdt[] = {
    SEG_NULL,
    [SEG_KTEXT] = SEG(STA_X | STA_R, 0x0, 0xFFFFFFFF, DPL_KERNEL),
    [SEG_KDATA] = SEG(STA_W, 0x0, 0xFFFFFFFF, DPL_KERNEL),
    [SEG_UTEXT] = SEG(STA_X | STA_R, 0x0, 0xFFFFFFFF, DPL_USER),
    [SEG_UDATA] = SEG(STA_W, 0x0, 0xFFFFFFFF, DPL_USER),
    [SEG_TSS]    = SEG_NULL,
};
``` 
`SEG()`原型为：`#define SEG(type, base, lim, dpl)`，其中type为类型，base是段基址，lim为段界限，dpl为特权级，宏定义内部实现其实就是对这四个参数进行位移、位与合成一个8Byte符合GDT格式的`struct segdesc`数据，`struct segdesc`与段描述符结构相同。
``` C++ 
struct segdesc {
    unsigned sd_lim_15_0 : 16;        // low bits of segment limit
    unsigned sd_base_15_0 : 16;        // low bits of segment base address
    unsigned sd_base_23_16 : 8;        // middle bits of segment base address
    unsigned sd_type : 4;            // segment type (see STS_ constants)
    unsigned sd_s : 1;                // 0 = system, 1 = application
    unsigned sd_dpl : 2;            // descriptor Privilege Level
    unsigned sd_p : 1;                // present
    unsigned sd_lim_19_16 : 4;        // high bits of segment limit
    unsigned sd_avl : 1;            // unused (available for software use)
    unsigned sd_rsv1 : 1;            // reserved
    unsigned sd_db : 1;                // 0 = 16-bit segment, 1 = 32-bit segment
    unsigned sd_g : 1;                // granularity: limit scaled by 4K when set
    unsigned sd_base_31_24 : 8;        // high bits of segment base address
};
```
初始化`gdt[]`数组后使用`lgdt()`函数注册即可，`lgdt()`中使用内联汇编指令直接调用`lgdt`命令，但是`lgdt`命令的入参要求不是直接的gdt数组，所以还要对gdt表封装一层：
``` C++
static struct pseudodesc gdt_pd = {
    sizeof(gdt) - 1, (uint32_t)gdt
};

static void
gdt_init(void) {
    ...
    // reload all segment registers
    lgdt(&gdt_pd);
    ...
}
```
`pmm_init()`函数中直接跳转`gdt_init()`，`gdt_init()`中除了重新加载全局描述表外还对之前定义为NULL的TSS段进行了设置。


## 中断处理及其中段机制的应用
中断处理中用到的段机制主要在初始化idt表是，要给出中断服务程序所在段的段基址和偏移地址。  
IDT表大小是8Byte，其中6Byte存储逻辑地址，2Byte存储门描述符种类、特权级等信息。
IDT表的设置使用宏定义`SETGATE()`来完成：
``` C++
#define SETGATE(gate, istrap, sel, off, dpl)
```
其中`gate`是门描述符地址，后续信息写入该地址下的门描述符中，`istrap`选择门描述符类型(trap、task、interrupt,task没有用到，所以1 trap， 0 interrupt)，`sel`是段选择子，`off`是段内偏移，`dpl`是特权级，`sel`和`off`组成逻辑地址；  
``` C++
    SETGATE(idt[i], 0, GD_KTEXT, __vectors[i], DPL_KERNEL);
    SETGATE(idt[0], 0, GD_KTEXT, __vectors[T_SWITCH_TOK], DPL_USER);
```
使用时`sel`全部都写`GD_KTEXT`，即段选择子选择内核代码段，因为所有中断函数都写在内核代码中，off段内便宜则写`__vectors[]`数组中定义的相应地址，该数组在vectors.S文件中，这个文件通过vectors.c文件执行后生成（注意是执行后生成，不是编译后生成）。特权级则是看中断服务程序类型设置，如T_SWITCH_TOK是转换特权级到内核，这个中断一定在用户级执行。  
使用`SETGATE()`设置好idt[]后即可使用lidt()函数加载idt表。

# 特权级转换
特权级转换仅发生在中断、异常、系统调用期间，当前CPL与要执行代码段的DPL不等时需要进行特权级转换。  
首先需要明确如下几个名词：
- **TSS任务状态段**：每个任务都有且仅有一个TSS段用于记录当前程序运行的状态，其内容在程序中以结构体变量`struct taskstate ts`记录。TSS中保存了当前程序四个特权级ring0~ring4的堆栈选择子`ss0~3`和栈顶指针`esp0~3`，还有当前各寄存器的当前状态数据，padding是reserved保留的空间，由于寄存器都是16bit的而系统是32bit的所以预留出来了。当前使用的TSS段首地址存在TR寄存器中，暂未用到的程序的TSS段存放在GDT表中，当切换任务是自动把对应TSS装载到TR中。TR和CS、SS寄存器一样是一个段寄存器。所以TSS段主要有两个作用：1、变换特权级；2、切换当前寄存器，例如切换任务时将当前所有寄存器存入TSS后载入新的TSS。但是第二项功能只是Intel设计的时期望的，用于任务切换的功能，但是实际上由于一次切换需要消耗200多个时钟周期太慢，所以现代操作系统使用的是其他方式进行线程和进程的切换。
``` c
struct taskstate {
    uint32_t ts_link;        // old ts selector
    uintptr_t ts_esp0;        // stack pointers and segment selectors
    uint16_t ts_ss0;        // after an increase in privilege level
    uint16_t ts_padding1;
    uintptr_t ts_esp1;
    uint16_t ts_ss1;
    uint16_t ts_padding2;
    uintptr_t ts_esp2;
    uint16_t ts_ss2;
    uint16_t ts_padding3;
    uintptr_t ts_cr3;        // page directory base
    uintptr_t ts_eip;        // saved state from last task switch
    uint32_t ts_eflags;
    uint32_t ts_eax;        // more saved state (registers)
    uint32_t ts_ecx;
    uint32_t ts_edx;
    uint32_t ts_ebx;
    uintptr_t ts_esp;
    uintptr_t ts_ebp;
    uint32_t ts_esi;
    uint32_t ts_edi;
    uint16_t ts_es;            // even more saved state (segment selectors)
    uint16_t ts_padding4;
    uint16_t ts_cs;
    uint16_t ts_padding5;
    uint16_t ts_ss;
    uint16_t ts_padding6;
    uint16_t ts_ds;
    uint16_t ts_padding7;
    uint16_t ts_fs;
    uint16_t ts_padding8;
    uint16_t ts_gs;
    uint16_t ts_padding9;
    uint16_t ts_ldt;
    uint16_t ts_padding10;
    uint16_t ts_t;            // trap on task switch
    uint16_t ts_iomb;        // i/o map base address
};
```
- **内核栈和用户栈**：CPU的每个程序的每个特权级都有一个对应的程序堆栈，Linux和学习用的ucoreOS都只用到了ring0和3，所以分别称为内核栈和用户栈，当进行特权级变换时要从TSS段中取出要跳转栈的段选择子和栈顶指针，并将当前栈的段选择子和栈顶指针存入新栈。也即内核栈的ss和esp存在用户栈中，用户栈的ss和esp存在内核栈中。

所以中断中特权级由用户态到内核态的切换方式为：
1. 根据中断向量的DPL和当前的CPL判断是否需要切换内核态；
2. 需要切换内核态则从当前TSS中读取内核栈的ss和esp，并将当前系统栈切换到内核栈，将用户栈的ss和esp存入内核栈，之后将当前现场的寄存器也存入内核态；
3. 开始执行中断服务程序；
4. 执行完毕转回用户态时依次弹出现场寄存器数据和ss，esp，然后切回用户态开始执行。

所以ucoreOS中触发中断后的堆栈操作为：
1. 触发中断后从idt表找到相应的服务函数地址，也即vectors.S文件中相应`.globl vector[n]`的位置，然后执行后面的汇编压入errorcode和trapno，调转alltrap执行；
2. 后面要调用trap函数，再次之前要进行一些准备工作，主要是trap()函数有入参，需要通过调用栈的形式传入，所以这里先向调用栈压入各段寄存器的数据，然后设置数据段寄存器为内核数据段；
3. 调用trap后，调用栈中的数据被强制类型转换成了`struct trapframe`形式，同时因为从栈顶开始强制转换，底部不属于调用栈的触发中断后自动压入的ss、esp等数据也被包含进该结构体；
4. 然后根据trapno中保存的中断号执行相应的中断程序，如果需要切换特权级，则修改程序中本来不能修改的代码段寄存器cs和数据段寄存去ds、es等，还有相应的特权级栈寄存器ss和esp，这些本来是保存的进入中断前的现场数据，这里进行修改后恢复现场时就恢复不到之前的现场了，而是恢复到了新的特权级现场；
5. 退出中断时CPU自动执行iret命令，此时会自动恢复现场，给ss、esp、cs等寄存器赋压入调用栈的各个值，但由于这些值已经被改过了，所以恢复的现场的特权级也发生了变化，即完成了特权级的切换；
>- 按理说触发中断后应该从idt表找到相应的服务函数后进入各自的中断服务程序，但这里由于操作系统很不完善，所以所有中断都进入了alltrap函数，然后进入trap函数通过switch执行不同命令，其实应该进入不同的函数。
>- struct trapframe不是自己随便定义的结构体，而是根据中断发生、函数调用时的Intel规定的压栈操作，定义的结构体，然后通过修改该结构体强行修改本来访问不到的内存地址存的数据，实际改错了很可能导致系统崩溃。
``` c
struct trapframe {
    struct pushregs tf_regs;
    uint16_t tf_gs;
    uint16_t tf_padding0;
    uint16_t tf_fs;
    uint16_t tf_padding1;
    uint16_t tf_es;
    uint16_t tf_padding2;
    uint16_t tf_ds;
    uint16_t tf_padding3;
    uint32_t tf_trapno;
    /* below here defined by x86 hardware */
    uint32_t tf_err;
    uintptr_t tf_eip;
    uint16_t tf_cs;
    uint16_t tf_padding4;
    uint32_t tf_eflags;
    /* below here only when crossing rings, such as from user to kernel */
    uintptr_t tf_esp;
    uint16_t tf_ss;
    uint16_t tf_padding5;
} __attribute__((packed));
```
