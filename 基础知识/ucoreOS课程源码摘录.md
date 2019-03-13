# ucoreOS课程源码摘录
[TOC]

## 断言与致命错误后监视模式
### assert宏和panic函数
`assert(x)`宏定义，该宏定义入参时一个条件判断，如果判断失败则进入`panic()`函数，该函数在出现不可修复问题时调用，并在内部打印错误信息（使用va_list结构体，打印参数变长的打印消息）和堆栈信息（用到内联汇编，根据调用栈的ebp和eip指针，往回找并打印调用栈信息）、禁用中断后进入`kmonitor()`函数；

- 其中%s对应的#x将x作为字符串打印；
```c
#define assert(x)                                       \
    do {                                                \
        if (!(x)) {                                     \
            panic("assertion failed: %s", #x);          \
        }                                               \
    } while (0)
```

- panic函数实现：

``` c
void
__panic(const char *file, int line, const char *fmt, ...) {
    if (is_panic) {
        goto panic_dead;
    }
    is_panic = 1;

    // print the 'message'
    va_list ap;
    va_start(ap, fmt);
    cprintf("kernel panic at %s:%d:\n    ", file, line);
    vcprintf(fmt, ap);
    cprintf("\n");
    
    cprintf("stack trackback:\n");
    print_stackframe();
    
    va_end(ap);

panic_dead:
    intr_disable();
    while (1) {
        kmonitor(NULL);
    }
}
```

### kmonitor.c设计

`kmonitor.c`文件主要设计了一个内核监视器，当内核出现不可修复的问题时从`panic()`函数中自动进入该函数并死循环。
- 其中循环执行从用户接收指令并执执行。
```cpp
void
kmonitor(struct trapframe *tf) {
    cprintf("Welcome to the kernel debug monitor!!\n");
    cprintf("Type 'help' for a list of commands.\n");

    if (tf != NULL) {
        print_trapframe(tf);
    }

    char *buf;
    while (1) {
        if ((buf = readline("K> ")) != NULL) {
            if (runcmd(buf, tf) < 0) {
                break;
            }
        }
    }
}
```

### readline接收命令
`readline(const char *prompt)`函数，入参为接收输入前打印的消息，之后`\b`退格、 回车结束；
```cpp
char *
readline(const char *prompt) {
    if (prompt != NULL) {
        cprintf("%s", prompt);
    }
    int i = 0, c;
    while (1) {
        c = getchar();
        if (c < 0) {
            return NULL;
        }
        else if (c >= ' ' && i < BUFSIZE - 1) {
            cputchar(c);
            buf[i ++] = c;
        }
        else if (c == '\b' && i > 0) {
            cputchar(c);
            i --;
        }
        else if (c == '\n' || c == '\r') {
            cputchar(c);
            buf[i] = '\0';
            return buf;
        }
    }
}
```

### runcmd执行命令
该函数把命令与一个命令列表结构体`struct command commands[]`比对，并执行相应命令函数，结构体中成员函数指针指向的三个函数参数斗鱼结构体中的设置一样，但其实三个函数都没用到这些参数，后两个还是直接调用了其他函数然后`return 0`。
```cpp
struct command {
    const char *name;
    const char *desc;
    // return -1 to force monitor to exit
    int(*func)(int argc, char **argv, struct trapframe *tf);
};

static struct command commands[] = {
    {"help", "Display this list of commands.", mon_help},
    {"kerninfo", "Display information about the kernel.", mon_kerninfo},
    {"backtrace", "Print backtrace of stack frame.", mon_backtrace},
};

#define NCOMMANDS (sizeof(commands)/sizeof(struct command))
#define MAXARGS         16

static int
runcmd(char *buf, struct trapframe *tf) {
    char *argv[MAXARGS];
    int argc = parse(buf, argv);
    if (argc == 0) {
        return 0;
    }
    int i;
    for (i = 0; i < NCOMMANDS; i ++) {
        if (strcmp(commands[i].name, argv[0]) == 0) {
            return commands[i].func(argc - 1, argv + 1, tf);
        }
    }
    cprintf("Unknown command '%s'\n", argv[0]);
    return 0;
}
```

### parse分割收到的字符串为单独命令
把`readline`函数收到的字符串按照`WHITESPACE`分割成一个个单独的命令存在字符数组`argv`中并返回字数数组个数`argc`；
- `strchr()`函数可以同时寻找多个字符，不是只能找一个；
```cpp
/* parse -  parse the command buffer into whitespace-separated arguments */
#define WHITESPACE      " \t\n\r" //这是四个字符，空格、制表、回车、换行

static int
parse(char *buf, char **argv) {
    int argc = 0;
    while (1) {
        // find global whitespace
        while (*buf != '\0' && strchr(WHITESPACE, *buf) != NULL) {
            *buf ++ = '\0';
        }
        if (*buf == '\0') {
            break;
        }

        // save and scan past next arg
        if (argc == MAXARGS - 1) {
            cprintf("Too many arguments (max %d).\n", MAXARGS);
        }
        argv[argc ++] = buf;
        while (*buf != '\0' && strchr(WHITESPACE, *buf) == NULL) {
            buf ++;
        }
    }
    return argc;
}
```

## 一个链表链接不同数据结构

链表定义：

```c
struct list_entry {
    struct list_entry *prev, *next;
};
typedef struct list_entry list_entry_t;
```

使用该链表的结构体举例：

``` c
// vma 结构体
struct vma_struct {
    struct mm_struct *vm_mm;
    uintptr_t vm_start;
    uintptr_t vm_end;
    uint32_t vm_flags;
    list_entry_t list_link; 
};
// Page 结构体
struct Page {
    int ref;
    uint32_t flags;
    unsigned int property;
    list_entry_t page_link;
    list_entry_t pra_page_link;
    uintptr_t pra_vaddr;
};
```

地址转换宏：

``` c#
// list_entry_t to struct Page
#define le2page(le, member)               	\
    to_struct((le), struct Page, member)
// list_entry_t to struct vma_struct
#define le2vma(le, member)               	\
    to_struct((le), struct vma_struct, member)
// list_entry_t to member
#define to_struct(ptr, type, member)     	\
    ((type *)((char *)(ptr) - offsetof(type, member)))
// offset from struct type's head to member
#define offsetof(type, member) 				\
    ((size_t)(&((type *)0)->member))
```

使用举例：

``` c
list_entry_t *le = list;
struct vma_struct *vma 	= le2vma(le, list_link);
						= to_struct((le), struct vma_struct, list_link);
                        = ((struct vma_struct *)((char *)(le) - \
                        	offsetof(struct vma_struct, list_link)));
						= (struct vma_struct *)((char *)(le) - 16Byte;
						= (struct vma_struct *)((char *)(le) - 0x40000;
offsetof(struct vma_struct, list_link) = ((size_t)(&((struct vma_struct *)0)->list_link)) = ((size_t)(vma_struct->list_link的地址)) = (size_t)从地址0到lisk_link地址的大小 = 4*32bit（32位系统中，64位系统中则1*32bit+3*64bit） = 16Byte = 0x40000
// 注：(sturct vma_struct *)0表示将0地址作为该结构体的起始地址，所以其成员偏移的大小即为这个结构体头到成员地址的举例，即offset
```

## 管理接口结构体

定义举例：

``` c
// pmm.h文件中，对外的调用接口，里面定义了alloc_page等函数让外界调用，alloc_page等函数中则直接调用struct pmm_manager->alloc_pages
struct pmm_manager {
    const char *name;                                 // XXX_pmm_manager's name
    void (*init)(void);                               // initialize internal description&management data structure
                                                      // (free block list, number of free block) of XXX_pmm_manager 
    void (*init_memmap)(struct Page *base, size_t n); // setup description&management data structcure according to
                                                      // the initial free physical memory space 
    struct Page *(*alloc_pages)(size_t n);            // allocate >=n pages, depend on the allocation algorithm 
    void (*free_pages)(struct Page *base, size_t n);  // free >=n pages with "base" addr of Page descriptor structures(memlayout.h)
    size_t (*nr_free_pages)(void);                    // return the number of free pages 
    void (*check)(void);                              // check the correctness of XXX_pmm_manager 
};
```

实现举例：

``` c
// default_pmm.c中，实现具体的算法，然后放到结构体中
const struct pmm_manager default_pmm_manager = {
    .name = "default_pmm_manager",
    .init = default_init,
    .init_memmap = default_init_memmap,
    .alloc_pages = default_alloc_pages,
    .free_pages = default_free_pages,
    .nr_free_pages = default_nr_free_pages,
    .check = default_check,
};

// default_pmm.h中，只对外显示该结构体
#ifndef __KERN_MM_DEFAULT_PMM_H__
#define  __KERN_MM_DEFAULT_PMM_H__

#include <pmm.h>

extern const struct pmm_manager default_pmm_manager;

#endif /* ! __KERN_MM_DEFAULT_PMM_H__ */

// pmm.c中使用default_pmm的实现算法
const struct pmm_manager *pmm_manager;
static void init_pmm_manager(void) {
    pmm_manager = &default_pmm_manager;
    cprintf("memory management: %s\n", pmm_manager->name);
    pmm_manager->init();
}
```

## linux内核hash实现

hash32实现（这里摘录的是ucoreOS中的实现，linux实现类似，GOLDEN_RATIO_PRIME_32是一样的）：

``` c
/* 2^31 + 2^29 - 2^25 + 2^22 - 2^19 - 2^16 + 1 */
#define GOLDEN_RATIO_PRIME_32       0x9e370001UL

/* *
 * hash32 - generate a hash value in the range [0, 2^@bits - 1]
 * @val:    the input value
 * @bits:   the number of bits in a return value
 *
 * High bits are more random, so we use them.
 * */
uint32_t
hash32(uint32_t val, unsigned int bits) {
    uint32_t hash = val * GOLDEN_RATIO_PRIME_32;
    return (hash >> (32 - bits));
}
```

hash的方式是，让key乘以一个大数，于是结果溢出，就把留在32/64位变量中的值作为hash值，又由于散列表的索引长度有限，我们就取这hash值的高几为作为索引值，之所以取高几位，是因为高位的数更具有随机性，能够减少所谓“冲突”。什么是冲突呢？从上面的算法来看，key和hash值并不是一一对应的。有可能两个key算出来得到同一个hash值，这就称为“冲突”。

那么，乘以的这个大数应该是多少呢？从上面的代码来看，32位系统中这个数是0x9e370001UL，64位系统中这个数是0x9e37fffffffc0001UL。这个数是怎么得到的呢？

“Knuth建议，要得到满意的结果，对于32位机器，2^32做黄金分割，这个大树是最接近黄金分割点的素数，0x9e370001UL就是接近 2^32*(sqrt(5)-1)/2 的一个素数，且这个数可以很方便地通过加运算和位移运算得到，因为它等于2^31 + 2^29 - 2^25 + 2^22 - 2^19 - 2^16 + 1。对于64位系统，这个数是0x9e37fffffffc0001UL，同样有2^63 + 2^61 - 2^57 + 2^54 - 2^51 - 2^18 + 1。”

从程序中可以看到，对于32位系统计算hash值是直接用的乘法，因为gcc在编译时会自动优化算法。而对于64位系统，gcc似乎没有类似的优化，所以用的是位移运算和加运算来计算。首先n=hash, 然后n左移18位，hash-=n，这样hash = hash * (1 - 2^18)，下一项是-2^51，而n之前已经左移过18位了，所以只需要再左移33位，于是有n <<= 33，依次类推，最终算出了hash值。

## ucoreOS系统调用接口实现

内核系统调用实现kernel/syscall.c

``` c
// 一个函数数组
static int (*syscalls[])(uint32_t arg[]) = {
    [SYS_exit]              sys_exit,
    [SYS_fork]              sys_fork,
    [SYS_wait]              sys_wait,
    [SYS_exec]              sys_exec,
    [SYS_yield]             sys_yield,
    [SYS_kill]              sys_kill,
    [SYS_getpid]            sys_getpid,
    [SYS_putc]              sys_putc,
    [SYS_pgdir]             sys_pgdir,
};

#define NUM_SYSCALLS        ((sizeof(syscalls)) / (sizeof(syscalls[0])))

void
syscall(void) {
    struct trapframe *tf = current->tf;
    uint32_t arg[5];
    int num = tf->tf_regs.reg_eax;
    if (num >= 0 && num < NUM_SYSCALLS) {
        if (syscalls[num] != NULL) {
            arg[0] = tf->tf_regs.reg_edx;
            arg[1] = tf->tf_regs.reg_ecx;
            arg[2] = tf->tf_regs.reg_ebx;
            arg[3] = tf->tf_regs.reg_edi;
            arg[4] = tf->tf_regs.reg_esi;
            tf->tf_regs.reg_eax = syscalls[num](arg);
            return ;
        }
    }
    print_trapframe(tf);
    panic("undefined syscall %d, pid = %d, name = %s.\n",
            num, current->pid, current->name);
}
```

用户库系统调用实现user/libs/syscall.c

``` c
#include <syscall.h>		// 这是include的内核头文件

#define MAX_ARGS            5

static inline int
syscall(int num, ...) {
    va_list ap;
    va_start(ap, num);
    uint32_t a[MAX_ARGS];
    int i, ret;
    for (i = 0; i < MAX_ARGS; i ++) {
        a[i] = va_arg(ap, uint32_t);
    }
    va_end(ap);

    asm volatile (
        "int %1;"
        : "=a" (ret)
        : "i" (T_SYSCALL),
          "a" (num),
          "d" (a[0]),
          "c" (a[1]),
          "b" (a[2]),
          "D" (a[3]),
          "S" (a[4])
        : "cc", "memory");
    return ret;
}

int
sys_exit(int error_code) {
    return syscall(SYS_exit, error_code);
}
```

syscall的最终汇编代码如下（更易懂）：

``` asm
……
  34:    8b 55 d4               mov    -0x2c(%ebp),%edx
  37:    8b 4d d8               mov    -0x28(%ebp),%ecx
  3a:    8b 5d dc                mov    -0x24(%ebp),%ebx
  3d:    8b 7d e0                mov    -0x20(%ebp),%edi
  40:    8b 75 e4                mov    -0x1c(%ebp),%esi
  43:    8b 45 08               mov    0x8(%ebp),%eax
  46:    cd 80                   int    $0x80
48: 89 45 f0                mov    %eax,-0x10(%ebp)
……
```

可以看到其实是把系统调用号放到EAX，其他5个参数a[0]~a[4]分别保存到EDX/ECX/EBX/EDI/ESI五个寄存器中，及最多用6个寄存器来传递系统调用的参数，且系统调用的返回结果是EAX。比如对于getpid库函数而言，系统调用号（SYS_getpid=18）是保存在EAX中，返回值（调用此库函数的的当前进程号pid）也在EAX中。

用户和内核的syscall通过中断联系，IDT中设置了用于系统调用的中断门idt[T_SYSCALL]。

``` c
static void
trap_dispatch(struct trapframe *tf) {
    char c;

    int ret=0;

    switch (tf->tf_trapno) {
    case T_PGFLT:  //page fault
        if ((ret = pgfault_handler(tf)) != 0) {
            print_trapframe(tf);
            if (current == NULL) {
                panic("handle pgfault failed. ret=%d\n", ret);
            }
            else {
                if (trap_in_kernel(tf)) {
                    panic("handle pgfault failed in kernel mode. ret=%d\n", ret);
                }
                cprintf("killed by kernel.\n");
                panic("handle user mode pgfault failed. ret=%d\n", ret); 
                do_exit(-E_KILLED);
            }
        }
        break;
    case T_SYSCALL:
        syscall();
        break;
```

与用户态的函数库调用执行过程相比，系统调用执行过程的有四点主要的不同：

- 不是通过“CALL”指令而是通过“INT”指令发起调用；
- 不是通过“RET”指令，而是通过“IRET”指令完成调用返回；
- 当到达内核态后，操作系统需要严格检查系统调用传递的参数，确保不破坏整个系统的安全性；
- 执行系统调用可导致进程等待某事件发生，从而可引起进程切换；

> 总结来说就是，系统调用不是通过CALL调用函数的方式，而是INT触发软中断的方式

系统调用执行实例：

例如getpid()：

```
vector128(vectors.S)-->__alltraps(trapentry.S)-->trap(trap.c)-->
trap_dispatch(trap.c)-->syscall(syscall.c)-->sys_getpid(syscall.c)-->
……-->__trapret(trapentry.S)
```

