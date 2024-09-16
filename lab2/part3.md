# 权限和故障隔离

这里的权限主要分为用户，内核两个对象，以及可写可读两个可操作权限（当然用户可以干的内核都能干）

- ULIM：对于用户来说，ULIM上的内存没有任何权限；对于内核来说可写可读
- UTOP：UTOP下的内存留给用户使用
- [UTOP,ULIM)：用户和内核都只可以读，这里的存放一些特定的向用户公开的内核数据结构

# Exercise 5（mem_init）

```c
	// Map 'pages' read-only by the user at linear address UPAGES
	boot_map_region(kern_pgdir,UPAGES,
		ROUNDUP((sizeof(struct PageInfo)*npages), PGSIZE),PADDR(pages),PTE_U);

	// Use the physical memory that 'bootstack' refers to as the kernel
	// stack.  The kernel stack grows down from virtual address KSTACKTOP.
	// We consider the entire range from [KSTACKTOP-PTSIZE, KSTACKTOP)
	// to be the kernel stack, but break this into two pieces:
	//     * [KSTACKTOP-KSTKSIZE, KSTACKTOP) -- backed by physical memory
	boot_map_region(kern_pgdir,KSTACKTOP-KSTKSIZE,KSTKSIZE,PADDR(bootstack),PTE_W);
	
	// Map all of physical memory at KERNBASE.
	// Ie.  the VA range [KERNBASE, 2^32) should map to
	//      the PA range [0, 2^32 - KERNBASE)
	boot_map_region(kern_pgdir,KERNBASE,0XFFFFFFFF-KERNBASE,0,PTE_W);
```

对于mem_init中的最后的三个映射，总共需要三个函数；我们一个个分析

- 第一个是映射pages，作为用户可读到虚拟地址UPAGES；pages中是我们存入的所有PageInfo
- 第二个是映射内核栈。其中bootstack的获取和boot_alloc时的end获取方法一样，都是由链接器自己生成。内核栈的主要作用是放在中断处理和上下文切换中
- 最后是映射所有上部的虚拟空间，从物理地址的0处开始，按照本例中`KERNBASE=0xf0000000`，那么就是映射了256MB

其实这里还可以看到一个很有意思的东西（对于初学者的我个人来说），所有物理页面和内核栈的权限这样是合理的。但是对于顶部256MB的空间也是对用户不可见的。

经过查阅资料，其实，在每个程序运行时，所需要的虚拟地址空间都由操作系统分配，由操作系统为每个进程维护一个页表，设置哪些虚拟地址时可以读取或写入的。所以，用户程序在其虚拟程序范围内拥有读写权限，但是这些权限都受制于操作系统。

对于每个用户程序来说，地址空间都是相同的，都会由MMU进行相应的映射，同时完成各个用户程序之间的隔离。同时，这样也可以有助于动态链接库或共享库来共享代码，因为如果地址空间相同，只要加载到相同的地址就行

*以上示例，可能不适用于本lab的操作系统*

### --------over——

这个Exercise结束后，运行make grade就可以得到全部的OK了

# Question

## 2、页目录中的entries中的映射关系，它们都映射什么位置，指向何处

因为是页目录，所以一个entry对应一个二级页表，而一个二级页表可以对应4KB * 1KB 的物理页所以相邻的entry之间相关应该相差4MB

| Entry | Base Virtual Address | Points to (logically):                |
| ----- | -------------------- | ------------------------------------- |
| 1023  | ?                    | Page table for top 4MB of phys memory |
| 1022  | ?                    | ?                                     |
| .     | ?                    | ?                                     |
| .     | ?                    | ?                                     |
| .     | ?                    | ?                                     |
| 2     | 0x00800000           | ?                                     |
| 1     | 0x00400000           | ?                                     |
| 0     | 0x00000000           | [see next question]                   |

按照顺序往上以4MB累加就可以。在指向的内存上，最上端指向物理内存的最顶部。

当然按照我们刚刚做的题目，可以知道，并不是所有的内存都有映射，加上我们最开始boot阶段的(0-0x00100000)一张4MB页表包含在了最后的虚拟映射中，我们只映射了三个内存区

- 物理页面info数组：对内核和用户都只可读，用于共享页面信息
- 内核栈：只对内核可写可读，用于中断处理，任务切换等等
- 256MB物理页面映射：对内核可读可写，有内核向用户程序分配，并由MMU来控制地址转换

## 3、内核和用户程序被放在了同一个地址空间，如何进行保护

首先看看什么叫做放在了同一个地址空间——内核和用户程序共用一套虚拟地址空间，也即单地址空间

A：页表中有读写保护位，区分了不同的权限，这由MMU来管理

## 4、可以支持的最大物理内存

32位地址线，可以支持最大2^32次幂的内存，也即4GB，这里没有段映射，否则应该可以更大，类比于16位8086，使用段寄存器可以寻址到1MB的内存空间

## 5、如果我们可以管理支持的最大的物理内存，多少的内存用于管理内存

用于管理内存的也就是页表目录，页表索引，页表信息结构体三者

- 页表目录占用一个页：4KB
- 页表索引由页表目录映射：1KB个4KB也就是4MB
- 页表信息结构体：npages×(4+2),如果考虑结构体内存对齐，那么对于32位机，应该4字节一对齐。所以应该是npages×8。代码中可以得到`npages = totalmem / (PGSIZE / 1024);`而totalmem由两部分决定，一个是basemem，一个是extend的内存。通过log我们可以知道`Physical memory: 131072K available, base = 640K, extended = 130432K`

## 6、在启用分页之后，eip依然会在低地址运行一段时间。什么时候，eip会跳到高地址运行，开启分页后没有立刻跳跃到高地址运行，是否有什么影响

直接说结果——肯定是没有影响，要不然也不能这样写（bushi）。从实际情况来看，高地址的空间，其实就是低地址（物理空间）的一个映射，所以这里的eip即使工作在低地址空间也没有关系，因为，这就相当于直接操作物理页面，跳过了MMU这一层

```assembly
	# Turn on paging.
	movl	%cr0, %eax
	orl	$(CR0_PE|CR0_PG|CR0_WP), %eax
	movl	%eax, %cr0

	# Now paging is enabled, but we're still running at a low EIP
	# (why is this okay?).  Jump up above KERNBASE before entering
	# C code.
	mov	$relocated, %eax
	jmp	*%eax
relocated:
```

其实也就对应了代码中的这一段，在开启页面管理之后，后面紧接着跟了几句跳转语句（进入到c语言然后）

再来看看调试代码

```shell
0x100025:	mov    %eax,%cr0
0x00100025 in ?? ()
1: $eip = (void (*)()) 0x100025
(gdb) si
=> 0x100028:	mov    $0xf010002f,%eax
0x00100028 in ?? ()
1: $eip = (void (*)()) 0x100028
(gdb) si
=> 0x10002d:	jmp    *%eax
0x0010002d in ?? ()
1: $eip = (void (*)()) 0x10002d
(gdb) si
=> 0xf010002f <relocated>:	mov    $0x0,%ebp
relocated () at kern/entry.S:74
74		movl	$0x0,%ebp			# nuke frame pointer
1: $eip = (void (*)()) 0xf010002f <relocated>
```

所以在执行完jmp语句之后，就进入了虚拟地址空间

也就意味着，之后的所有其他初始化操作，都是在虚拟地址空间进行的，除了下面对内核栈所用的物理空间进行了保存。