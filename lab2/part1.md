# 写在前面

在开始写代码之前，先来看看源文件，记录一些关于虚拟内存的信息。以及其他的相关知识

## 页表相关信息

```c
#define NPDENTRIES	1024		// page directory entries per page directory
#define NPTENTRIES	1024		// page table entries per page table
```

这里两条分别是页表项和页目录项。其实也就是二级页表对应页中的PTE结构为10（页目录，也就是一级页表），10（二级页表），12（页内地址偏移）。

```c
#define PGSIZE		4096		// bytes mapped by a page
```

由此，得到的是一张页面的大小，是4KB，一张页正好可以存放1K个PTE，对应了上面的页目录项和页表项的大小

```c
#define PTXSHIFT	12		// offset of PTX in a linear address
#define PDXSHIFT	22		// offset of PDX in a linear address
```

这两个宏，分表表示的是页表索引和页目录索引的偏移量。所以如果我们获得了一个变量的地址为x（或是其他地址），那么将这个地址右移12位，那么就可以得到页表索引，同理得到页目录索引

这里，我们的页表索引是带着页目录索引的，我们可以这样理解，因为页表索引应该能够指示它是第几张页表，那么就当然应该是某一个页目录中的某一页才对应了完整的页表地址

## 回顾

上一节，我们进行了一个4MB页的映射，根据上面我们可以知道，这是一个页表索引所对应的所有页的大小。

在上一节讲义中所提及的代码`kern/entrypgdir.c`中，我们可以看到注释中提到`We choose 4MB because that's how much we can map with one page table `

## 内存布局

先总的来看.

通过lab1中可以知道, JOS的内存分为了三块

- 0x00000 - 0xA0000: 也即640KB的常规内存, 也就是代码中提到的basemem

- 0xA0000 - 0x100000: hole 以及 BIOS等

  Modern PCs therefore have a "hole" in physical memory from 0x000A0000 to 0x00100000, dividing RAM into "low" or "conventional memory" (the first 640KB) and "extended memory" (everything else).

- 0x100000 - 0x-------: 扩展内存,也即代码中提到的extmem(ext16mem),这是本节中主要讨论的部分

  其主要结构,可以参看`memlayout.c`中的结构图

# boot_alloc

*This simple physical memory allocator is used only while JOS is setting up its virtual memory system.  page_alloc() is the real allocator.* 这是注释中对这个函数功能的描述, 也即boot_alloc是一个简单的物理内存分配器, 从代码中我们可以看到, 这个分配的最小粒度是一页(PGSIZE=4KB)

所以boot_alloc就是一个内存分配器, 举例来说, 就像是c语言中malloc之类的内存分配函数(当然, 一个是直接对物理内存进行分配, 一个是分配的虚拟内存). 

这里用了一个很有意思的操作`extern char end[]`, end作为内核的bss段的结束位置, 所以通过end, 我们可以得到我们可以分配的内存的起始位置. 同时, 由于分配的最小粒度是一个4KB页, 下面使用

```c
nextfree = ROUNDUP((char *) end, PGSIZE);
```

进行了对齐操作, 将end地址向上对齐

之后就是我们的操作, 按照提示, 我们应该分为三种情况来看, 也即注释提及的n==0, n>0, 越界(内存不够用).

```c
if (n==0)
{
     return nextfree;
}
//the begin of the memory is result, and the end of the memory is nextfree
//the nextfree is added to exam whether we're out of memory and for the next alloc
result = nextfree;
nextfree += ROUNDUP(n, PGSIZE);

if(nextfree-KERNBASE > npages*PGSIZE){
	panic("out of memory(return NULL)!!!\r\n");
	nextfree = result;//reset the static nextfree;
	return NULL;
}

return result;
```

这里, 单独看一下n>0的两种情况

- 当够分配的时候, 我们将result作为初始地址分配出去即可, 同时,更新static变量nextfree,这样,我们才知道下一次再分配的时候, 从哪里开始

- 当越界的时候, 这里我的理解是, nextfree-KERNBASE得到的是总共使用了的物理地址. 所以当它超过了我们所能使用的全部内存时, 那么就算做out of memory了. 同时, 越界时返回NULL最好, 比较安全. 同时, 恢复nextfree.

  其中, npages可以看到对他的注释`size_t npages;			// Amount of physical memory (in pages)` 用页数表示的物理内存.


# mem_init

一句一句分析，从刚刚写完的boot_alloc开始

```c
	// create initial page directory.
	kern_pgdir = (pde_t *) boot_alloc(PGSIZE);
	memset(kern_pgdir, 0, PGSIZE);
```

这里我们，使用boot_alloc获取一个大小是PGSIZE(4KB)的内存空间，根据注释，这里要将这个页作为页目录索引。然后使用memset初始化它为全零。我们得到的`kern_pgdir`就是这个页目录的首地址.

```c
	// Permissions: kernel R, user R
	//PDX(UVPT) get the position of UVPT's index of PD
	//PADDR get the kern_pgdir's offset addr of the KERNBASE
	kern_pgdir[PDX(UVPT)] = PADDR(kern_pgdir) | PTE_U | PTE_P;
```

(以上两行注释是个人添加)

通过PDX(UVPT)来获取UVPT的页目录地址(也就是其地址的高10位,但是取到手就移到了低10位变成了数,具体可以看源代码),从`memlayout.c`中我们可以得到UVPT的具体值--0xef400000, 所以这里访问到的kern_pgdir中的位置就是11_1011_1101B, 那么在虚拟地址的这个位置, 就存放了一张物理页面的目录索引地址。记住这句话，在part2中会用到类似的操作

PADDR获取kern_pgdir相对于KERNBASE的偏移地址, 而KERNBASE就是我们虚拟地址的起始地址, 所以这里得到的结果其实也就是真实的物理地址.

后面两个或上的, 就是权限了, 暂时可以不管

那么这句话总体来看, 就是在kern_pgdir的0xef400000位置存入了页表目录的起始地址, 同时与物理地址做了映射.

这里同时还有一个要注意的点——我们的映射是将kern_pgdir的物理地址映射到了kern_pgdir[11_1011_1101B]的位置, 在对于kern_pgdir来说,并不是对齐的, 但是总的来说, 之后我们可以直接用变量来代替, 进而去访问这个位置的内存

```c
//your code
//这里要我们自己写代码了, 他是这样要求的
//Allocate an array of npages 'struct PageInfo's and store it in 'pages'.

pages = (struct PageInfo*)boot_alloc(npages * sizeof(struct PageInfo));
	memset(pages,0,npages * sizeof(struct PageInfo));
```

先把代码放出来, 再来做点解释——他的要求中有很多代码中的名词, 我水平不行, 多费了些功夫, 才看懂. 分配一个数组, 大小是npages个struct PageInfo, 并且将它储存在pages, pages他已经作为全局变量将我们定义好了.

所以, 我们通过boot_alloc获取足够的空间存放页表. 然后再用memset将其置为0即可.

这里的pages将用来作为映射所有的物理页面的信息.



<center>------------------切割----切割----------------------</center>

注释中这样说, 当我们初始化好这个info结构体之后, 后面的所有内存管理, 都将通过page_* function来实现了.也即,后面的操作都是围绕page上进行了

# page_init

先看看要让我们做什么—— *The example code here marks all physical pages as free. However this is not truly the case.  What memory is free?* 也就是说，他把所有的页都拿来使用了，但是有些不是free的，也即不是能够自由使用的。下面列举了从头到尾所有的内存类型，这里可以去看看`pre.md`中的内存结构

下面对`pre.md`进行补充。

- `page 0`，这里提到，page 0保留给实模式中断描述表和BIOS结构。

- rest memory就是640KB的保留内存，不过这里拿去了page 0

- IO hole（0xA0000-0x100000）

- ext memory：它说明了有些是给内核用了，有些是可以拿来用的。

  所以，对于ext memory的问题转变为，哪里开始是交由我们自己分配使用的，这我们上面提及到了end，这里之后是给我使用的。

  但是，我们又使用了一部分，用来存放pageInfo和页表目录。所以只要从这个地址后的，都是我们可以用的，也就是我们要获取到现在指向的地址，就可以了——而获取方法就是`boot_alloc(0)`, 得到`nextfree`

所以，有以下代码

```c
	size_t i;
	for (i = 0; i < npages; i++) {
		if(i == 0){
			pages[i].pp_ref = 0;
			pages[i].pp_link = NULL;
		}
		else if(i >= npages_basemem  
			&& i <= ((EXTPHYSMEM - IOPHYSMEM)/PGSIZE + ((uint32_t)boot_alloc(0) - KERNBASE)/PGSIZE)){
			pages[i].pp_ref = 0;
			pages[i].pp_link = NULL;
		}
		else{
			pages[i].pp_ref = 1;
			pages[i].pp_link = page_free_list;
			page_free_list = &pages[i];
		}
	}
```

两个连续区段也就是，第一个就是page 0；第二个就是IO hole和内核区，这两个在物理地址中是连接在一起的，所以一起小于（右边是开区间就可以，因为nextfree获取到的是当前free的界面）

其中，结构体中的几个数据，

- pp_ref就是指向这个页面的指针数，0则表示free
- pp_link当然就是下一个free的页面，最后将`page_free_list`作为了free page的头指针（过程采用头插法）

### 做到这里，我们可以查看第一个check了

在运行`make qemu-nox`之前，先注释掉`pmap.c`的第140行的panic。

运行qemu，会输出check_page_free_list() succeeded!，这就算是通过了`mem_init`中的第一个check。

# page_alloc

回想我们在写boot_loader时注释中说：boot_alloc只是一个简单的物理地址分配函数，只在建立虚拟地址空间的时候发挥作用，而真正的分配工作由page_alloc来完成，也即现在这个函数

所以函数page_alloc的功能还是分配以页为单位的物理空间（物理页），而boot_loader用来构建好了

```c
static inline physaddr_t
page2pa(struct PageInfo *pp)
{
	return (pp - pages) << PGSHIFT;
}

static inline void*
page2kva(struct PageInfo *pp)
{
	return KADDR(page2pa(pp));
}
```

提示我们去使用函数`page2kva`，这里我们先看源代码，该函数调用`page2pa`，借此获取页表索引（pages是page_info的起始指针）。所以这里函数`page2kva`的功能就是获取到页表索引的虚拟地址（KADDR我们之前见到过了，是将地址加上KERNBASE这个偏移量，将物理地址转换为虚拟地址）

```c
struct PageInfo *
page_alloc(int alloc_flags)
{
	if(page_free_list == NULL){
		//you can also throw a panic
		//panic("out of memory,return null");
		return NULL;
	} 
	//get head point
	struct PageInfo* head = page_free_list;
	//point the next,so that the head is the page we want
	page_free_list = page_free_list->pp_link;
	head->pp_link = NULL;

	if(alloc_flags & ALLOC_ZERO){
		memset(page2kva(head),'\0',PGSIZE);
	}

	// Fill this function in
	return head;
}
```

还是先把代码放出来。这里我们通过头指针page_free_list来获取到第一个page_info，之后，按照题目要求，当`alloc_flags & ALLOC_ZERO`成立的时候，就将这个页面指向的内存全部置为'\0'。所以，我们先找到这个页面所指向的开始地方，注释中提示我们使用page2kva函数，也即上代码中所示。最后返回head就算是ok了。

但是，写到这里，我产生了一个问题——页面（内存）之间是否会发生冲突。`page2kva`获取到了head，也就是页面信息表的头指针所对应的页面的起始地址的虚拟地址。代码里面可以看到这个索引是根据`head-pages`获取到的，也就是说，将head与pages的插值作为页的索引。 故而，页面的起始地址都是向下增长的。

为了得到一些更详细的信息，我在pmap.c中打上断点，调试，查看`page_free_list`和`pages`的值

```c
(gdb) p page_free_list
$1 = (struct PageInfo *) 0xf0119ff8
(gdb) p pages
$2 = (struct PageInfo *) 0xf0118000
```

也确实如此，pages指向页表数列的最尾，page_free_list指向头。

所以，只要接着往下减，那么总有一天，二者的差会变成0，最后拿出一张起始地址为0xf0000000的页面。但是这是大概不可能的，前面在page_init中，将一部分page的ref置为了1，也就是表示它们已经被使用，而且并没有被插入到page_free_list中，所以不会有被抽出的那一天。

这里也得到了一个信息，pages中存放了所有的物理页面，page_free_list中存放了所有可以使用的page（确切的描述应该是page的地址）

# page_free

根据注释，我们可以知道，page_free的最终结果是将page又放回到空闲页列表中。所以，我们先检测page此时是否处于正常状态，注释中所说，`pp_ref`和`pp_link`的状态

```c
void
page_free(struct PageInfo *pp)
{
	// Hint: You may want to panic if pp->pp_ref is nonzero or
	// pp->pp_link is not NULL.
	if(pp->pp_ref || pp->pp_link){
		panic("ooops!!some task is using this page.you can't free it now!\r\n");
	}
	//the meaning of free is the insert back the pp
	pp->pp_link = page_free_list;
	page_free_list = pp;
}
```

注意：这里的page_free_list插入时，当作无头指针的链表插入

这里，其实有一些问题可以联想到

- 这里的`pp_ref`表示指向使用此页面的程序数（或者说是任务数），但是我们在分配page的时候，并没有去进行ref的加减。

  注释中也有说到 *Does NOT increment the reference count of the page - the caller must do these if necessary (either explicitly or via page_insert).*

  这样为何还要进行检测呢，所有一开始`pp_ref`=1的页面都没有被进入到这个链表中，那么这个检测有什么用呢？这个变量肯定是有用的，这个检测肯定也是有用的，那么应该在什么时候进行增加操作呢？这里留下疑问，下面的part肯定可以知道

# end of part1

到这里，运行qemu，就可以看到两条success了。这样就算是大功告成了。

```shell
check_page_free_list() succeeded!
check_page_alloc() succeeded!
```

part1中，我们主要完成了这样两件事。

- 实现了一个简单的物理页内存分配`boot_alloc`
- 建立了一个合适的结构体列表去管理这些空闲页
