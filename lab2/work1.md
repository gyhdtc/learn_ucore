## 练习 1：实现 first-fit 连续物理内存分配算法（需要编程）

在实现 first fit 内存分配算法的回收函数时，要考虑地址连续的空闲块之间的合并操作。提示:在建立空闲页块链表时，需要按照空闲页块起始地址来排序，形成一个有序的链表。可能会修改 default_pmm.c 中的 default_init，default_init_memmap，default_alloc_pages， default_free_pages 等相关函数。请仔细查看和理解 default_pmm.c 中的注释。

请在实验报告中简要说明你的设计实现过程。请回答如下问题：

你的 first fit 算法是否有进一步的改进空间

----
### 阅读注释
在First Fit算法中，分配器保持一个空闲块列表(称为空闲列表)。一旦收到内存分配请求，它就沿着列表扫描第一个足够大的块来满足请求。如果选择的块比请求的大得多，通常会将其拆分，其余的块将作为另一个空闲块添加到列表中。  
在 LAB2 EXERCISE 1 中，你可能需要修改函数 `default_init`, `default_init_memmap`，`default_alloc_pages`, `default_free_pages`.  

**FFMA 算法详情**  
- (1) 准备：  
为了完成这个算法，我们应该创建、管理一个链表，代码定义为 free_area_t。  
首先你应该熟悉`list` 在文件 list.h 中。结构体 `list` 是一个简单的双向链表。你应该知道如何使用以下函数：`list_init`, `list_add`(`list_add_after`), `list_add_before`, `list_del`, `list_next`, `list_prev`.  
> 有一个棘手的问题就是将`list` struct 转变为一个特殊的结构体（如 `page`），用以下宏指令：`le2page`(memlayout.h), (并且以后的实验中: `le2vma` (vmm.h), `le2proc` (proc.h), etc).  
- (2) `default_init`： 
用 `default_init` 函数来初始化 `free_list`。设置 `nr_free` 为 0。`free_list`用来记录内存。`nr_free` 是可以用内存块的总数。  
- (3) `default_init_memmap`：  
调用关系 : `kern_init` --> `pmm_init` --> `page_init` --> `init_memmap` -->
`pmm_manager` --> `init_memmap`.  
这个函数用于初始化一个空闲块(使用参数 `addr_base`,`page_number`). 为了初始化空闲内存块，首先要初始化这个空闲内存块中的每个`page` (定义在 in memlayout.h) 。 这个步骤包括：  
- 1. 设置比特位 `PG_property` of `p->flags`, 它代表页面有效。 另外在函数 `pmm_init` 中(in pmm.c), 比特位 `PG_reserved` of `p->flags` 已经搞定了。  
- 2. 如果该页是空闲的，并且不是空闲块的第一页，那么 `p->property` 应该被设置为0。  
- 3. 如果该页是空闲的，并且是空闲块的第一页，那么 `p->property` 应该被设置为总页数。
- 4. `p->ref` 应该为 0, 因为现在的 `p` 是空闲的，并且没有被引用。  
- 5. 我们可以用 `p->page_link` 来把页面加入 `free_list` 链表中。  
(比如 : `list_add_before(&free_list, &(p->page_link));` )  
- 6. 最后，我们应该更新空闲内存块的总和：`nr_free += n`。    
- (4) `default_alloc_pages`:  
搜索空闲列表中的第一个空闲块(块大小>= n)并调整找到的块的大小，返回该块的地址作为`malloc`函数所需要的参数。  
- (4.1) 所以首先你需要找到空闲块：  
```
list_entry_t le = &free_list;
while((le=list_next(le)) != &free_list) {
...
```
- (4.1.1)  
在while循环中, 获得结构体 `page` 并且检查 `p->property` 是否 >= n，(记录了block中的空闲页)。  
```
struct Page *p = le2page(le, page_link);
if(p->property >= n) { ...  
```
- (4.1.2)  
如果我们找到了这样的 `p`, 就意味着我们找到了一个页大小大于 n 的空闲block，它的开始 n 个page可以被分配。一些这个`page`的标志位会被设置为：`PG_reserved = 1`, `PG_property = 0`。  
然后, 把页从链表 `free_list` 去掉。    
- (4.1.2.1)  
如果 `p->property > n`, 我们应该重新计算这个空闲block的剩余`page`。(比如: `le2page(le,page_link))->property = p->property - n;`)  
- (4.1.3)  
重新计算 `nr_free` (剩余的空闲块).  
- (4.1.4)  
返回 `p`.  
- (4.2)  
如果找不到一个大小 >=n 的block, 返回 NULL.  
- (5) `default_free_pages`:  
重新把`page`连接进空闲块链表，并且可能会进行空闲块合并的操作。  
- (5.1)  
根据被移出块的基址，查找空闲列表的正确位置(地址由低到高)，插入`page`。(可能会用到 `list_next`, `le2page`, `list_add_before`)  
- (5.2)  
重新设置 `p->ref` and `p->flags` (page特权)  
- (5.3)  
尝试在高或者低地址合并块。注意: 这可能要改变 `p->property`。  

----
```
free_area_t free_area;

#define free_list (free_area.free_list)
#define nr_free (free_area.nr_free)
// 初始化，将他的两个指针都指向它自己；
// 双向链表个数为 0；
static void
default_init(void) {
    list_init(&free_list);
    nr_free = 0;
}
```
free_area_t的定义：
```
typedef struct {
    list_entry_t free_list;         // the list header
    unsigned int nr_free;           // # of free pages in this free list
} free_area_t;
```
list_entry_t的定义：定义为 list_entry，一个双向链表，有两个指针；
```
/* *
 * Simple doubly linked list implementation.
 * */

struct list_entry {
    struct list_entry *prev, *next;
};

typedef struct list_entry list_entry_t;
```