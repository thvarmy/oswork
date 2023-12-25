# Lab 8文件系统

**文件系统(file system)**，指的是操作系统中管理（硬盘上）持久存储数据的模块。

本次实验涉及的是文件系统，完善读写文件操作(即实现sfs_io_nolock()函数)，重新实现基于文件系统的执行程序机制（即实现load_icode()函数），从而可以完成执行存储在磁盘上的文件和实现文件读写等功能。

#### 文件系统初始化过程

kern_init函数中增加了对fs_init函数的调用。fs_init函数就是文件系统初始化的总控函数，它进一步调用了虚拟文件系统初始化函数vfs_init，与文件相关的设备初始化函数dev_init和Simple FS文件系统的初始化函数sfs_init。这三个初始化函数联合在一起，协同完成了整个虚拟文件系统、SFS文件系统和文件系统对应的设备（键盘、串口、磁盘）的初始化工作。

vfs_init主要建立了一个device list双向链表vdev_list，为后续具体设备（键盘、串口、磁盘）以文件的形式呈现建立查找访问通道。dev_init函数通过进一步调用disk0/stdin/stdout_device_init完成对具体设备的初始化，把它们抽象成一个设备文件，并建立对应的inode数据结构，最后把它们链入到vdev_list中。这样通过虚拟文件系统就可以方便地以文件的形式访问这些设备了。sfs_init是完成对Simple FS的初始化工作，并把此实例文件系统挂在虚拟文件系统中，从而让ucore的其他部分能够通过访问虚拟文件系统的接口来进一步访问到SFS实例文件系统。

### **文件系统概述**

处理具体设备，具体协议并向上提供简单接口的软件，我们叫做**设备驱动(device driver)，简称驱动**。

一段逻辑上相关联的数据看作一个整体，叫做**文件**。

假如应用程序操作文件（打开/创建/删除/读写），首先需要通过文件系统的`通用文件系统访问接口层`给用户空间提供的访问接口进入文件系统内部，接着由`文件系统抽象层`把访问请求转发给某一具体文件系统（比如 SFS 文件系统），具体文件系统（`Simple FS 文件系统层`）把应用程序的访问请求转化为对磁盘上的 block 的处理请求，并通过`外设接口层`交给磁盘驱动例程来完成具体的磁盘操作。结合用户态写文件函数 write 的整个执行过程，我们可以比较清楚地看出 ucore 文件系统架构的层次和依赖关系。

### 文件系统抽象层VFS

把不同文件系统的对外共性接口提取出来，形成一个`函数指针数组`，这样，通用文件系统访问接口层只需访问文件系统抽象层，而不需关心具体文件系统的实现细节和接口。

file&dir 接口层定义了进程在内核中直接访问的文件相关信息（kern/fs/file.h）

在 `kern/process/proc.h`中的 proc_struct 结构中加入了描述了进程访问文件的数据接口 files_struct（kern/fs/fs.h）

当创建一个进程后，该进程的 files_struct 将会被初始化或复制父进程的 files_struct。当用户进程打开一个文件时，将从 fd_array 数组中取得一个空闲 file 项，然后会把此 file 的成员变量 node 指针指向一个代表此文件的 inode 的起始地址。

`index node `是位于内存的索引节点，负责把不同文件系统的特定索引节点信息（甚至不能算是一个索引节点）统一封装起来，避免了进程直接访问具体文件系统。（kern/fs/vfs/inode.h）

### 硬盘文件系统SFS

![](C:\Users\33508\AppData\Roaming\marktext\images\2023-12-16-22-18-31-image.png)

第 0 个块（4K）是超级块（superblock），它包含了关于文件系统的所有关键参数，当计算机被启动或文件系统被首次接触时，超级块的内容就会被装入内存。（kern/fs/sfs.h/sfs_super）

第 1 个块放了一个 root-dir 的 inode，用来记录根目录的相关信息。root-dir 是 SFS 文件系统的根结点，通过这个 root-dir 的 inode 信息就可以定位并查找到根目录下的所有文件信息。

从第 2 个块开始，根据 SFS 中所有块的数量，用 1 个 bit 来表示一个块的占用和未被占用的情况。这个区域称为 SFS 的 freemap 区域，这将占用若干个块空间。为了更好地记录和管理 freemap 区域，专门提供了两个文件 kern/fs/sfs/bitmap.[ch]来完成根据一个块号查找或设置对应的 bit 位的值。

最后在剩余的磁盘空间中，存放了所有其他目录和文件的 inode 信息和内容数据信息。

在 sfs_fs.c 文件中的 sfs_do_mount 函数中，完成了加载位于硬盘上的 SFS 文件系统的超级块 superblock 和 freemap 的工作。这样，在内存中就有了 SFS 文件系统的全局信息。

在 SFS 文件系统中，需要记录文件内容的存储位置以及文件名与文件内容的对应关系。sfs_disk_inode (fs/sfs.h)记录了文件或目录的内容存储的索引信息。sfs_disk_entry 表示一个目录中的一个文件或目录，包含该项所对应 inode 的位置和文件名。占用一个 block。

默认的，ucore 里 SFS_NDIRECT 是 12，即直接索引的数据页大小为 12 *4k = 48k；当使用一级间接数据块索引时，ucore 支持最大的文件大小为 12* 4k + 1024 * 4k = 48k + 4m。数据索引表内，0 表示一个无效的索引，inode 里 blocks 表示该文件或者目录占用的磁盘的 block 的个数。indiret 为 0 时，表示不使用一级索引块。

操作系统中，每个文件系统下的 inode 都应该分配唯一的 inode 编号。每个 inode 直接用他所在的磁盘 block 的编号作为 inode 编号。

这些后缀为 nolock 的函数，只能在已经获得相应 inode 的 semaphore 才能调用。

### 设备

表示一个设备device（kern/fs/devs/dev.h）

 vfs_dev_t 数据结构（kern/fs/vfs/vfsdev.c）：把 device 和 inode 联通起来。让文件系统通过一个链接 vfs_dev_t 结构的双向链表找到 device 对应的 inode 数据结构，一个 inode 节点的成员变量 in_type 的值是 0x1234，则此 inode 的成员变量 in_info 将成为一个 device 结构。这样 inode 就和一个设备建立了联系，这个 inode 就是一个设备文件。

使用`iobuf`结构体（kern/fs/iobuf.h）传递一个IO请求（要写入设备的数据当前所在内存的位置和长度/从设备读取的数据需要存储到的位置）

设备文件的inode也有一个`inode_ops`成员, 提供设备文件应具备的接口。（kern/fs/devs/dev.c）

trap.c改变了对`stdin`的处理, 将`stdin`作为一个设备(也是一个文件), 通过`sys_read()`接口读取标准输入的数据。

在执行用户程序之前，我们先执行了`umain.c`建立一个运行时环境，这里主要做的工作，就是让程序能够使用`stdin`, `stdout`。

把命令行的输入转换成一个文件，需要一个缓冲区：把已经在命令行输入，但还没有被读取的数据放在缓冲区里。借助时钟中断，每次时钟中断检查是否有新的字符

`stdin`设备的实现（kern/fs/devs/dev_stdin.c）

`stdout`设备只需要支持写操作，调用`cputchar()`把字符打印到控制台。（

kern/fs/devs/dev_stdout.c）

disk0设备封装了一下`ramdisk`的接口，每次读取或者写入若干个block。（

kern/fs/devs/dev_disk0.c）

### open系统调用的执行过程

经过syscall.c的处理之后，进入内核态，执行sysfile_open()函数（

kern/fs/sysfile.c）

分配一个空闲的file数据结构变量file在文件系统抽象层的处理中，首先调用的是<mark>file_open</mark>函数（kern/fs/file.c）它要给这个即将打开的文件分配一个file数据结构的变量，这个变量其实是当前进程的打开文件数组current->fs_struct->filemap[]中的一个空闲元素（即还没用于一个打开的文件），而这个元素的索引值就是最终要返回到用户进程并赋值给变量fd。到了这一步还仅仅是给当前用户进程分配了一个file数据结构的变量，还没有找到对应的文件索引节点。

调用<mark>vfs_open</mark>函数来找到path指出的文件所对应的基于inode数据结构的VFS索引节点node。vfs_open函数需要完成两件事情：通过vfs_lookup找到path对应文件的inode；调用vop_open函数打开文件。

<mark>vfs_lookup</mark>函数是一个针对目录的操作函数，它会调用vop_lookup函数来找到SFS文件系统中的目录下的文件。vfs_lookup函数首先调用<mark>get_device</mark>函数,进一步调用vfs_get_bootfs函数,来找到根目录“/”对应的inode。这个inode就是位于vfs.c中的inode变量bootfs_node。这个变量在init_main函数（位于kern/process/proc.c）执行时获得了赋值。通过调用vop_lookup函数来查找到根目录“/”下对应文件sfs_filetest1的索引节点，，如果找到就返回此索引节点。

大量以vop开头的函数，它们都通过一些宏和函数的转发，最后变成对inode结构体里的inode_ops结构体的“成员函数”（实际上是函数指针）的调用。对于SFS文件系统的inode来说，会变成对sfs文件系统的具体操作。

在sfs_inode.c中的sfs_node_dirops变量定义了“.vop_lookup = sfs_lookup”(sfs_inode.c)

### Read系统调用执行过程

读文件其实就是读出目录中的目录项

先进入通用文件访问接口层的处理流程，即进一步调用如下用户态函数：read(uesr/libs/file.c)->sys_read(kern/fs/syscall/syscall.c)->syscall，从而引起系统调用进入到内核态。

vop_read函数实际上是对sfs_read的包装。在sfs_inode.c中sfs_node_fileops变量定义了.vop_read = sfs_read

### 从zhong duan(中断)到zhong duan（终端）

实现一个简单的终端(shell)：根据输入的程序名称, 从文件系统里加载对应的程序并执行。我们采取`fork()` `exec()`的方式来加载执行程序，`exec()`的一系列接口都需要重写来使用文件系统。以`do_execve()`为例，

终端程序的实现（user/sh.c）

## 练习0：填写已有实验

本实验依赖实验2/3/4/5/6/7。请把你做的实验2/3/4/5/6/7的代码填入本实验中代码中有“LAB2”/“LAB3”/“LAB4”/“LAB5”/“LAB6” /“LAB7”的注释相应部分。并确保编译通过。注意：为了能够正确执行lab8的测试应用程序，可能需对已完成的实验2/3/4/5/6/7的代码进行进一步改进。

```cpp
//LAB6 2112414 : (update LAB5 steps)
    /*
     * below fields(add in LAB6) in proc_struct need to be initialized
     *     struct run_queue *rq;                       // running queue contains Process
     *     list_entry_t run_link;                      // the entry linked in run queue
     *     int time_slice;                             // time slice for occupying the CPU
     *     skew_heap_entry_t lab6_run_pool;            // FOR LAB6 ONLY: the entry in the run pool
     *     uint32_t lab6_stride;                       // FOR LAB6 ONLY: the current stride of the process
     *     uint32_t lab6_priority;                     // FOR LAB6 ONLY: the priority of process, set by lab6_set_priority(uint32_t)
     */
    proc->rq = NULL;// 进程就绪队列指针初始化为NULL
        list_init(&(proc->run_link));// 初始化进程在就绪队列中的链接
        proc->time_slice = 0; // 时间片初始化为0
        proc->lab6_run_pool.left = proc->lab6_run_pool.right = proc->lab6_run_pool.parent = NULL;// Lab6运行队列初始化为NULL
        proc->lab6_stride = 0; // Lab6步幅初始化为0
        proc->lab6_priority = 0;// Lab6优先级初始化为0

     //LAB8 2112414 : (update LAB6 steps)
      /*
     * below fields(add in LAB6) in proc_struct need to be initialized
     *       struct files_struct * filesp;                file struct point        
     */
    proc->filesp = NULL;// 进程文件系统指针初始化为NULL
```

```cpp
void
proc_run(struct proc_struct *proc) {
    if (proc != current) {// 如果指定的进程不是当前运行的进程
         // LAB4:EXERCISE3 2112414
        /*
        * Some Useful MACROs, Functions and DEFINEs, you can use them in below implementation.
        * MACROs or Functions:
        *   local_intr_save():        Disable interrupts
        *   local_intr_restore():     Enable Interrupts
        *   lcr3():                   Modify the value of CR3 register
        *   switch_to():              Context switching between two processes
        */
        bool intr_flag;
        struct proc_struct *prev = current, *next = proc;// 获取当前运行的进程和将要运行的进程
        local_intr_save(intr_flag);// 保存中断状态并禁用中断
        {
            current = proc;// 切换当前运行的进程为指定的进程
            lcr3(next->cr3);// 切换页表，使用指定进程的页目录
            flush_tlb();//刷新 TLB（Translation Lookaside Buffer），清空页表缓存，确保使用新的页表信息
            switch_to(&(prev->context), &(next->context));// 执行上下文切换，将控制权从当前进程切换到指定的进程
        }
        local_intr_restore(intr_flag);
       //LAB8 2112414 : (update LAB4 steps)
      /*
     * below fields(add in LAB6) in proc_struct need to be initialized
    *       before switch_to();you should flush the tlb
    *        MACROs or Functions:
     *       flush_tlb():          flush the tlb        
     */

    }
}
```


## 练习1: 完成读文件操作的实现（需要编码）

首先了解打开文件的处理流程，然后参考本实验后续的文件读写操作的过程分析，填写在 kern/fs/sfs/sfs_inode.c中 的sfs_io_nolock()函数，实现读文件中数据的代码。

```cpp
if ((blkoff = offset % SFS_BLKSIZE) != 0) {//计算当前块内的偏移量 blkoff
        size = (nblks != 0) ? (SFS_BLKSIZE - blkoff) : (endpos - offset);//计算本次读写的数据块大小 size，考虑是否跨越多个块
        if ((ret = sfs_bmap_load_nolock(sfs, sin, blkno, &ino)) != 0) {//调用 sfs_bmap_load_nolock 加载块映射，并获取实际块号 ino
            goto out;
        }
        if ((ret = sfs_buf_op(sfs, buf, size, ino, blkoff)) != 0) {//调用 sfs_buf_op 进行缓冲区的读写操作
            goto out;
        }
        //更新累计长度 alen 和缓冲区指针 buf
        alen += size;
        buf += size;

        if (nblks == 0) {//如果没有要读写的块了，跳到标签 out
            goto out;
        }

        blkno++;
        nblks--;
    }
    //if (nblks > 0)：如果还有整块的数据需要读写
    if (nblks > 0) {
        if ((ret = sfs_bmap_load_nolock(sfs, sin, blkno, &ino)) != 0) {//调用 sfs_bmap_load_nolock 加载块映射，并获取实际块号 ino
            goto out;
        }
        if ((ret = sfs_block_op(sfs, buf, ino, nblks)) != 0) {//调用 sfs_block_op 进行整块的读写操作
            goto out;
        }
        //更新累计长度 alen、缓冲区指针 buf、块号 blkno 和块数量 nblks
        alen += nblks * SFS_BLKSIZE;
        buf += nblks * SFS_BLKSIZE;
        blkno += nblks;
        nblks -= nblks;//将 nblks 减去已经读写的块数量
    }
    //如果结束位置不是块对齐的
    if ((size = endpos % SFS_BLKSIZE) != 0) {
        if ((ret = sfs_bmap_load_nolock(sfs, sin, blkno, &ino)) != 0) {//调用 sfs_bmap_load_nolock 加载块映射，并获取实际块号 ino
            goto out;
        }
        if ((ret = sfs_buf_op(sfs, buf, size, ino, 0)) != 0) {//调用 sfs_buf_op 进行缓冲区的读写操作
            goto out;
        }
        // alen += size;/更新累计长度 alen
    }
```



## 练习2: 完成基于文件系统的执行程序机制的实现（需要编码）

改写proc.c中的load_icode函数和其他相关函数，实现基于文件系统的执行程序机制。执行：make qemu。如果能看看到sh用户程序的执行界面，则基本成功了。如果在sh用户界面上可以执行”ls”,”hello”等其他放置在sfs文件系统中的其他执行程序，则可以认为本实验基本成功。

```cpp
static int
load_icode(int fd, int argc, char **kargv) {
    assert(argc >= 0 && argc <= EXEC_MAX_ARG_NUM);
    if (current->mm != NULL) {
        panic("load_icode: current->mm must be empty.\n");
    }

    int ret = -E_NO_MEM;
    struct mm_struct *mm;
    if ((mm = mm_create()) == NULL) {
        goto bad_mm;
    }
    if (setup_pgdir(mm) != 0) {
        goto bad_pgdir_cleanup_mm;
    }

    struct Page *page;
/*由于在此处需要从磁盘中读取文件，所以需要根据`fd`获取`elf`头部的内容：
这里是定义了变量`__elf`，再为指针`elf`赋值，这种方式写可以省去`kmalloc`再`kfree`的步骤。*/
    struct elfhdr __elf, *elf = &__elf;
    if ((ret = load_icode_read(fd, elf, sizeof(struct elfhdr), 0)) != 0) {
        goto bad_elf_cleanup_pgdir;
    }

    if (elf->e_magic != ELF_MAGIC) {
        ret = -E_INVAL_ELF;
        goto bad_elf_cleanup_pgdir;
    }
/*这里可以发现与起始位置的偏移是`elf->e_phoff`，后面需要对`ph`进行遍历，于是可以改写为*/
    struct proghdr __ph, *ph = &__ph;
    uint32_t vm_flags, perm, phnum;
    for (phnum = 0; phnum < elf->e_phnum; phnum ++) {
        off_t phoff = elf->e_phoff + sizeof(struct proghdr) * phnum;
        if ((ret = load_icode_read(fd, ph, sizeof(struct proghdr), phoff)) != 0) {
            goto bad_cleanup_mmap;
        }
        if (ph->p_type != ELF_PT_LOAD) {
            continue ;
        }
        if (ph->p_filesz > ph->p_memsz) {
            ret = -E_INVAL_ELF;
            goto bad_cleanup_mmap;
        }
        if (ph->p_filesz == 0) {
            // continue ;
            // do nothing here since static variables may not occupy any space
        }
        vm_flags = 0, perm = PTE_U | PTE_V;
        if (ph->p_flags & ELF_PF_X) vm_flags |= VM_EXEC;
        if (ph->p_flags & ELF_PF_W) vm_flags |= VM_WRITE;
        if (ph->p_flags & ELF_PF_R) vm_flags |= VM_READ;
        // modify the perm bits here for RISC-V
        if (vm_flags & VM_READ) perm |= PTE_R;
        if (vm_flags & VM_WRITE) perm |= (PTE_W | PTE_R);
        if (vm_flags & VM_EXEC) perm |= PTE_X;
        if ((ret = mm_map(mm, ph->p_va, ph->p_memsz, vm_flags, NULL)) != 0) {
            goto bad_cleanup_mmap;
        }
        off_t offset = ph->p_offset;
        size_t off, size;
        uintptr_t start = ph->p_va, end, la = ROUNDDOWN(start, PGSIZE);

        ret = -E_NO_MEM;

        end = ph->p_va + ph->p_filesz;
        while (start < end) {
            if ((page = pgdir_alloc_page(mm->pgdir, la, perm)) == NULL) {
                ret = -E_NO_MEM;
                goto bad_cleanup_mmap;
            }
            off = start - la, size = PGSIZE - off, la += PGSIZE;
            if (end < la) {
                size -= la - end;
            }
            if ((ret = load_icode_read(fd, page2kva(page) + off, size, offset)) != 0) {
                goto bad_cleanup_mmap;
            }
            start += size, offset += size;
        }
        end = ph->p_va + ph->p_memsz;

        if (start < la) {
            /* ph->p_memsz == ph->p_filesz */
            if (start == end) {
                continue ;
            }
            off = start + PGSIZE - la, size = PGSIZE - off;
            if (end < la) {
                size -= la - end;
            }
            memset(page2kva(page) + off, 0, size);
            start += size;
            assert((end < la && start == end) || (end >= la && start == la));
        }
        while (start < end) {
            if ((page = pgdir_alloc_page(mm->pgdir, la, perm)) == NULL) {
                ret = -E_NO_MEM;
                goto bad_cleanup_mmap;
            }
            off = start - la, size = PGSIZE - off, la += PGSIZE;
            if (end < la) {
                size -= la - end;
            }
            memset(page2kva(page) + off, 0, size);
            start += size;
        }
    }
    sysfile_close(fd);

    vm_flags = VM_READ | VM_WRITE | VM_STACK;
    if ((ret = mm_map(mm, USTACKTOP - USTACKSIZE, USTACKSIZE, vm_flags, NULL)) != 0) {
        goto bad_cleanup_mmap;
    }
    assert(pgdir_alloc_page(mm->pgdir, USTACKTOP-PGSIZE , PTE_USER) != NULL);
    assert(pgdir_alloc_page(mm->pgdir, USTACKTOP-2*PGSIZE , PTE_USER) != NULL);
    assert(pgdir_alloc_page(mm->pgdir, USTACKTOP-3*PGSIZE , PTE_USER) != NULL);
    assert(pgdir_alloc_page(mm->pgdir, USTACKTOP-4*PGSIZE , PTE_USER) != NULL);
    
    mm_count_inc(mm);
    current->mm = mm;
    current->cr3 = PADDR(mm->pgdir);
    lcr3(PADDR(mm->pgdir));

/*设置参数
在计算长度时需要考虑每一个字符串末尾的`\0`。
由于用户函数执行需要参数`argc`和`argv`，所以需要在用户的栈中加入这两个参数，其中`argc`在`argv`之上。而`argv`中的每一项都需要从内核内存空间中拷贝到用户内存空间中，所以需要在栈中也为他们预留空间。*/
    //setup argc, argv
    uint32_t argv_size=0, i;
    for (i = 0; i < argc; i ++) {
        argv_size += strnlen(kargv[i],EXEC_MAX_ARG_LEN + 1)+1;
    }

    uintptr_t stacktop = USTACKTOP - (argv_size/sizeof(long)+1)*sizeof(long);
    char** uargv=(char **)(stacktop  - argc * sizeof(char *));
    
    argv_size = 0;
    for (i = 0; i < argc; i ++) {
        uargv[i] = strcpy((char *)(stacktop + argv_size ), kargv[i]);
        argv_size +=  strnlen(kargv[i],EXEC_MAX_ARG_LEN + 1)+1;
    }
    
    stacktop = (uintptr_t)uargv - sizeof(int);
    *(int *)stacktop = argc;
    
    struct trapframe *tf = current->tf;
    // Keep sstatus
    uintptr_t sstatus = tf->status;
    memset(tf, 0, sizeof(struct trapframe));
    tf->gpr.sp = stacktop; //最后还需要更新栈顶
    tf->epc = elf->e_entry;
    tf->status = sstatus & ~(SSTATUS_SPP | SSTATUS_SPIE);
    ret = 0;
out:
    return ret;
bad_cleanup_mmap:
    exit_mmap(mm);
bad_elf_cleanup_pgdir:
    put_pgdir(mm);
bad_pgdir_cleanup_mm:
    mm_destroy(mm);
bad_mm:
    goto out;
}
```
