# 写在前面

开始之前，我们先看看，他的Exercise2中提到的这个[Linear Address]([80386 Programmer's Reference Manual -- Table of Contents (mit.edu)](https://pdos.csail.mit.edu/6.828/2018/readings/i386/toc.htm))

## x86的段与页

*In x86 terminology, a virtual address consists of a segment selector and an offset within the segment. A linear address is what you get after segment translation but before page translation. A physical address is what you finally get after both segment and page translation and what ultimately goes out on the hardware bus to your RAM.*

这里对于段页地址转换有一个基本流程，就是由段的（logic address（virtual address））转换到页表的（linear address），最后再转换为物理地址（physical address)。然后到这里才能去RAM或者其他储存器中取出指令或者地址。所以说，这样的结构总共需要三次对内存的访问

#### 段

为了执行段的转换工作，需要段描述符，描述表，段选择子和段寄存器。下面一个个来看

- 段描述符：由软件例如编译器，链接器等创建。其中主要记载了段的一些属性，理解基址，大小，访问位，存在位等等

- 段描述表：段描述符是段描述表的一个元素。段描述表分为LDT（局部描述表）和GDT（全局描述表）。两个表的地址都分别保存在GDTR和LDTR。

  段描述表中存有很多的段描述符，这样就可以支持多个任务并行，每个任务都可以在同一时间拥有自己的段描述符。同时，操作系统并不直接访问段描述符，而是先找到段描述表，从中得到段描述符，进而对段描述表的控制，就进而修改了段描述符

- 选择子(Selector)：上面我们说到了段描述符，但是如何定位到段描述标中的段描述符呢——通过选择子来定位
- 段寄存器：80386中将描述符中的信息储存到段寄存器中。同时段寄存器中又分为可见与不可见的部分——16bit的段地址由程序控制，当然是可见的；不可见的其余段描述符部分，由cpu操控

#### 页

页的转换不是必要的，如果要进行virtual 8086 tasks（保护模式下多任务并行多个实模式程序（DOS））, page-oriented protection（页面级别的内存保护机制）, or page-oriented virtual memory（以页为单位的虚拟内存管理）。

几个有关页的专有名词

- 页框(page frame)：就是我们常说的多级页表中的最后一级，4KB连续空间的地址
- 线性地址(linear address)：也就是一个PTE中的内容，在二级页表中包含页表目录，页表索引，地址偏移
- 页表，页目录：这里这样描述到页目录和页表——它们本身都是可以容纳4KB的一个页，在二级页表中，第一级为页目录。如此类推，就可以得到二级页表中，应该有1k个目录项，1M个页表索引项，4GB的页框大小

#### 页级别的保护机制

- Restricting Addressable Domain（限制可寻址域）：这个限制主要是区分两种用户——supervisor和user。
- Type Checking（类型检查，或者较为操作类型检查更好）：也是区分于supervisor和user的读写权限。对于用户来说，只有处于用户级别的页面才允许读或者写。

# Exercise3

练习3主要是查看虚拟内存和物理地址之间的映射关系。但是我们在part1中还没有建立物理和虚拟之间的映射，只是完成了物理页面内存的分配和相关结构体的实现。所以这里去查看的，主要是我们在lab1中，完成了的0xf0100000开始的1MB的页表映射，所以这里查看的话，单纯是指这4MB的内存

这里主要依靠两个指令，一个是gdb命令`x/Nx addr`——用来查看从addr往后N个字的虚拟内存内容16进制转储；一个是qemu中的指令`xp/Nx paddr`——用来查看paddr往后N个字的物理地址。

同时我们先让gdb要运行过启动页表之后，也不用打断点，直接`c`，让内核运行起来就行了。

```shell
##gdb中调试
(gdb) x/16xw 0xf0100000
0xf0100000 <_start-268435468>:	0x1badb002	0x00000000	0xe4524ffe	6
0xf0100010 <entry+4>:	0x34000004	0x5000b812	0x220f0011	0xc02008
0xf0100020 <entry+20>:	0x0100010d	0xc0220f80	0x10002fb8	0xbde0f0
0xf0100030 <relocated+1>:	0x00000000	0x113000bc	0x0002e8f0	0

##qemu中
(qemu) xp/16x 0x100000
0000000000100000: 0x1badb002 0x00000000 0xe4524ffe 0x7205c766
0000000000100010: 0x34000004 0x5000b812 0x220f0011 0xc0200fd8
0000000000100020: 0x0100010d 0xc0220f80 0x10002fb8 0xbde0fff0
0000000000100030: 0x00000000 0x113000bc 0x0002e8f0 0xfeeb0000

#mem目前映射关系
(qemu) info mem
0000000000000000-0000000000400000 0000000000400000 -r-
00000000f0000000-00000000f0400000 0000000000400000 -rw
```

  紧接着，他提出了一个问题，就是内核有时需要去访问他所知道物理地址的内存，但是内核无法绕过虚拟地址转换的限制，所以不能直接访问到物理地址，因此，他给出了一个解决办法，也就是我们早就使用的方法，就是使用KADDR函数，来获取对应物理地址的虚拟地址，使用PADDR来获取对应虚拟地址的物理地址

写到这，我突然想到，如果只是将地址做一个差就可以绕过虚拟地址的话，那么我们直接去做一个差，或者直接去做一个低地址访问，会发生什么

```c
#include<stdio.h>

int main()
{
        unsigned int* s = (unsigned int*)0x100000;
        *s = 10;
        printf("%x",s);
}
```

编译运行

```shell
root@ubuntu:/home/cwb/Documents/6.828# ./a.out 
Segmentation fault (core dumped)
```

不出意料的报错了，内存是不允许访问的，受操作系统保护，也是虚拟内存的手段。经过查询，在内核模式下可以直接对硬件地址进行操作。

# Exercise4

有关[MMU]([MMU (一） - blackBox - 博客园 (cnblogs.com)](https://www.cnblogs.com/ikaka/p/3602536.html))的说明。

## pgdir_walk

这个函数的重点是区分开物理内存和虚拟内存。我们在程序中获取到的变量地址都是虚拟地址。在PTE中记录的都是物理地址，且PTE结构都相同，即31-12位为物理地址，低位为标志位.

该函数只要求做一件事，就是返回va所对应的页的PTE。同时，对于没有存在在页表中的地址，建造新的页接收它

```c
pte_t *
pgdir_walk(pde_t *pgdir, const void *va, int create)
{
	// Fill this function in
	//get the pd and pt
	uintptr_t pd_index = PDX(va);
	uintptr_t pt_index = PTX(va);
	pte_t *pte;
	pde_t *pde;

	struct PageInfo* pi;

	pde = &pgdir[pd_index];//pde now store the content of the directory
	if(*pde & PTE_P){
		//the second page table exits,the page frame address specifies a physical address
		pte = (pte_t*)KADDR(PTE_ADDR(*pde));
	}
	else{
		if(create==false){
			return NULL;
		}
		if(!(pi=page_alloc(ALLOC_ZERO))){
			//fail to alloc
			return NULL;
		}
		pte = (pte_t*)page2kva(pi);
		pi->pp_ref++;
		*pde = PADDR(pte) | PTE_U | PTE_W | PTE_P;
	}
	return &pte[pt_index];
}
```

做几点解释：

- 直接看当没有创建过时，拆开来看：对于二级页表中所存PTE，其实就是每个物理页面的起始地址，offset来自va的低12位，但是我们这里只要得到pte即可，所以对于这个offset并不感兴趣；同时这里让PTE指向每个页面所对应的虚拟地址，这也就看成了一种将这里开始的页面分配给了二级页表；（用更夸张的说法，几乎所有的内存现在都是分配给每个指针的，所以掌控权完全交给程序员）

- `page2kva`的使用，我们在上一个part中说明过，传入一个PageInfo，返回一张物理页面所对应的虚拟地址。

- `*pde = PADDR(pte) | PTE_U | PTE_W | PTE_P`,这句代码，我们在说到part1中的`mem_int`时有提到`kern_pgdir[PDX(UVPT)] = PADDR(kern_pgdir) | PTE_U | PTE_P`，也即建立起和物理地址之间的映射

  可以再重新理解一下映射，映射其实不是一件很高级的事，映射的本质是地址的对映，或者说是通过MMU的地址的变换，而这个映射的关系通过页表来解释，所以这里的这句代码，就是**将物理地址包括标志位存入页表项中**，这就完成了映射。
  
- 还有一件事，很重要！！！在`pgdir_walk`中，其实只建立了一个映射关系，这个映射是页目录（一级页表）和页面索引（二级页表）中的关系。

## boot_map_region

有了上一题的基础，这一题就简单许多了，因为是类似的——都是对指定的地址做映射

```c
static void
boot_map_region(pde_t *pgdir, uintptr_t va, size_t size, physaddr_t pa, int perm)
{
	//size is multiple of PGSIZE
	size_t num_pages = size / PGSIZE;
	pte_t* pte;
	for(int i = 0;i < num_pages;i++){
		if(!(pte = (pte_t*)pgdir_walk(pgdir,(void*)va,1))){
			//it must panic from alloc_page
			panic("out of memory!!!\r\n");
		}
		*pte = pa | perm | PTE_P;
		va += PGSIZE;
		pa += PGSIZE; 
	}
}
```

这里由于注释中说到，size是PGSIZE的倍数，那么直接除以就行，得到的就是所需要的pages数

使用pgdir_walk，获取到这张页面的pte，之后，再对其中的映射进行修改就行，还是记住，映射在这里就是对pte的地址的修改。基址由KERNBASE提供

## page_lookup

该函数的作用是找到va所对应的pte，并将结果返回给pte_store，同时返回一个指向这个页面的PageInfo

```c
struct PageInfo *
page_lookup(pde_t *pgdir, void *va, pte_t **pte_store)
{
	// the pte is the PTE of the last level page
	pte_t* pte = (pte_t*)pgdir_walk(pgdir,va,0);
	if(!pte) return NULL;
	if(pte_store) *pte_store = pte;

	if(*pte & PTE_P){
		return pa2page(PTE_ADDR(*pte));
	}
	else return NULL;
}
```

通过pgdir_walk获取到最后一级页表的PTE，这时根据注释中的要求，进行一下检查。

最后一步，通过pa2page获取到对应页面的info。这里我们引入一个概念PFN（物理页框号），这个变量对应着物理页面的起始地址（或者说二者直接关联）。而在最后一级的PTE中储存的也就是这个PFN，所以，通过操作它，就可以从我们之前储存的`pages`数组中获取到对应info。

这里有一点，我没有搞懂；通过查看源代码`PTE_ADDR()`就是将pte的地址低12位清零，然后再去右移12位。但是如果直接右移难道不是也可以吗，这点我没有明白

## page_remove

首先，我们先设想一下正常情况下，我们使用这个函数的场景：我们要取消这个页面和物理页面之间的映射——也即清除pte中的值，这点我们在`pgdir_walk中有提到映射的具体含义.

不仅如此，取消虚拟页面对物理页面的映射，写到这里，这个lab的很多数据结构就更加清晰了，`pages`中的每个`info`对应了每一个物理页面，所以我们在循环中才能一个个向下去连接，同时去除部分。而info中的`pp_ref`就是有多少虚拟页面正在指向我（物理页面），而一个info对应了一个物理页面，所以这些虚拟页面是不同的，但都指向了一个相同的物理页面

```c
void
page_remove(pde_t *pgdir, void *va)
{
	pte_t* pte;
	struct PageInfo* pi = page_lookup(pgdir,va,&pte);
	if(!pi) return;
	page_decref(pi);
	*pte = 0;
	tlb_invalidate(pgdir,va);
}
```

首先就是用page_lookup获取到对应pte，然后这里使用`page_decref`来完成引用的减减操作，避免重复的造轮子。

额外说明一下最后一句：`tlb_invalidate(pgdir,va);`

这句代码完成的工作，就对应了注释中的*The TLB must be invalidated if you remove an entry from the page table.*  TLB是类似于cache一样，将常用到的页表的pte直接放在和cache一样的地方。这句代码，对应到最后是一句内联汇编	`asm volatile("invlpg (%0)" : : "r" (addr) : "memory");`，invlpg就是负责清除TLB中地址为addr的页表项。

## page_insert

```c
int
page_insert(pde_t *pgdir, struct PageInfo *pp, void *va, int perm)
{
	//first get the pte of va and store in the *pte
	pte_t* pte = pgdir_walk(pgdir,va,1);
	if(!pte){
		return -E_NO_MEM;
	}
	//must in the front of remove
	pp->pp_ref++;
	if(*pte & PTE_P){
		//here is already a page
		page_remove(pgdir, va);
	}
	*pte = page2pa(pp) | perm | PTE_P;
	return 0;
}
```

`page_insert`也即将一张物理页面`pp`映射到虚拟地址`va`上，所以先找到这个虚拟地址对应的pte，如果这个虚拟地址已经存在映射，那么就释放这个映射（不允许一个虚拟地址映射多个物理页面上），特别说明，这里的`ref`必须在判断之前进行加一操作，因为，如果不立即执行，若是进入了`page_remove`，使得ref减到0，就会导致该页面被释放。

这里拓展联系一下Linux下的”写时复制“，在父进程创建子进程的时候，如果只是对父进程的内容进行读取，那么就没有必要给子进程分配新的页面，当且仅当子进程要修改他原有的虚拟页面时（虚拟页面确实是实际不存在的，其实就是带有offset的物理页面），内核就会给他分配一个新的物理页面，并将原有的物理页面对他标记成只读（当然，之前也是只读），同时，操作系统更新子进程的页表，不再是共享页面。所以说，这时候，他们的虚拟地址就不一样了。也即，不能让一个虚拟地址指向多个物理页面。

最后将这个pte中写入物理页面所对应的地址。如此之后，就完成了我们在`pgdir_walk`中留下的问题，二级页表中的pte并没有真正映射到物理地址上，我们只通过`pgdir_walk`找到了，该存放pte的虚拟地址。

## success

到这里做完之后，进行make grade就可以得到如下的输出了

```shell
  Page management: OK 
```

# 总结

这一个part主要说明了页面管理的主要流程，并用二级页表来做了编写。

完成一个页面管理，主要三个流程——页面内存分配，多级页表的创建，页面的释放

- 对于页面内存分配，通过上一part完成的`page_alloc`可以获取到一个free page，获取到一个物理页面
- 多级页表的创建时本part的一个重点：首先通过`pgdir_walk`，创建了一级页表也即页目录到二级页表也即页索引的映射，如此之后，通过pde就可以访问到pte了。而pte到物理页面的映射完成在`page_insert`中才实现。
- 页面的释放，主要由两个过程组成，一个是基础的`page_free`，一个是物理地址映射的解除`page_remove`.二者共同完成对页面的释放

