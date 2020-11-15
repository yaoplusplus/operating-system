



#  lab2物理内存管理

## 实验目的

> 理解基于段页式内存地址转换机制
>
> 理解页表的建立和使用方法
>
> 理解物理内存的管理方法

## 实验内容

> 了解如何发现系统中的物理内存
>
> 了解如何建立对物理内存的初步管理
>
> 了解如何建立页表来实现虚拟内存到到物理地址之间的映射及段页式内存管理机制

### 练习一

>  实现first-fit连续物理内存分配算法
>
> 重写`default_init`, `default_init_memmap`,`default_alloc_pages`, `default_free_pages`四个函数.

#### `default_init` 无需修改

```c
static void
default_init(void) {
    list_init(&free_list); 
    nr_free = 0;
}
```

```c
typedef struct {
    list_entry_t free_list;         // the list header
    unsigned int nr_free;           // # of free pages in this free list
} free_area_t;
```

```c
static inline void
list_init(list_entry_t *elm) {
    elm->prev = elm->next = elm;
}
```

阅读代码我们可以知道default_init()函数的意义是初始化一个记录着所有空闲内存块的循环双链表free_list，（其定义在`list.h`中）并将其记录的空闲块数`nr_free`置为0

#### `default_init_memmap` 无需修改

```c
static void
default_init_memmap(struct Page *base, size_t n) {
    assert(n > 0);										//检验n的合法性
    struct Page *p = base;									
    for (; p != base + n; p ++) {						//逐页初始化page
        assert(PageReserved(p));						//page不能是内核预留的
        p->flags = p->property = 0;						//page不是首页
        set_page_ref(p, 0);								//page未被引用
    }
    base->property = n;									//base 为首页且总页数为n
    SetPageProperty(base);								//base设置为空闲且为首页
    nr_free += n;
    list_add(&free_list, &(base->page_link));			//插入free_list后
}
```

`default_init_memmap`的功能是由基地址`base`开始初始化一段首页地址为`base`的页数为n的空闲块，并将其插入free_list中。

#### `default_alloc_pages` 有修改

```c
static struct Page *
default_alloc_pages(size_t n) {
    assert(n > 0);
    if (n > nr_free) {
        return NULL;
    }
    struct Page *page = NULL;
    list_entry_t *le = &free_list;
    while ((le = list_next(le)) != &free_list) {
        struct Page *p = le2page(le, page_link);
        if (p->property >= n) {
            page = p;
            break;
        }
    }
    if (page != NULL) {
        if (page->property > n) {
            struct Page *p = page + n;
            p->property = page->property - n;
            SetPageProperty(p); //此处修改，将p->flag的PG_property位设为1，意思是p空闲且为首页
            list_add_after(&(page->page_link), &(p->page_link));
//此处修改，将p在free_list的插入位置由表头之后改为page之后，因此删除page_link的操作也要放在插入p->pagelink操作之后
        }
        list_del(&(page->page_link));
        nr_free -= n;
        ClearPageProperty(page);//修改首页状态为已被alloc且为首页
    }
    return page;
}
```

功能为处理一个申请页数为n的内存分配，在`free_list`中从头遍历，找到第一个满足申请大小的空闲块，并且，如果该块的大小m大于所申请大小n，将该空闲块分割为大小为n和m-n的两块，从`free_list`中插入后者，删除前者。

####  `default_free_pages` 有修改

````c
static void
default_free_pages(struct Page *base, size_t n) {
    assert(n > 0);
    struct Page *p = base;
    for (; p != base + n; p ++) {//
        assert(!PageReserved(p) && !PageProperty(p));
        p->flags = 0;
        set_page_ref(p, 0);
    }
    base->property = n;
    SetPageProperty(base);
    list_entry_t *le = list_next(&free_list);
    while (le != &free_list) {
        p = le2page(le, page_link);
        le = list_next(le);
        if (base + base->property == p) {//合并较高地址的空闲块，合并后首页地址为base
            base->property += p->property;
            ClearPageProperty(p);
            list_del(&(p->page_link));
        }
        else if (p + p->property == base) {//合并较低地址的空闲块，合并后首页地址为p
            p->property += base->property;
            ClearPageProperty(base);	
            base = p;
            list_del(&(p->page_link));
        }
    }
    nr_free += n;
    //下方代码有修改
    le = list_next(&free_list);				//增加一个循环用来判断无法合并其余空闲块的情况
    while (le != &free_list) {
        p = le2page(le, page_link);
        if (base + base->property <= p) {	//找到一个地址正好高于base空闲块的空闲块
            assert(base + base->property != p);
            break;
        }
        le = list_next(le);
    }
    list_add_before(le, &(base->page_link));	//将base页的page_link插入free_list
}
````

`default_free_pages`的功能是将内存块重新链接进`free_list`，并且执行可能的合并块的操作。

### 练习二

> 实现寻找虚拟地址和对应的页表项
>
> 请描述页目录项和页表中每个组成部分的含义和以及对ucore而言的潜在用处
>
> 如果ucore执行过程中存在访问内存，出现了页访问异常，请问硬件要做哪些事情

注释

![image-20201115213050948](image-20201115213050948.png)

```
pte:Page Table Entry 页表项 (二级页表项)
pde：Page Director Entry 页目录项(一级页表项)
get_pte函数：获取某一页表项，并以线性地址的形式返回其内核虚拟地址
la:需要寻址的内存的线性地址(在x86的分段地址转换下的线性地址)
create: 如果为页表分配一页将要设置的一个逻辑值
return：这个页表项的内核虚拟地址

使用KADDR()访问一个物理地址
阅读pmm.h 了解有用的宏
宏或函数：
PDX(la)   ： 虚拟地址la的页目录项的索引
KADDR(pa) ：  转换一个物理地址为其对应的内核虚拟地址
set_page_ref(page,1) ：页面被引用了一次
page2pa(page): 返回当前页面的物理地址
alloc_page() : 分配地址
memset(void *s, char c, size_t n) :将一片由s指向的内存区域的前n个byte初始化为c
PTE_P           0x001 ：在
PTE_W           0x002 ：可写
PTE_U           0x004 ：用户可访问
```

```c
pte_t *
get_pte(pde_t *pgdir, uintptr_t la, bool create) {
pde_t *pdep = &pgdir[PDX(la)]; //通过线性地址查找到对应的页目录表项的索引并使pde_t指向该页表目录项
    if (!(*pdep & PTE_P)) {		//检查其PTE_P位是否为1(虚拟页面是否在物理页表中)
        //虚拟页面不在页框中，则分配页框    
        struct Page *page;		
        if (!create || (page = alloc_page()) == NULL) {//分配页框
            return NULL;
        }
        set_page_ref(page, 1);			//设置页的引用值
        uintptr_t pa = page2pa(page);	 //得到页面物理地址
        memset(KADDR(pa), 0, PGSIZE);	 //用0初始化(清理)页面
        *pdep = (pa & 0x0FFF)| PTE_U | PTE_W | PTE_P; //填写目录表项内容
        
    }
    return &((pte_t *)KADDR(PDE_ADDR(*pdep)))[PTX(la)];
    //返回la所在页表项的地址
    //PDE_ADDR(*pdep) ：舍弃*pdep的低12位
    //KADDR(PDE_ADDR(*pdep)) 获取对应虚拟地址
    //(pte_t *)KADDR(PDE_ADDR(*pdep)) 转换为页表指针
    }
    
```

```c
#define PTE_ADDR(pte)   ((uintptr_t)(pte) & ~0xFFF) 
#define PDE_ADDR(pde)   PTE_ADDR(pde)

```

Question:

> 为什么一个物理页面的大小为4096位？
>
> 为什么高20位作为页面数
>
> page2pa

### 练习三

> 释放某虚地址所在的页并取消对应二级页表项的映射
>
> 当释放一个包含某虚地址的物理内存页时，需要让对应此物理内存页的管理数据结构Page做相关的清除处理，使得此物理内存页成为空闲；另外还需把表示虚地址与物理地址对应关系的二级页表项清除。为此，需要补全在kern/mm/pmm.c中的page_remove_pte函数。

先读注释进行分析：

使用pte2page宏可以得到从页表项得到对用的物理页面对应的Page结构体，得到结构体之后，判断此页被引用的次数，如果只被引用一次，则对页面引用进行更改，降为0这个页可以被释放；如果被多次引用，则释放页表入口，清除二级页表项。

```c
if (*ptep & PTE_P) {//判断页表是否存在
        struct Page *page = pte2page(*ptep);//从页表项得到对应的物理页对应的Page结构体
        if (page_ref_dec(page) == 0) {//判断此页是否被多次引用
            free_page(page);
        }
        *ptep = 0;
        tlb_invalidate(pgdir, la);//使TLB无效，清除二级页表项
    }
```

> 数据结构Page的全局变量（其实是一个数组）的每一项与页表中的页目录项和页表项有无对应关系？如果有，其对应关系是啥？

有对应关系

在mmu.h中找到pte2page函数的定义，从页表项找到对应物理页的Page结构体，Page结构体和对应的物理页的起始地址是一一对应的

```c
static inline struct Page *
pte2page(pte_t pte) {
    if (!(pte & PTE_P)) {
        panic("pte2page called with invalid pte");
    }
    return pa2page(PTE_ADDR(pte));
}
```

> 如果希望虚拟地址与物理地址相等，则需要如何修改lab2，完成此事？鼓励通过编程来具体完成这个问题

由ld工具形成的ucore的起始虚拟地址是0xC0100000，bootloader把ucore放在了起始物理地址0x100000

PA（物理地址）=LA（线性地址）=VA（逻辑地址）-KERNBASE

先把KERNBASE从0xC000000改成0x00000000，再把起始虚拟地址从0xC0100000改成0x00100000