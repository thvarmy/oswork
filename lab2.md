# lab2:物理内存和页表

#### 练习1：理解first-fit 连续物理内存分配算法（思考题）

`default_init`函数用于初始化内存管理器，包括初始化空闲列表（free_list）和可用空闲内存块数（nr_free）。

`default_init_memmap`函数用于初始化内存映射，将一段内存块划分为页面，并将其添加到空闲列表中。

初始空闲内存块，参数是基址和页数

- `p->flags` 应设置为 PG_property 位（表示此页面有效）。在 `pmm_init` 函数中（在 `pmm.c` 中），`p->flags` 的 `PG_reserved` 位被设置。
- 如果此页面为空闲并且不是空闲块的第一个页面，则将 `p->property` 设置为0。
- 如果此页面为空闲并且是空闲块的第一个页面，则将 `p->property` 设置为块的总数。
- `p->ref` 应设置为0，因为此时 `p` 是空闲的，没有引用。
- 我们可以使用 `p->page_link` 将此页面链接到 `free_list`，例如：`list_add_before(&free_list, &(p->page_link));` 最后，我们应该计算空闲内存块的数量为n：nr_free+=n。

`default_alloc_pages`函数用于分配指定数量的物理页面。它会遍历空闲列表，找到第一个可用的页面块，并将其从空闲列表中移除然后重新调整空闲列表中空闲块的大小。如果找到的页面块大小大于需求的页面数量，会将多余的页面块切割出来，并添加回空闲列表。

- 在while循环中，获取struct page并检查p->property（记录空闲块的数量）是否大于n？
  struct Page *p = le2page(le, page_link);
  if (p->property >= n) { ...

- 如果我们找到了这个p，那么意味着我们找到了一个空闲块（块大小 >=n），并且可以分配前n页。
  此页面的一些标志位应设置为：PG_reserved =1，PG_property =0
  从空闲列表中取消链接这些页面

- 如果（p->property > n），我们应重新计算剩余的空闲块的数量
  （例如：le2page(le, page_link))->property = p->property - n;）

- 重新计算nr_free（所有剩余空闲块的数量）

- 返回p

`default_free_pages`函数用于释放已分配的物理页面。它会将页面块添加回空闲列表，并检查是否可以与相邻的空闲页面块合并。

- 根据要撤销的块的基本地址，搜索空闲列表，找到正确的位置
  （从低地址到高地址），并插入这些页面。 （可以使用list_next、le2page、list_add_before等操作）

- 重置页面的字段，如p->ref、p->flags（PageProperty）

- 尝试合并低地址或高地址的块。请注意：应正确更改某些页面的p->property值。

最后，代码中定义了一个名为`default_pmm_manager`的结构体，包含了物理内存管理器的各个函数指针和名称。

#### 练习2：实现 Best-Fit 连续物理内存分配算法（需要编程）

于First-Fit算法类似，实现Best-Fit算法。根据代码提示进行编写

- 清空当前页框的标志和属性信息，并将页框的引用计数设置为0
  
  ```
   ClearPageProperty(p);
   p->flags = 0;  
   set_page_ref(p, 0);
  ```

- 1、当base < page时，找到第一个大于base的页，将base插入到它前面，并退出循环
  
   2、当list_next(le) == &free_list时，若已经到达链表结尾，将base插入到链表尾部
  
  ```
  if(base<page){
      list_add_before(le, &(base->page_link));
      break;
   }else if(list_next(le) == &free_list){
       list_add(le, &(base->page_link));
       break;
   }
  ```

- best-fit遍历空闲链表，查找满足需求的空闲页框.如果找到满足需求的页面，记录该页面以及当前找到的最小连续空闲页框数量。只有这个地方与First-Fit算法不同，需要根据Best-Fit算法的思想进行调整与修改。
  
  ```
      struct Page *best_fit_page = NULL; // 用于记录当前找到的最佳页框
      while ((le = list_next(le)) != &free_list) {
          struct Page *p = le2page(le, page_link);
          if (p->property >= n && p->property - n < min_size) {
              // 如果该页框满足需求且比之前找到的最小尺寸小，更新最佳页框
              best_fit_page = p;
              min_size = p->property - n;
          }
      }
  
      if (best_fit_page != NULL) {
          // 找到最佳页框后，分配页框并更新空闲链表
          list_entry_t *prev = list_prev(&(best_fit_page->page_link));
          list_del(&(best_fit_page->page_link));
  
          if (best_fit_page->property > n) {
              struct Page *p = best_fit_page + n;
              p->property = best_fit_page->property - n;
              SetPageProperty(p);
              list_add(prev, &(p->page_link));
          }
          nr_free -= n;
          ClearPageProperty(best_fit_page);
          page = best_fit_page;
      }
  ```

- 设置当前页块的属性为释放的页块数、并将当前页块标记为已分配状态、最后增加nr_free的值
  
  ```
      base->property = n;
      SetPageProperty(base);
      nr_free += n;
  ```

- 1、判断前面的空闲页块是否与当前页块是连续的，如果是连续的，则将当前页块合并到前面的空闲页块中
  
  2、首先更新前一个空闲页块的大小，加上当前页块的大小
  
  3、清除当前页块的属性标记，表示不再是空闲页块
  
  4、从链表中删除当前页块
  
  5、将指针指向前一个空闲页块，以便继续检查合并后的连续空闲页块

```
        if (p + p->property == base) {
        p->property += base->property;
        ClearPageProperty(base);
        list_del(&(base->page_link));
        base = p;
        }
```

![](https://github.com/thvarmy/oswork/blob/master/picture/lab2/test.png?raw=true)

#### Challenge3：硬件的可用物理内存范围的获取方法（思考题）

1. 使用 BIOS 中断：使用BIOS中断来查询系统的硬件信息，包括可用的物理内存范围。这种方法通常需要在实模式下执行，可能需要特权级别切换，并且在不同的系统上可能有不同的BIOS中断调用。

2. 使用虚拟化接口：在虚拟化环境中，虚拟化平台通常提供了一些接口来获取硬件信息，包括可用的物理内存范围。

3. 使用启动参数：有些系统在引导操作系统时会传递额外的信息，包括可用的物理内存范围。这些信息通常以引导参数的形式传递给操作系统内核。在Linux中，可以使用`mem=`引导参数来指定可用的物理内存范围。

4. 手动配置：在某些情况下，手动配置操作系统以指定可用的物理内存范围。这需要深入了解硬件和操作系统的工作原理，并且可能需要修改操作系统内核或引导加载程序。

#### 实验总结

本实验中重要的知识点，以及与对应的OS原理中的知识点：

为了完成物理内存管理，首先需要探测可用的物理内存资源；了解到物理内存位于什么地方，有多大之后，就以固定页面大小来划分整个物理内存空间，并准备以此为最小内存分配单位来管理整个物理内存，管理在内核运行过程中每页内存，设定其可用状态（free的，used的，还是reserved的），这其实就对应了我们在课本上讲到的连续内存分配概念和原理的具体实现；接着ucore kernel就要建立页表，启动分页机制，让CPU的MMU把预先建立好的页表中的页表项读入到TLB中，根据页表项描述的虚拟页（Page）与物理页帧（Page Frame）的对应关系完成CPU对内存的读、写和执行操作。这一部分其实就对应了我们在课本上讲到内存映射、页表、多级页表等概念和原理的具体实现。



操作系统原理中重要但本实验中并未涉及的知识点：



①虚拟内存管理：lab2主要涉及物理内存管理，但没有涉及虚拟内存管理的具体实现和管理。虚拟内存管理是操作系统中重要的内存管理技术，用于将虚拟地址映射到物理地址，并提供更大的地址空间给应用程序使用；

②页面置换算法：lab2中没有涉及页面置换算法的具体实现和管理。页面置换算法是在内存不足时，选择合适的页面进行置换的算法，如最佳置换算法、先进先出算法等。
