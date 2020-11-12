# 练习1

> 一.注释翻译

> >  1. first-fit算法综述		

```
first-fit算法(简称FFMA)中,分配器保存了一个空闲块组成的表(俗称空闲表).当收到分配内存请求时,它依序遍历以找到第一个够大的能满足请求的块.
如果这个块比需求的大的多,那么它会被分割,没被选中的会分割出来的那块会成为表中的新空闲块.
应重写 `default_init`, `default_init_memmap`,`default_alloc_pages`, `default_free_pages`四个函数.
```

> > 2. 实现细节		

````
(1)准备

free_area_tleiixng定义了一个循环双链表空闲表free_list以及一个表项总数nr_free

list_entry指针转化为page指针的方法,存储在头文件`memlayout.h`中的宏`le2page`中.
````

```
(2)函数 `default_init`

可复用演示中的 `default_init`函数初始化空闲表`free_list`和给`nr_free`置零.`nr_free`指空闲块页数总和.
```

```
(3)函数 `default_init_memmap`

调用关系:`kern_init` --> `pmm_init` --> `page_init` --> `init_memmap` -->`pmm_manager` --> `init_memmap`

此函数用于初始化空闲块(使用基地址`addr_base`和页数`page_number`作为参数).

为初始化空闲块,先得初始化空闲块的每一页(按照头文件`memlayout.h`).此过程需要(p为`page`型指针)

- 将`p->flag`置为`PG_property`(表示此页合法).
	ps.在文件`pmm.c`中的`pmm_init`函数中,`p->flag`被初始化为`PG_reserved`
 如果此页空闲,且不为此块的第一页,那么`p->flag`置为 0
- 如果此页空闲,且为此块的第一页,那么`p->property`置为该块中的总页数
- 若p为空闲且未被引用,`p->ref`应该置为 0

然后使用`p->page_link`将此页其连接至空闲表`free_list`中.
	例 :`list_add_before(&free_list, &(p->page_link));` )

最后,更新`nr_free`的值
	例 :`nr_free += n`
```

```
(4)函数 `default_alloc_pages`

先找到空闲表中第一个够大的空闲块(块大小即页数大于`n`)并调整其大小,将其地址作为函数`malloc`的返回值.

一.遍历空闲表过程
list_entry_t le = &free_list;
while((le=list_next(le)) != &free_list) {...
	1.while循环中,创建一个page型结构体(*p指向该块首页)并检查其所在块大小是否大于n
 		struct Page *p = le2page(le, page_link);
        if(p->property >= n){ ...
    2.如果找到了这个块(*p),那意味着块大小符合要求,malloc函数能以此申请n页的空间.malloc函数请求其空间后,我们应将`PG_reserved`置为 1, `PG_property`置为 0, 并且断开其与空闲表的连接
    	ps.如果块大小大于n(`p->property > n`),我们应重新计算此空闲块中的空闲页数
    		例 :`le2page(le,page_link))->property = p->property - n;`
    3.重新计算剩余空闲页数总和`nr_free`
    4.返回这个块(*p)

二.如果不能找到够大的块,那么返回空值`NULL`
```

```
(5)函数 `default_free_pages`

将某页连接回空闲表(释放该页),并将某些小空闲块合并形成大空闲块.

1.根据将要释放的该页的基地址,找到空闲表中的适当位置以插入,因为空闲表是按地址从低到高排的.

2.重置该页的状态,例如给`p->ref`和`p->flags`置值.

3.尝试合并小块,但这将改变某些页的`p->property`值
```




>二.代码修改详解

​		根据上述的注释,我们可以依次对四个函数及其相关代码进行修改.

> > 1. `default_init`函数及其修改

```
static void
default_init(void) {
    list_init(&free_list);
    nr_free = 0;
}
```

该函数功能依次是初始化头节点和将空闲页数置零.

代码不需修改.

> > 2. `default_init_memmap`函数及其修改

```
static void
default_init_memmap(struct Page *base, size_t n) {
    assert(n > 0);
    struct Page *p = base;
    for (; p != base + n; p ++) {
        assert(PageReserved(p));
        p->flags = p->property = 0;
        set_page_ref(p, 0);
    }
    base->property = n;
    SetPageProperty(base);
    nr_free += n;
    list_add_before(&free_list, &(base->page_link));
}
```

该函数功能是,检查从给定基地址开始的大小为n页的空闲块,并将其加入空闲表中.

代码不需修改.

> > ​	3.`default_alloc_pages`函数及其修改

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
            list_add_after(&(page->page_link), &(p->page_link));//此处修改，将p在free_list的插入位置由表头改为page之后，因此删除page_link的操作也放在了插入p->pagelink之后
        }
        list_del(&(page->page_link));
        nr_free -= n;
        ClearPageProperty(page);//修改状态为已被alloc且为首页
    }
    return page;
}
```

该函数功能是,申请大小为n页的空闲页空间,并返回所需空间的首页的指针.

由于原函数中没有将所申请空闲块多余的部分的首页(即指针`p`指向的页)置为新的首页(即`flag`值设为`PG_property`),于是修改时调用`SetPageProperty()`函数修改`flag`值.

> > ​	4.`default_free_pages`函数及其修改

```c
static void
default_free_pages(struct Page *base, size_t n) {
    assert(n > 0);
    struct Page *p = base;
    for (; p != base + n; p ++) {
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
        if (base + base->property == p) {
            base->property += p->property;
            ClearPageProperty(p);
            list_del(&(p->page_link));
        }
        else if (p + p->property == base) {
            p->property += base->property;
            ClearPageProperty(base);
            base = p;
            list_del(&(p->page_link));
        }
    }
    nr_free += n;
    le = list_next(&free_list);
    while (le != &free_list) {
        p = le2page(le, page_link);
        if (base + base->property <= p) {
            assert(base + base->property != p);
            break;
        }
        le = list_next(le);
    }
    list_add_before(le, &(base->page_link));
}

```

该函数功能是,先取消对给定基地址开始的n页空闲页的引用,再将其加入空闲表中,然后合并小空闲块,最后进行按地址从低到高插入空闲表中的的合适位置.

原函数中,没有对新空闲页的相对于原空闲表的首页的地址进行大小对比,导致无法按照地址大小排序和插入新空闲页.