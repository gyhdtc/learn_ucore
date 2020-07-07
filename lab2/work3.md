## 练习 3：释放某虚地址所在的页并取消对应二级页表项的映射（需要编程）

当释放一个包含某虚地址的物理内存页时，需要让对应此物理内存页的管理数据结构 Page 做相关的清除处理，使得此物理内存页成为空闲；另外还需把表示虚地址与物理地址对应关系的二级页表项清除。请仔细查看和理解 page_remove_pte 函数中的注释。为此，需要补全在 kern/mm/pmm.c 中的 page_remove_pte 函数。  

请在实验报告中简要说明你的设计实现过程。请回答如下问题：  
- 数据结构 Page 的全局变量（其实是一个数组）的每一项与页表中的页目录项和页表项有无对应关系？如果有，其对应关系是啥？
- 如果希望虚拟地址与物理地址相等，则需要如何修改 lab2，完成此事？ 鼓励通过编程来具体完成这个问题

----
```
    if (*ptep & PTE_P) {
        struct Page *page = pte2page(*ptep);
        if (page_ref_dec(page) == 0) {
            free_page(page);
        }
        *ptep = 0;
        tlb_invalidate(pgdir, la);
    }
```
代码很简单。重点在于回答回答问题。  
本来想自己写的，不过看到了一个回答很好：  
https://www.cnblogs.com/kongj/p/12850111.html#%E7%BB%83%E4%B9%A02%EF%BC%9A%E5%AE%9E%E7%8E%B0%E5%AF%BB%E6%89%BE%E8%99%9A%E6%8B%9F%E5%9C%B0%E5%9D%80%E5%AF%B9%E5%BA%94%E7%9A%84%E9%A1%B5%E8%A1%A8%E9%A1%B9  
其中的一些概念，在 https://blog.csdn.net/qq_38410730/article/details/81071084 中。  