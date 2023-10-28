# lab3：缺页异常和页面置换

#### 练习1：理解基于FIFO的页面替换算法（思考题）

在FIFO页面置换算法下，一个页面从被换入到被换出的过程中，涉及到以下函数/宏的处理：  

1.`list_init(list_entry_t *list)`：初始化一个链表，将链表的头节点指针置为空。  

2.`list_add(list_entry_t *list, list_entry_t *entry)`：将一个节点插入到链表的末尾，即将entry节点添加到list链表的最后一个节点之后。  

3.`list_del(list_entry_t *entry)`：从链表中删除一个节点，即将entry节点从链表中断开。  

4.`list_prev(list_entry_t *list)`：获取链表中某个节点的前一个节点。  

5.`list_next(list_entry_t *list)`：获取链表中某个节点的后一个节点。  

6.`le2page(list_entry_t *le, pra_page_link)`：将链表节点转换为对应的Page结构体指针， 其中le是链表节点的指针，pra_page_link是Page结构体中用于链接的字段。

7.`_fifo_init_mm(struct mm_struct *mm)`：初始化FIFO算法所需的数据结构，将pra_list_head初始化为空链表。  

8.`_fifo_map_swappable(struct mm_struct *mm, uintptr_t addr, struct Page *page, int swap_in)`：将页面链接到FIFO算法的链表末尾，表示最近被访问。使用list_add将页面的pra_page_link字段添加到pra_list_head链表的末尾。  

9.`_fifo_swap_out_victim(struct mm_struct *mm, struct Page **ptr_page, int in_tick)`：选择最早被访问的页面作为置换的候选页面。使用list_prev获取pra_list_head链表的最前面的页面，然后使用list_del将其从链表中删除，并将页面的地址存储在ptr_page中。  

10.`list_entry_t`：链表节点的数据结构，包含指向前一个节点和后一个节点的指针。  

11.`assert(expression)`：断言函数，用于在代码中进行条件判断，如果条件不满足，则会触发断言错误。   

12.`_fifo_set_unswappable(struct mm_struct *mm, uintptr_t addr)`：将指定地址的页面设置为不可置换，即不会被FIFO算法置换出去。  

13.`_fifo_tick_event(struct mm_struct *mm)`：在时钟中断发生时调用的函数，用于处理FIFO算法的时钟事件。   

这些函数和数据结构在FIFO页面置换算法的实现中起到了重要的作用，包括链表的初始化、节点的插入和删除、页面的链接和选择以及算法的调试。例如，_fifo_init()函数用于初始化FIFO算法的全局数据结构；_fifo_set_unswappable()函数用于设置页面为不可置换；_fifo_tick_event()函数用于处理时钟中断事件；assert函数用于检查条件是否满足，确保算法的正确性。

#### 练习2：深入理解不同分页模式的工作原理（思考题）

> get_pte()函数（位于`kern/mm/pmm.c`）用于在页表中查找或创建页表项，从而实现对指定线性地址对应的物理页的访问和映射操作。这在操作系统中的分页机制下，是实现虚拟内存与物理内存之间映射关系非常重要的内容。
> 
> - get_pte()函数中有两段形式类似的代码， 结合sv32，sv39，sv48的异同，解释这两段代码为什么如此相像。
> - 目前get_pte()函数将页表项的查找和页表项的分配合并在一个函数里，你认为这种写法好吗？有没有必要把两个功能拆开？

get_pte()函数定义如下：

```cpp
pte_t *get_pte(pde_t *pgdir, uintptr_t la, bool create) {
    pde_t *pdep1 = &pgdir[PDX1(la)];
    if (!(*pdep1 & PTE_V)) {
        struct Page *page;
        if (!create || (page = alloc_page()) == NULL) {
            return NULL;
        }
        set_page_ref(page, 1);
        uintptr_t pa = page2pa(page);
        memset(KADDR(pa), 0, PGSIZE);
        *pdep1 = pte_create(page2ppn(page), PTE_U | PTE_V);
    }
    pde_t *pdep0 = &((pde_t *)KADDR(PDE_ADDR(*pdep1)))[PDX0(la)];//访问物理地址
//    pde_t *pdep0 = &((pde_t *)(PDE_ADDR(*pdep1)))[PDX0(la)];
    if (!(*pdep0 & PTE_V)) {
        struct Page *page;
        if (!create || (page = alloc_page()) == NULL) {
            return NULL;
        }
        set_page_ref(page, 1);
        uintptr_t pa = page2pa(page);
        memset(KADDR(pa), 0, PGSIZE);
 //       memset(pa, 0, PGSIZE);
        *pdep0 = pte_create(page2ppn(page), PTE_U | PTE_V);
    }
    return &((pte_t *)KADDR(PDE_ADDR(*pdep0)))[PTX(la)];
}
```

###### 回答问题：

1.get_pte()函数中的两段代码是用于处理不同级别的页表的。在RISC-V架构中，sv32、sv39和sv48分别代表32位、39位和48位的虚拟地址空间，这些不同的虚拟地址空间对应着不同级别的页表。  

- **sv32（32位虚拟地址，4级页表）**：在sv32模式下，虚拟地址被分为四个级别，分别是页全局目录（PGL）、页全局表（PML）、中间页表（PMD），以及页表（PT）。每个级别对应一个页表项，而页表项的结构是相同的，通常包括一个物理页号（PPN）和一些标志位，如有效位（V）、用户可读写位（U）等。

- **sv39（39位虚拟地址，4级页表）**：sv39模式下，虚拟地址仍然分为四个级别，但每级的索引位数和页表项结构略有不同。虚拟地址分为页全局目录（PGL）、页全局表（PML）、中间页表（PMD），以及页表（PT）。每级页表的页表项结构是相似的，仍然包括PPN和标志位。

- **sv48（48位虚拟地址，4级页表）**：在sv48模式下，虚拟地址也分为四个级别，但每级的索引位数和页表项结构不同于sv32和sv39。虚拟地址分为页全局目录（PGL）、页全局表（PML4）、中间页表（PDPT），以及页表（PTE）。虽然级别名称有所变化，但仍然具有相似的结构，包括PPN和标志位。

这两段代码相似是因为它们都是通过虚拟地址来查找对应的页表项（Page Table Entry，PTE）。虽然sv32、sv39和sv48的页表级别不同，但是他们的查找过程是类似的，都是从最高级的页目录（Page Directory）开始，根据虚拟地址的各个部分（例如，对于sv39，虚拟地址被分为三部分）来索引页表，直到找到对应的页表项。 不同的是，sv32、sv39和sv48的虚拟地址空间大小不同，因此虚拟地址的划分和页表的级别也不同。

具体来说，这两段代码的主要步骤如下：  

- 检查当前级别的页目录项（Page Directory Entry，PDE）是否存在。如果不存在，并且create参数为true，则分配一个新的页表页面，并将其地址设置为页目录项的值。  

- 如果页目录项存在，则根据虚拟地址的下一部分来索引页表，获取下一级的页目录项。  

2.get_pte()函数将页表项的查找和分配合并在一个函数中，这种写法有利有弊：

- 优点：
  
  - 简化了代码的调用。只调用一个函数，就可以完成页表项的查找和分配，无需关心具体的实现细节。
  
  - 提高了代码的效率。在查找页表项的过程中，如果发现需要的页表项不存在，可以直接在函数内部进行分配，无需再次遍历页表。

- 缺点：
  
  - 函数的职责不够单一。一般而言，一个函数对应一项功能，将查找和分配合并在一个函数中，可能会使函数的逻辑变得复杂，不利于理解和维护。
  
  - 函数的复用性降低。如果在其他地方只需要查找页表项，而不需要分配，那么这个函数就无法满足需求。  

一般而言，是否需要将查找和分配的功能拆开，取决于具体的需求和设计考虑。在本实验代码中，get_pte()函数在查找页表项时，如果发现页表项不存在，会根据create参数决定是否需要创建新的页表项。这种设计使得查找和创建页表项的操作紧密相关，且在大多数情况下，这两个操作是需要同时进行的。  

因此，将查找和创建页表项的功能合并在get_pte()函数中是合理的，没有必要将查找和创建页表项的功能拆开。这样可以简化函数调用，提高代码的效率。同时，由于create参数的存在，当只需要查找而不需要创建页表项时，也可以通过设置create为false来实现。  

#### 练习3：给未被映射的地址映射上物理页（需要编程）

`do_pgfault()`函数的作用是根据页面错误的类型和地址执行相应的操作，在给出的代码中，处理了页面的创建和分配出现的错误，要补充的代码段是页面交换发生错误时的处理方法。

首先调用`get_pte`函数尝试找到一个页表项，如果页表项不存在，则创建一个页表。如果找到的页表项的值为0，则调用`pgdir_alloc_page`函数来分配一个页面，并将该页面与给的那个地址建立映射，如果分配失败，则打印错误信息并跳转到`failed`标签处。

如果找到的页表项的值不为0，则执行一些与交换管理相关的操作。

首先，检查是否已经初始化了交换管理器（`swap_init_ok`），如果未完成初始化，则打印错误信息并跳转到`failed`标签处，如果已经完成初始化，则继续执行后续操作。声明一个名为`page`的指针变量，并将其初始化为`NULL`。

###### 代码编写部分：

1.调用`swap_in`函数根据给定的`mm`、`addr`和`page`，尝试将正确的磁盘页的内容加载到有`page`管理的内存中。然后将`page`的`pra_vaddr`属性设置为给定的地址。

```cpp
// (1) According to the mm AND addr, 
//try to load the content of right disk page into 
//the memory which page managed.
        if ((ret = swap_in(mm, addr, &page)) != 0) {
            cprintf("swap_in in do_pgfault failed\n");
            goto failed;
            }
        page->pra_vaddr = addr;
```

2.调用`page_insert`函数根据给定的`mm`、`addr`、`page`和权限，建立物理地址和逻辑地址之间的映射。

```cpp
// (2) According to the mm, addr AND page, 
//setup the map of phy addr <--> logical addr
        if ((ret = page_insert(mm->pgdir, page, addr, perm)) != 0) {
            cprintf("page_insert in do_pgfault failed\n");
            goto failed;
        }
```

3.调用`swap_map_swappable`函数将页面设置为可交换状态，以触发交换管理器记录该页面的访问情况。

```cpp
// (3) make the page swappable.
        swap_map_swappable(mm, addr, page, 1);
        page->pra_vaddr = addr;
```

###### 回答问题：

1.页目录项和页表项是用于管理虚拟内存的数据结构。它们在`ucore`实现页替换算法中可以提供页面状态信息、访问情况记录、页面交换管理和页面权限管理等功能。

- 存储页面状态信息：页目录项和页表项中的一些位可以用于存储页面的状态信息如页面是否被修改、是够被访问等，这些信息可以帮助算法选择最适合替换的页面。

- 记录页面访问情况：例如页面最近被访问的时间戳，这些信息可以用于实现基于页面访问频率的替换算法，如LRU算法。

- 管理页面交换：例如可以使用一个位来标记页面是否已经被交换到磁盘上，或者使用一个位来记录页面在磁盘上的位置，这些信息对于实现页面置换算法是必要的如页面交换或页面换入换出算法。

- 管理页面权限：例如页面是否可读、可写、可执行等。这些权限信息对于保护页面的完整性和安全性非常重要，同时也可以用于实现一些特定的替换策略，例如只替换可写页面或只替换可执行页面。

2.当ucore的缺页服务例程在执行过程中访问内存时出现了页访问异常，硬件将触发异常并将控制权转移到操纵系统的异常处理程序，该程序负责处理异常并执行相应的缺页处理逻辑。处理完成后，将控制权返回到用户程序，使其可以继续执行：

- 异常触发：硬件检测到页访问异常，例如缺页异常。

- 中断处理：硬件将终端当前的指令流，保存当前的上下文并切换到内核模式。

- 异常处理：硬件将控制权转移到操作系统的异常处理程序，由该程序负责处理页访问异常。

- 异常处理程序：操作系统的异常处理程序根据异常类型（页访问异常）和异常的原因（如：缺页）执行相应的处理逻辑。

- 缺页处理：在缺页异常的情况下，操作系统的异常处理程序将尝试处理缺页，通常包括以下步骤：
  
  - 确定缺页的原因和位置；
  
  - 分配物理内存页面；
  
  - 将缺页所需的数据从磁盘或其他存储介质加载到物理内存页面。
  
  - 更新页表或其他相关数据结构，建立正确的页面映射；
  
  - 恢复被中断的指令流，使程序继续执行

- 返回用户程序：一旦缺页处理完成，操作系统的异常处理程序将返回到用户程序，使其可以继续执行。

3.通过`Page`数组，可以索引访问和管理物理内存页面，而通过页目录项和页表项，可以访问和管理虚拟内存页面。`Page`数组的每一项与页表中的页目录项和页表项时一一对应的，他们共同用于管理虚拟内存和物理内存之间的映射关系。

- 页目录项：`Page`数组的每一项对应一个页目录项。页目录项中存储了页表的物理地址。

- 页表项：`Page`数组的每一项对应一个页表项。页表项中存储了物理页面的物理地址。

#### 练习4：补充完成Clock页替换算法（需要编程）

###### 代码编写：

初始化页面替换算法的结构：

```cpp
static int
_clock_init_mm(struct mm_struct *mm)
{     
     /*LAB3 EXERCISE 4: YOUR CODE*/ 
    list_init(&pra_list_head);
    // 初始化pra_list_head为空链表
    curr_ptr = &pra_list_head;
    // 初始化当前指针curr_ptr指向pra_list_head，表示当前页面替换位置为链表头
    mm->sm_priv = &pra_list_head;
    // 将mm的私有成员指针指向pra_list_head，用于后续的页面替换算法操作
    //cprintf(" mm->sm_priv %x in fifo_init_mm\n",mm->sm_priv);
    return 0;
}
```

将一个页面添加到页面链表，并设置该页面的访问位：

```cpp
static int
_clock_map_swappable(struct mm_struct *mm, uintptr_t addr, struct Page *page, int swap_in)
{
    list_entry_t *entry=&(page->pra_page_link);
    assert(entry != NULL && curr_ptr != NULL);
    //record the page access situlation
    /*LAB3 EXERCISE 4: YOUR CODE*/
    //link the most recent arrival page at the back of the pra_list_head qeueue.
    // 将页面page插入到页面链表pra_list_head的末尾
    list_add_before(curr_ptr, entry);
    page->pra_vaddr = addr;
    // 将页面的visited标志置为1，表示该页面已被访问
    pte_t *ptep = get_pte(mm->pgdir, addr, 0);
    assert(ptep != NULL);
    *ptep |= PTE_A; 
    return 0;
}
```

通过Clock页面置换算法，选择一个被换出的页面：

```cpp
list_entry_t *le = curr_ptr;
     struct Page *p;

while (1) {//遍历页面链表，查找最早未被访问的页面
        /*LAB3 EXERCISE 4: YOUR CODE*/ 
        le = list_next(le);
        if (le == head) {
            le = list_next(le);
        }
        p = le2page(le, pra_page_link);//获取当前页面对应的Page结构指针
        pte_t *ptep = get_pte(mm->pgdir, p->pra_vaddr, 0);
        if (*ptep & PTE_A) {//如果当前页面已被访问
            *ptep &= ~PTE_A;//则将visited标志置为0，表示该页面已被重新访问
            tlb_invalidate(mm->pgdir, p->pra_vaddr);
        } else {//如果当前页面未被访问
            list_del(le);//将该页面从页面链表中删除
            assert(p !=NULL);
            *ptr_page = p;//并将该页面指针赋值给ptr_page作为换出页面
            break;
        }
        curr_ptr = le;
        cprintf("curr_ptr %p\n",curr_ptr);
    }
```

###### 回答问题：

Clock页替换算法和FIFO在选择被换出页面时的策略上有所不同，相比于FIFO算法，Clock算法更能够考虑到页面的访问情况，从而提高系统性能。

- FIFO算法：FIFO算法是最简单的页面置换算法。它总是选择最早进入内存的页面进行替换。这种算法实现简单，但可能导致频繁访问的页面被替换出去，从而影响系统性能。  

- Clock算法：Clock算法是一种改进的FIFO算法。它通过一个循环链表（类似于时钟的指针）来记录所有在内存中的页面。每个页面都有一个访问位，当页面被访问时，访问位被设置为1。在选择被换出页面时，Clock算法从当前指针开始，顺时针检查每个页面的访问位。如果访问位为1，那么将访问位清零；如果访问位为0，那么选择该页面进行替换。这种算法可以避免FIFO算法中的问题，因为它倾向于替换最近未被访问的页面。  

#### 练习5：阅读代码和实现手册，理解页表映射方式相关知识（思考题）

**好处和优势：**

1. **更高的内存访问性能：** 使用大页可以减少页表的条目数，从而降低了页表的访问时间。这对于内存访问速度有显著的提升，特别是在大内存系统中。

2. **减少TLB（Translation Lookaside Buffer）缺失：** 大页表意味着更多的虚拟地址范围映射到同一个物理页框架，这减少了TLB缺失的可能性，因为一个TLB条目可以覆盖更多的虚拟地址范围。

3. **减少页表大小：** 大页表可以减小页表的大小，从而减少了对物理内存的消耗。这对于资源受限的系统（如嵌入式系统）可能特别有利。

4. **简化操作系统管理：** 大页表可以减少页表级别的层次结构，简化了操作系统的管理工作，因为操作系统需要维护更少的页表条目。

**坏处和风险：**

1. **内部碎片：** 大页表可能会导致内部碎片，因为一个大页可能包含多个不完全填满的小页。这可能会浪费一些物理内存。

2. **不适用于小内存系统：** 大页表的好处主要体现在大内存系统中。在小内存系统中，使用大页表可能会导致内存浪费，并不会提供明显的性能优势。

3. **不适用于稀疏内存分配：** 大页表不适用于需要稀疏内存分配的情况，因为它们会导致大页内的大量未使用内存。

4. **复杂的页替换算法：** 在大页表中实现页面替换算法可能更复杂，因为选择替换哪个大页可能比小页更复杂。

5. **不适用于多任务操作系统：** 在多任务操作系统中，不同的进程可能需要不同的页表大小，这可能会导致大页表无法适应各种内存访问模式。
