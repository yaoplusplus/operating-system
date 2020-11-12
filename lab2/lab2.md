



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

阅读代码我们可以知道default_init()函数的意义是初始化一个记录着所有空闲内存块的循环双链表free_list，（free_list的第一个表项和GDT一样也是空表项）（其定义在`list.h`中）并将其记录的空闲块数`nr_free`置为0

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

#### get_pte函数



### 练习三