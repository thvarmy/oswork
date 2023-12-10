# Lab-5

## 实验内容：

实验5将创建用户进程，让用户进程在用户态执行，且在需要ucore支持时，可通过系统调用来让ucore提供服务。构造出第一个用户进程，并通过系统调用`sys_fork`/`sys_exec`/`sys_exit`/`sys_wait`来支持运行不同的应用程序，完成对用户进程的执行过程的基本管理。

### 练习0：填写已有实验

vmm.c中do_pgfault（）函数

```cpp
//(1）According to the mm AND addr, try
            //to load the content of right disk page
            //into the memory which page managed.
            swap_in(mm, addr, &page);
            //(2) According to the mm,
            //addr AND page, setup the
            //map of phy addr <--->
            //logical addr
            page_insert(mm->pgdir, page, addr, perm);
            //(3) make the page swappable.
            swap_map_swappable(mm, addr, page, 1);
```

alloc_proc()函数

```cpp
//LAB5 YOUR CODE : (update LAB4 steps)
     /*
     * below fields(add in LAB5) in proc_struct need to be initialized  
     *       uint32_t wait_state;                        // waiting state
     *       struct proc_struct *cptr, *yptr, *optr;     // relations between processes
     */
    proc->wait_state = 0; // 将等待状态初始化为0，表示进程的等待状态
    proc->cptr = proc->yptr = proc->optr = NULL; // 将进程之间的关系初始化为NULL
```

proc_run()函数

```cpp
 bool intr_flag;
        local_intr_save(intr_flag);  // 禁用中断
        {
            struct proc_struct *prev = current, *next = proc; // 定义前一个进程和后一个进程
            current = proc;//切换当前进程为要运行的进程。
            lcr3(next->cr3); //切换页表，以便使用新进程的地址空间。
            switch_to(&(prev->context), &(next->context)); //实现上下文切换
        }

        local_intr_restore(intr_flag);  // 启用中断
```

在`alloc_proc`中添加额外的初始化：

```cpp
proc->wait_state = 0;
proc->cptr = NULL;
proc->optr = NULL;
proc->yptr = NULL;
```

在`do_fork`中修改代码如下：

```cpp
// 分配并初始化PCB
    if ((proc = alloc_proc()) == NULL) {
        cprintf("cannot alloc idleproc.\n");
        goto fork_out;
    }
    proc->parent = current;//修改
    assert(current->wait_state == 0);

    // 分配并初始化内核栈
    if (setup_kstack(proc) != 0) {
        cprintf("cannot alloc kernel stack for initproc.\n");
        goto bad_fork_cleanup_proc;
    }

    // 根据clone_flags决定是复制还是共享内存管理系统
    // 注意传参的时候，clone_flags中已经包含了CLONE_VM
    if(copy_mm(clone_flags, proc) != 0) {
        cprintf("cannot duplicate or share mm.\n");
        goto bad_fork_cleanup_kstack;
    }

    // 设置进程中断帧和上下文
    copy_thread(proc, stack, tf);

    // 把设置好的进程加入链表
    /*
    hash_proc(proc);
    list_add(&proc_list, &proc->list_link);
    nr_process++;
    */
    bool intr_flag;
    local_intr_save(intr_flag);
    {
        proc->pid = get_pid();
        hash_proc(proc);
        set_links(proc);
    }
    local_intr_restore(intr_flag);


    // 将新建的进程设为就绪态
    //initproc->state = PROC_RUNNABLE;
    wakeup_proc(proc);
```

### 练习1：加载应用程序并执行（需要编程）

**题目1**： `do_execve `函数调用` load_icode `（位于kern/process/proc.c中）来加载并解析一个处于内存中的ELF执行文件格式的应用程序。你需要补充`load_icode` 的第6步，建立相应的用户内存空间来放置应用程序的代码段、数据段等，且要设置好proc_struct 结构中的成员变量trapframe中的内容，确保在执行此进程后，能够从应用程序设定的起始执行地址开始执行。需设置正确的trapframe内容。

- 为了让用户级别的进程可以从内核返回到用户模式，所以：
  
  - tf->gpr.sp 应该是用户栈的顶部（即 sp 的值）。
  
  - tf->status 应该适用于用户程序（即 sstatus 的值）。
  
  - tf->epc 应该是用户程序的入口点（即 sepc 的值）。

代码如下：

```cpp
tf->gpr.sp=USTACKTOP;
tf->status=sstatus&(~(SSTATUS_SPP|SSTATUS_SPIE));
tf->epc=elf->e_entry;
```

即：

- 将 tf 结构中的通用寄存器（gpr）中的栈指针（sp）设置为 USTACKTOP。这表明将用户栈的顶部地址赋给用户进程的栈指针。

- 将 tf 结构中的状态寄存器（status）设置为给定的 sstatus，但清除了 SPP（Supervisor PreviousPrivilege）和 SPIE（Supervisor Previous Interrupt Enable）标志。这两个标志通常用于处理从内核返回用户模式时的特权级别和中断使能状态。

- 将 tf 结构中的程序计数器（epc）设置为 ELF 文件的入口点地址。这是用户程序的启动地址，将控制权转移到用户程序的执行起点。

**题目2**：请简要描述这个用户态进程被ucore选择占用CPU执行（RUNNING态）到具体执行应用程序第一条指令的整个经过。

通过do_execve函数来完成用户进程的创建工作，这里调用load_icode函数给用户进程建立一个能够让用户进程正常运行的用户环境。上下文切换实际上要借助中断处理的返回来完成。进行系统调用sys_exec，在trap返回的时候调用了`sret`指令。load_icode()`函数在构造新进程的时候，会把`SSTATUS_SPP`设置为0，使得`sret的时候能回到U mode。

### 练习2: 父进程复制自己的内存空间给子进程（需要编码）

**题目1**：创建子进程的函数 do_fork 在执行中将拷贝当前进程（即父进程）的用户内存地址空间中的合法内容到新进程中（子进程），完成内存资源的复制。具体是通过 `copy_range` 函数（位于`kern/mm/pmm.c`中）实现的。

`do_fork`函数调用`copy_mm`函数实现进程mm的复制，后者根据`clone_flags & CLONE_VM`的取值调用了`dup_mmap`函数。

`dup_mmap`函数在两个进程之间复制内存映射关系。具体来说，该函数的两个参数分别表示目标进程和源进程的内存管理结构mm。然后通过循环迭代，每次创建一个新的内存映射区域（vma），然后将其插入到目标进程的mm中，之后调用copy_range函数将源进程的内存映射区域的内容复制到目标进程中。

填写`copy_range`函数后，其代码实现如下：

```cpp
            /* LAB5:EXERCISE2 YOUR CODE
             * replicate content of page to npage, build the map of phy addr of
             * nage with the linear addr start
             *
             * Some Useful MACROs and DEFINEs, you can use them in below
             * implementation.
             * MACROs or Functions:
             *    page2kva(struct Page *page): return the kernel vritual addr of
             * memory which page managed (SEE pmm.h)
             *    page_insert: build the map of phy addr of an Page with the
             * linear addr la
             *    memcpy: typical memory copy function
             *
             * (1) find src_kvaddr: the kernel virtual address of page
             * (2) find dst_kvaddr: the kernel virtual address of npage
             * (3) memory copy from src_kvaddr to dst_kvaddr, size is PGSIZE
             * (4) build the map of phy addr of  nage with the linear addr start
             */
            // (1) 找到 src_kvaddr：page 的内核虚拟地址
            uintptr_t src_kvaddr = page2kva(page);

            // (2) 找到 dst_kvaddr：npage 的内核虚拟地址
            uintptr_t dst_kvaddr = page2kva(npage);

            // (3) 使用 memcpy 函数将 src_kvaddr 处的内容复制到 dst_kvaddr，大小为 PGSIZE
            memcpy((void *)dst_kvaddr, (void *)src_kvaddr, PGSIZE);

            // (4) 使用 page_insert 函数建立 nage 的物理地址映射，映射到线性地址 start
            ret = page_insert(to, npage, start, perm);
```

函数首先通过断言确保start和end是页面对齐且属于用户地址空间的。然后通过循环每次处理一页数据。在每次迭代中，函数调用` get_pte `函数查找源进程 A 的页表项（`pte`），如果不存在，则跳过当前页并继续下一个页面。如果 `pte `存在，则再调用 `get_pte` 函数查找目标进程 B 的页表项（`nptep`）。如果` nptep` 不存在，则分配一个新的页表并建立映射。

接下来，函数调用 `alloc_page` 函数为进程 B 分配一个新的物理页面（`npage`）。然后，函数将源进程 A的物理页面（`page`）中的内容复制到新的物理页面中。具体实现上，就是通过`page2kva`函数获取到目标进程和源进程各自页面的虚拟地址，然后使用`memcpy`函数实现复制。页面复制完成后，函数再调用`page_insert `函数将新的物理页面与目标进程 B 的线性地址建立映射。

**题目2**：如何设计实现 `Copy on Write `机制？给出概要设计，鼓励给出详细设计。

Copy on Write（COW）机制是一种在多进程共享资源时，延迟复制的技术。它允许多个进程共享同一块内存空间，直到其中一个进程尝试修改数据时，才会进行实际的复制操作，以确保数据的独立性。

在设计该机制时，考虑到了以下两点：

- 未修改时，如何实现内存共享？

- 修改时，如何实现资源复制以及之后的数据一致性？

针对以上两点，进行了如下设计：

- 内存管理：
  
  - 使用页表来跟踪进程间共享的内存页，记录每个页的引用计数。
  
  - 当进程创建时，内核分配一块共享内存区域，并将该区域映射到多个进程的地址空间中。

- 写时复制实现：
  
  - 当进程试图写入一个共享页时，内核会触发页故障：
    
    - 检查被访问的页是否是共享的。
    
    - 如果引用计数为1，表示该页只有一个进程在使用，直接允许修改。
    
    - 如果引用计数大于1，执行复制操作：
      
      - 分配一个新的页，并将原始页的内容复制到新页中。
      
      - 更新进程的页表，使其指向新的页。
      
      - 更新引用计数和页表条目，确保进程间的独立性。

- 引用计数：
  
  - 引用计数用于跟踪共享页的引用情况。每当一个新的进程开始使用某个共享页时，引用计数增加；当进程停止使用该页时，引用计数减少。
  
  - 当引用计数为0时，释放对应的页。

- 数据一致性：
  
  - 确保在进行写时复制时，修改后的页仅影响到修改它的进程，而不影响其他进程。
  
  - 维护页表和引用计数，以确保每个进程都能正确访问到自己的数据，并且在修改时能够获得独立的拷贝。

- 同步机制：
  
  - 在进行写时复制时，需要采取同步机制以避免多个进程同时修改相同的共享页。可以使用锁或其他同步手段来确保数据的一致性和正确性。

### 练习3：阅读分析源代码，理解进程执行 fork/exec/wait/exit 的实现，以及系统调用的实现

简要对fork/exec/wait/exit函数进行分析：

#### 1.fork：执行完毕后，如果创建新进程成功，则出现两个进程，一个是子进程，一个是父进程。在子进程中，fork函数返回0，在父进程中，fork返回新创建子进程的进程ID。我们可以通过fork返回的值来判断当前进程是子进程还是父进程

```assembly
// 调用过程：fork->SYS_fork->do_fork + wakeup_proc

// wakeup_proc 函数主要是将进程的状态设置为等待。

// do_fork()

1、分配并初始化进程控制块(alloc_proc 函数);

2、分配并初始化内核栈(setup_stack 函数);

3、根据 clone_flag标志复制或共享进程内存管理结构(copy_mm 函数);

4、设置进程在内核(将来也包括用户态)正常运行和调度所需的中断帧和执行上下文(copy_thread 函数);

5、把设置好的进程控制块放入hash_list 和 proc_list 两个全局进程链表中;

6、自此,进程已经准备好执行了,把进程状态设置为“就绪”态;

7、设置返回码为子进程的 id 号
```

- 用户态程序调用sys_fork()系统调用，通过syscall进入内核态。

- 内核态处理sys_fork()系统调用，调用do_fork()函数创建子进程，完成后返回到用户态。

#### 2.exit：会把一个退出码error_code传递给ucore，ucore通过执行内核函数do_exit来完成对当前进程的退出处理，主要工作简单地说就是回收当前进程所占的大部分内存资源，并通知父进程完成最后的回收工作。

```assembly
// 调用过程： SYS_exit->exit

1、先判断是否是用户进程，如果是，则开始回收此用户进程所占用的用户态虚拟内存空间;（具体的回收过程
不作详细说明）
2、设置当前进程的中hi性状态为PROC_ZOMBIE，然后设置当前进程的退出码为error_code。表明此时这个
进程已经无法再被调度了，只能等待父进程来完成最后的回收工作（主要是回收该子进程的内核栈、进程控制
块）
3、如果当前父进程已经处于等待子进程的状态，即父进程的wait_state被置为WT_CHILD，则此时就可以唤
醒父进程，让父进程来帮子进程完成最后的资源回收工作。
4、如果当前进程还有子进程,则需要把这些子进程的父进程指针设置为内核线程init,且各个子进程指针需要
插入到init的子进程链表中。如果某个子进程的执行状态是 PROC_ZOMBIE,则需要唤醒 init来完成对此子
进程的最后回收工作。
5、执行schedule()调度函数，选择新的进程执行。子进程的 id 号
```

- 用户态程序调用sys_exit()系统调用，通过syscall进入内核态。

- 内核态处理sys_exit()系统调用，调用do_exit()函数结束当前进程，最终返回到用户态。

#### 3.execve：完成用户进程的创建工作。首先为加载新的执行码做好用户态内存空间清空准备。接下来的一步是加载应用程序执行码到当前进程的新创建的用户态虚拟空间中。

```assembly
// 调用过程： SYS_exec->do_execve

1、首先为加载新的执行码做好用户态内存空间清空准备。如果mm不为NULL，则设置页表为内核空间页表，且进一步判断mm的引用计数减1后是否为0，如果为0，则表明没有进程再需要此进程所占用的内存空间，为此将根据mm中的记录，释放进程所占用户空间内存和进程页表本身所占空间。最后把当前进程的mm内存管理指针为空。

2、接下来是加载应用程序执行码到当前进程的新创建的用户态虚拟空间中。之后就是调用load_icode从而使之准备好执行。
```

- 用户态程序调用sys_exec()系统调用，通过syscall进入内核态。

- 内核态处理sys_exec()系统调用，调用do_execve()函数加载新的程序，但由于当前是在S mode下，无法直接进行上下文切换。因此使用ebreak产生断点中断，转发到syscall()函数，在该函数 中完成上下文切换，最终返回到用户态。

#### 4.wait：等待任意子进程的结束通知。`wait_pid`函数等待进程id号为pid的子进程结束通知。这两个函数最终访问`sys_wait`系统调用接口让ucore来完成对子进程的最后回收工作。

```assembly
// 调用过程： SYS_wait->do_wait

1、 如果 pid!=0，表示只找一个进程 id 号为 pid 的退出状态的子进程，否则找任意一个处于退出状态
的子进程;

2、 如果此子进程的执行状态不为PROC_ZOMBIE，表明此子进程还没有退出，则当前进程设置执行状态为
PROC_SLEEPING（睡眠），睡眠原因为WT_CHILD(即等待子进程退出)，调用schedule()函数选择新的进
程执行，自己睡眠等待，如果被唤醒，则重复跳回步骤 1 处执行;

3、 如果此子进程的执行状态为 PROC_ZOMBIE，表明此子进程处于退出状态，需要当前进程(即子进程的父
进程)完成对子进程的最终回收工作，即首先把子进程控制块从两个进程队列proc_list和hash_list中删
除，并释放子进程的内核堆栈和进程控制块。自此，子进程才彻底地结束了它的执行过程，它所占用的所有资
源均已释放。
```

- 用户态程序调用sys_wait()系统调用，通过syscall进入内核态。

- 内核态处理sys_wait()系统调用，调用do_wait()函数等待子进程退出，完成后返回到用户态。

请给出ucore中一个用户态进程的执行状态生命周期图（包执行状态，执行状态之间的变换关系，以及产生变换的事件或函数调用）

```assembly
(alloc_proc)          (wakeup_proc)
---------------> NEW ----------------> READY
                                         |
                                         |
                                         | (proc_run)
                                         |
        (do_wait)              (do_exit) V
EXIT <---------- ZOMBIE <-------------- RUNNING
```

![](https://github.com/Koreyoshiy/111/blob/main/lab5-1.png?raw=true)

### 扩展练习 Challenge

#### 题目2：说明该用户程序是何时被预先加载到内存中的？与我们常用操作系统的加载有何区别，原因是什么？

本次实验中，用户程序是在执行 execve 系统调用时被预先加载到内存中的。与我们常用的操作系统加载过程的区别主要在于执行上述加载步骤的时机。在常用的操作系统中，用户程序通常在运行时（runtime）才被加载到内存。当用户启动程序或运行可执行文件时，操作系统负责将程序从磁盘加载到内存，然后执行。这么做的原因是简化用户程序执行过程：预先加载用户程序可以简化用户程序的执行过程，使得执行过程更加直接和快速。

## 知识点总结

#### 实验执行流程概述

1. 编译器编译用户程序源代码为可执行的目标程序（包含执行代码&内存分配信息）

2. 根据分配信息为其分配内存，并将其加载进内存，为其建立进程（分配资源等）

3. 通过进程调度运行该程序

#### 系统调用

用户程序在用户态运行(`U mode`), 系统调用在内核态执行(`S mode)`。涉及CPU特权级切换的过程，`ecall` 可以从U到S。`S mode`调用OpenSBI 提供的` M mode` 接口。用` ecall` 调用进入`M mode`，剩下的事情交给底层的OpenSBI来完成。

#### 内核线程到用户进程

各个内核线程间的配合是较为协调的，能够相互考虑各自的资源利用情况，从而可以在适当的时候使用或不使用资源。而用户进程则相对利己，只考虑自身的资源利用情况，所以需要操作系统管理它们，让有效的资源得到协调而安全的利用。

#### 让用户进程正常运行的用户环境

用户环境由以下部分组成:

- 建立用户虚拟空间的页表和支持页换入换出机制的用户内存访存错误异常服务例程：提供地址隔离和超过物理空间大小的虚存空间。

- 应用程序执行的用户态 CPU 特权级：在用户态 CPU 特权级，应用程序只能执行一般指令，如果特权指令，结果不是无效就是产生“执行非法指令”异常；

- 系统调用机制：给用户进程提供“服务窗口”；

- 中断响应机制：给用户进程设置“中断窗口”，这样产生中断后，当前执行的用户进程将被强制打断，CPU 控制权将被操作系统的中断服务例程使用。

#### 用户态进程的执行过程

用户进程执行过程中，会发生系统调用、外部中断等造成特权级切换的情况，这时执行的进程到底是什么级别的进程呢？

从进程控制块（PCB）的角度理解，当PCB发生变化时、即执行了进程执行现场的切换，就认为是进程的分界点，因此可以把执行应用程序的代码一直到执行操作系统中的进程切换处为止都认为是一个应用程序的执行过程。

从指令执行/逻辑功能的角度理解，就需要进一步的分类讨论。对于用户进程主动的切换行为（如系统调用），进程代码包括用户和内核两部分。而对于用户进程被动的切换行为（如外设中断和CPU执行异常），切换后的代码逻辑对用户程序而言不可见，因此是另外一段执行逻辑。

#### 5状态模型

- 创建态：创建了进程控制块（PCB）时，但还未分配任何资源，之后会转变为就绪态

- 就绪态：对进程分配/准备了运行所需的各种资源（虚拟内存空间、可执行代码、所需数据等），但还未分配CPU资源，之后会转变为运行态

- 运行态：进程调度后实现了上CPU运行。之后会出现三种情况，分别对应切换到三种状态——等待某事件（阻塞态）、可用时间结束（就绪态）、程序运行结束（退出态）

- 阻塞态：由于需要等待某事件（硬件资源可用、某数据产生等）发生而进入的状态，之后会切换为就绪态

- 退出态：回收进程的各种资源，最后回收对应的进程控制块
