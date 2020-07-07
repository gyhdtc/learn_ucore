## 练习 2：实现寻找虚拟地址对应的页表项（需要编程）

通过设置页表和对应的页表项，可建立虚拟内存地址和物理内存地址的对应关系。其中的 get_pte 函数是设置页表项环节中的一个重要步骤。此函数找到一个虚地址对应的二级页表项的内核虚地址，如果此二级页表项不存在，则分配一个包含此项的二级页表。本练习需要补全 get_pte 函数 in kern/mm/pmm.c，实现其功能。请仔细查看和理解 get_pte 函数中的注释。
请在实验报告中简要说明你的设计实现过程。请回答如下问题：  
- 请描述页目录项（Page Directory Entry）和页表项（Page Table Entry）中每个组成部分的含义以及对 ucore 而言的潜在用处。  
- 如果 ucore 执行过程中访问内存，出现了页访问异常，请问硬件要做哪些事情？

----
get_pte函数作用是返回pte页表项虚地址。  
提供三个参数：  
1. pgdir: PDT的内核虚拟基地址；
2. la: 需要映射的线性地址；
3. create: 一个逻辑值，用来决定是否为PT分配一个页面；

他分两个情况：  
- 一个是页表目录存储的值有构建映射，直接转换地址就行了；  
- 一个是页表目录存储的值没有构建映射，我们需要创建一个page，给他们构建映射（赋值）；

这里我们需要知道一个常识，我以前一直不懂：  
- 页表目录和页表中，20位存储的都是基地址，是 **物理地址** ；如果要访问 表 ，还需要经行**物理-虚拟**转换；  

----
那我们可以看看具体操作了，如果索引正常，那么直接访问就行了：  
一个指针指向页表目录中的索引`uintptr *pte = &pgdir[PDX(la)]`
返回操作就是一步一步来了：  
`return &((pte_t *)KADDR(PDE_ADDR(*pdx)))[PTX(la)];`
|&|((uintptr_t * )|KADDR|(PDE_ADDR|(*pte)))|[PTX(la)]|
|----|----|----|----|----|----|
|||||读取PD中的值||
||||读取值中的前20位<br>|||
|||转换为PT虚地址||||
|取地址|转换为指针<br>指向PT的基地址||||线性地址中间10位<br>PT序号|

----
最麻烦的就是没有对应索引了，对应检查措施：  
```
if (!(*pdx & PTE_P)) {
    	struct Page *p;
    	if (!create || (p = alloc_page()) == NULL) {
    		return NULL;
    	}
```
`!(*pdx & PTE_P)`与操作，判断末尾是否为1；    
`!create || (p = alloc_page()) == NULL`看看是否需要创建、创建page是否成功；  
我觉得后面才是难点。
```
set_page_ref(p, 1);
uintptr_t pa = page2pa(p);
memset(KADDR(pa), 0, PGSIZE);
*pdx = pa | PTE_P | PTE_W | PTE_U;
```
看看组织结构图吧，page其实是在LA中分配的。  
上述第二行，获得了page管理的物理地址：**有一说一我不知道他有啥用。**  
实际上只是 Page 这个结构体所在的地址而已 故而需要 通过使用 page2pa() 将 Page 这个结构体的地址 转换成真正的物理页地址的线性地址 然后需要注意的是 无论是 * 或memset 都是对虚拟地址进行操作的所以需要将 真正的物理页地址再转换成 内核虚拟地址。  
![](https://img-blog.csdn.net/20171227160918632?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdGFuZ3l1YW56b25n/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)  
参考以下这个：  
https://www.cnblogs.com/kongj/p/12850111.html#%E7%BB%83%E4%B9%A02%EF%BC%9A%E5%AE%9E%E7%8E%B0%E5%AF%BB%E6%89%BE%E8%99%9A%E6%8B%9F%E5%9C%B0%E5%9D%80%E5%AF%B9%E5%BA%94%E7%9A%84%E9%A1%B5%E8%A1%A8%E9%A1%B9  
```
pte_t *
get_pte(pde_t *pgdir, uintptr_t la, bool create) {

	assert(pgdir != NULL);
	struct Page *struct_page_vp;	// virtual address of struct page
	uint32_t pdx = PDX(la), ptx = PTX(la);	// index of PDE, PTE

    pde_t *pdep, *ptep;  	
	pte_t *page_pa;			// physical address of page
	pdep = pgdir + pdx;
	ptep = (pte_t *)KADDR(PDE_ADDR(*pdep)) + ptx;

	// if PDE exists 
	if (test_bit(0, pdep)) {
		return ptep;
	}
	/* if PDE not exsits, allocate one page for PT and create corresponding PDE */
    if ((!test_bit(0, pdep)) && create) {           
		struct_page_vp = alloc_page();			// allocate page for PT
		assert(struct_page_vp != NULL);			// allocate successfully
		set_page_ref(struct_page_vp, 1);		// set reference count
		page_pa = (pte_t *)page2pa(struct_page_vp);	// convert virtual address to physical address
		ptep = KADDR(page_pa + ptx);			// virtual address of PTE
		*pdep = (PADDR(ptep)) | PTE_P | PTE_U | PTE_W;	// set PDE
		memset(ptep, 0, PGSIZE);				// clear PTE content
		return ptep;
	}	
    return NULL;     
```
从这里可以看出来，page2pa，其实可以获得这个page的物理地址，而不是上述的page管理的地址。这样理解就好多了。  