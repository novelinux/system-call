fork
========================================

传统的UNIX中用于复制进程的系统调用是fork。但它并不是Linux为此实现的唯一调用，实际上Linux实现了3个:

* fork

是重量级调用，因为它建立了父进程的一个完整副本，然后作为子进程执行。
为减少与该调用相关的工作量，Linux使用了写时复制（copy-on-write）技术.

* vfork

类似于fork，但并不创建父进程数据的副本。相反，父子进程之间共享数据。
这节省了大量CPU时间（如果一个进程操纵共享数据，则另一个会自动注意到）。
vfork设计用于子进程形成后立即执行execve系统调用加载新程序的情形。在子进程
退出或开始新程序之前，内核保证父进程处于堵塞状态。引用手册页vfork(2)的文字，
“非常不幸，Linux从过去复活了这个幽灵”。由于fork使用了写时复制技术，vfork速度
面不再有优势，因此应该避免使用它。

* clone

产生线程，可以对父子进程之间的共享、复制进行精确控制。

**写时复制**:

内核使用了写时复制（Copy-On-Write，COW）技术，以防止在fork执行时将父进程的所有数据复制到子进程。
该技术利用了下述事实：进程通常只使用了其内存页的一小部分。在调用fork时，内核通常对父进程的每个
内存页，都为子进程创建一个相同的副本。这有两种很不好的负面效应。

* 使用了大量内存。
* 复制操作耗费很长时间。如果应用程序在进程复制之后使用exec立即加载新程序，那么负面效应会更严重。
  这实际上意味着，此前进行的复制操作是完全多余的，因为进程地址空间会重新初始化，复制的数据不再
  需要了。

内核可以使用技巧规避该问题。并不复制进程的整个地址空间，而是只复制其页表。这样就建立了虚拟地址空间
和物理内存页之间的联系，因此，fork之后父子进程的地址空间指向同样的物理内存页。当然，父子进程不能
允许修改彼此的页，这也是两个进程的页表对页标记了只读访问的原因，即使在普通环境下允许写入也是如此。
假如两个进程只能读取其内存页，那么二者之间的数据共享就不是问题，因为不会有修改。只要一个进程试图
向复制的内存页写入，处理器会向内核报告访问错误（此类错误被称作缺页异常）。内核然后查看额外的内存
管理数据结构，检查该页是否可以用读写模式访问，还是只能以只读模式访问。如果是后者，则必须向进程报告
段错误。缺页异常处理程序的实际实现要复杂得多，因为还必须考虑其他方面的问题，例如换出的页。如果
页表项将一页标记为“只读”，但通常情况下该页应该是可写的，内核可根据此条件来判断该页实际上是COW页。
因此内核会创建该页专用于当前进程的副本，当然也可以用于写操作。COW机制使得内核可以尽可能延迟内存页
的复制，更重要的是，在很多情况下不需要复制。这节省了大量时间。


下面我们以bionic c库中的fork函数为例来讲解fork函数的执行过程:

fork
----------------------------------------

path: bionic/libc/bionic/fork.c
```
int  fork(void)
{
    int ret;
    ...
    ret = __fork();
    ...
    return ret;
}
```

__fork
----------------------------------------

path: bionic/libc/arch-arm/syscalls/__fork.S
```
ENTRY(__fork)
    mov     ip, r7          # 将r7寄存器的值保存到ip寄存器中.
    ldr     r7, =__NR_fork  # 将__NR_fork宏表示的值加载到r7寄存器中

    swi     #0 # 触发软中断
    mov     r7, ip
    cmn     r0, #(MAX_ERRNO + 1)
    bxls    lr
    neg     r0, r0
    b       __set_errno
END(__fork)
```

path: kernel/arch/arm/include/asm/unistd.h
```
#define __NR_OABI_SYSCALL_BASE  0x900000

#if defined(__thumb__) || defined(__ARM_EABI__)
#define __NR_SYSCALL_BASE 0
#else
#define __NR_SYSCALL_BASE __NR_OABI_SYSCALL_BASE
#endif

/*
 * This file contains the system call numbers.
 */
#define __NR_restart_syscall  (__NR_SYSCALL_BASE+  0)
...
#define __NR_fork             (__NR_SYSCALL_BASE+  2)
```

* 在较新的EABI规范中,是将系统调用号压入寄存器r7中;
  EABI是什么东西呢? ABI: Application Binary Interface, 应用二进制接口.
* 而在老的OABI中则是执行的swi中断号的方式, 也就是说原来的调用方式(Old ABI)是通过跟随在swi
  指令中的调用号来进行的.

sys_fork
----------------------------------------

https://github.com/novelinux/linux-4.x.y/tree/master/kernel/fork.c/sys_fork.md
