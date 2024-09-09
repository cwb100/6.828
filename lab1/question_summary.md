# part1：PC Bootstrap

## QUESTIONS

### Q1：16-bit如何寻址1MB内存

<u>The first PCs, which were based on the 16-bit Intel 8088 processor, were only capable of addressing 1MB of physical memory.</u>

联系到汇编语言中的CS，SS等寄存器。对于16bit寄存器可以寻址64KB地址，所以8086将内存分为64KB的逻辑段，再通过段偏移的手段，得到最终的地址。

cpu通过DS先找到相应的逻辑段（数据段），接着根据段内指针找到响应单元；对于堆栈同理，即SS + SP获取。

对应于，汇编语言中

```assembly
MOV AX,0B800H
MOV DS,AX #内存地址0xB8000 - 0xBFFFFF
```

这样就访问到了1MB的内存

### Q2：ROM BIOS is doing what?

```assembly
0xfcf71: mov    $0x8f,%ax
0xfcf77: out    %al,$0x70
0xfcf79: in     $0x71,%al
0xfcf7b: in     $0x92,%al
0xfcf7d: or     $0x2,%al
0xfcf7f: out    %al,$0x92
```

这里应该是通过I/O控制需要初始化的芯片（或者外设）

所以BIOS（Basic I/O System），对最底层的设备进行输入输出控制

## NOTES

### 1、[f000:fff0] 0xffff0:	ljmp   $0xf000,$0xe05b

- The IBM PC starts executing at physical address 0x000ffff0, which is at the very top of the 64KB area reserved for the ROM BIOS.
- The PC starts executing with `CS = 0xf000` and `IP = 0xfff0`.
- The first instruction to be executed is a `jmp` instruction, which jumps to the segmented address `CS = 0xf000` and `IP = 0xe05b`.

在我的机器上jump 到`CS = 0x3630 IP=$0xf000e05b`

### 2、Physical Address Space

![image-20240907193928870](C:\Users\20671\Desktop\wj\微机原理\微机图片\image-20240907193928870.png)

结合上面发现，启动之后，BIOS将程序（CS:IP）跳转到距离他16个字节的位置（0xffff0)



# part2: The Boot Loader

## QUESTIONS

### Q1、At what point does the processor start executing 32-bit code? What exactly causes the switch from 16- to 32-bit mode?

```assembly
.set CR0_PE_ON,      0x1         # protected mode enable flag 设置变量

# Jump to next instruction, but in 32-bit code segment.
# Switches processor into 32-bit mode.

ljmp    $PROT_MODE_CSEG, $protcseg
.code32				    #告诉汇编器，接下来要生成32位的机器码
	--------
```

### Q2、What is the *last* instruction of the boot loader executed, and what is the *first* instruction of the kernel it just loaded?

- bootloader由两部分组成，一个是boot.s，一个是bootmain.c其中最后执行的是在c文件中的（这里可以对照反汇编代码obj/boot/boot.asm）

```c
// call the entry point from the ELF header
// note: does not return!
((void (*)(void)) (ELFHDR->e_entry))();
7d81:	ff 15 18 00 01 00    	call   *0x10018
```

​	即跳转到操作系统内核程序的起始指令处。

- 内核加载到内存中执行的第一句是

### Q3、How does the boot loader decide how many sectors it must read in order to fetch the entire kernel from disk? Where does it find this information?

首先关于操作系统一共有多少个段，每个段又有多少个扇区的信息位于操作系统文件中的Program Header Table中。这个表中的每个表项分别对应操作系统的一个段。并且每个表项的内容包括这个段的大小，段起始地址偏移等等信息。所以如果我们能够找到这个表，那么就能够通过表项所提供的信息来确定内核占用多少个扇区。
　　　那么关于这个表存放在哪里的信息，则是存放在操作系统内核映像文件的ELF头部信息中。

```c
void
readseg(uint32_t pa, uint32_t count, uint32_t offset)
```

函数从硬盘`offset`指定的位置开始，读取`count`字节的数据，到内存中`pa`位置。在`bootmain`中调用形式为`readseg((uint32_t) ELFHDR, SECTSIZE*8, 0)`，可见是从硬盘的最开头读取了8个`SECTSIZE`这么多的内容到内存中制定位置`ELFHDR`。其中，`ELFHDR`指定为`0x10000`，是内核的开头，正如反汇编文件`obj/kern/kernel.asm`的第一个指令换算前的地址正是`0x10000`。读取了8个区块，区块大小`SECTSIZE`指定为512，则总大小为`8 * 512 = 4096`，这是一个`page`的大小。

读取进来的是一个**镜像**，也就是`ELF`文件的部分内容。之所以是**部分**，是因为我们还不知道整个内核的大小，但是这里读取进来的信息至少包含了**文件头**，真正的读取还要根据文件头中包含的信息执行。

（The program header table tells the system how to create a process image. It is found at file offset e_phoff, and consists of e_phnum entries, each with size e_phentsize. The layout is slightly different in [32-bit](https://en.wikipedia.org/wiki/32-bit) ELF vs [64-bit](https://en.wikipedia.org/wiki/64-bit) ELF, because the p_flags are in a different structure location for alignment reasons. ）来自wiki。

### Q4、Exercise 5. 将原链接地址0x7c00修改后会发生什么

这里先按照lab中的词汇以个人理解区分一下两个地址——link address和load address。对于这两个地址，lab中用了ELF中的VMA(link address)和LMA来说明。所以说link address相当于一种相对地址，是虚拟内存中所用到的基地址。

对于这里的0x7c00，lab中有说明，8086通过硬布线来实现，无论如何改装载地址不会变化。所以说，修改了boot中的makefrag并不影响bootloader的装载。而修改后，此处的link address实际上是发生变化的，所以说基地址会发生变化，就导致内部在执行跳转或者符号表的调用时，就会发生问题。

从代码角度来看，会跟明白。我们先修改boot/Makefrag中的0x7c00(这里我修改为了0x7000)，之后我们再回到lab目录下

```shell
make clean
make

#如下输出就是重新编译完成了
+ as kern/entry.S
+ cc kern/entrypgdir.c
+ cc kern/init.c
+ cc kern/console.c
+ cc kern/monitor.c
+ cc kern/printf.c
+ cc kern/kdebug.c
+ cc lib/printfmt.c
+ cc lib/readline.c
+ cc lib/string.c
+ ld obj/kern/kernel
ld: warning: section `.bss' type changed to PROGBITS
+ as boot/boot.S
+ cc -Os boot/main.c
+ ld boot/boot
boot block is 412 bytes (max 510)
+ mk obj/kern/kernel.img
```

之后，我们再来看看bootloader的ELF头里面的VMA和LMA（这里的二者是一样的，原因应该是因为暂时没有操作系统的进入，所以没有虚拟内存（但是VMA仍看作是link address））（可以联想实模式和保护模式）

```shell
objdump -h obj/boot/boot.out
```

```shell
Idx Name          Size      VMA       LMA       File off  Algn
  0 .text         0000019c  00007000  00007000  00000074  2**2
```

得到了这样的结果，发现，我们的VMA确实被修改了

之后我们调试内核，可以发现bios仍然会跳转到0x7c00，原因上面我们说了（硬布线），之后真正出错，也就是因为VMA发生改动，导致无法跳转到正确的地址。

## NOTES

### 1、[protect  mode   And real mode](https://pdos.csail.mit.edu/6.828/2018/readings/pcasm-book.pdfe)

- 对于real mode，内存大小很有限即1MB（对于为什么16bit可以寻址1MB在Q1中说明，这里对DS，CS等的‘S’翻译为selector）。

  这里导致一个问题的出现，一个selector还是只能访问64K的内存，所以要求将程序分为64K的小块；当然不止是程序段CS，还包括数据段DS，在面临大数据段时，将非常awkward。

- protect mode分为16位和32位的（现在应该也有64位的）。

  在保护模式下，主要的思想是通过虚拟内存来进行内存的控制。并且可以访问到1MB以上的空间（也就是全部的硬件功能）

  对于16位保护模式的情况来说，在实模式中出现的段问题还是仍然存在。不过在32-bit的情况下，就可以得到解决，因为他的一个段可以有4GB，同时可以分的更小，进而有了4KB的页表。**可以联想到段页储存法**

  

### 2、((void (*)(void)) (ELFHDR->e_entry))();

先对e_entry进行简单说明，这是ELF中保存的程序入口地址，指示了程序从何处开始执行。虽然是入口地址，但是它并不在0x00100000执行（bootloader加载的地址），在kern/entry.s中有这样一句话

```
	# We haven't set up virtual memory yet, so we're running from
	# the physical address the boot loader loaded the kernel at: 1MB
	# (plus a few bytes).  
```

也即，入口地址在加载地址稍微偏后一点的位置

1. **`ELFHDR->e_entry`**：
   - `ELFHDR` 是一个指向 ELF（Executable and Linkable Format）头部的结构体指针。`e_entry` 是这个结构体中的一个成员，通常表示程序入口点的地址（即程序开始执行的位置）。
2. **`(void (\*)(void))`**：
   - 这是一个类型转换，将 `e_entry` 的地址转换为一个函数指针。具体来说，这里将其转换为一个返回类型为 `void` 且不接受任何参数的函数指针。
   - `void (*)(void)` 表示一个指向返回类型为 `void` 的函数的指针，且这个函数不接受任何参数。
3. **`((void (\*)(void)) (ELFHDR->e_entry))`**：
   - 这部分将 `e_entry` 中的地址转换为函数指针后，即获得了一个可以调用的函数指针。
4. **`();`**：
   - 这部分表示调用刚刚转换得到的函数指针。由于函数不接受任何参数，因此括号是空的。

 ### 3、ELF

#### header：

- .bss: 未初始化的全局变量
- .text: 程序指令
- .rodata: 只读数据，如字符串常量，const修饰的变量等
- .data: 初始化的全局变量

#### VMA，LMA

​		其中VMA为虚拟内存地址（link address），它确定程序在最终可执行文件中的位置，在程序符号表中记录。

​		LMA（load memory address），程序最终装载的位置。

​		所以说，程序最终执行的位置为LMA。可以理解为VMA为逻辑地址，而LMA是实际物理地址

# part3、The Kernel

## Exercise7. 查看映射完成前后的内存变化

![image-20240908211228795](C:\Users\20671\Desktop\wj\微机原理\微机图片\image-20240908211228795.png)

可以发现映射前，内存内容不同

![image-20240908211411202](C:\Users\20671\Desktop\wj\微机原理\微机图片\image-20240908211411202.png)

映射后，0xf0100000的内容已相同。

*What is the first instruction after the new mapping is established that would fail to work properly if the mapping weren't in place? Comment out the `movl %eax, %cr0` in `kern/entry.S`, trace into it, and see if you were right.*

这里我猜测是68行的代码`jmp *%eax`，原因和之前修改bootloader的link address0x7c00的想法一样，映射基地址发生了变化，将导致这句指令指向一个空指针。

## Exercise8. 补全打印八进制的代码

(这里推荐使用vscode，因为可以直接go to definition)我们顺着print(就是cprintf)一步步找下去发现它逐次调用了`vcprintf vprintfmt`，发现在函数内处理fmt格式化字符，向下可以找到o，也就是要求补全的代码，这里仿照%u进行修改就可以

```c
case 'o':
     num = getint(&ap, lflag);
     base = 8;
     goto number;
-----最后都会跳到230行的number处打印--------
number:
	printnum(putch, putdat, num, base, width, padc);
	break;

------最后在printnum中递归实现逆序打印--------
/*
 * Print a number (base <= 16) in reverse order,
 * using specified putch function and associated pointer putdat.
 */
static void
printnum(void (*putch)(int, void*), void *putdat,
	 unsigned long long num, unsigned base, int width, int padc)
{
	// first recursively print all preceding (more significant) digits
	if (num >= base) {
		printnum(putch, putdat, num / base, base, width - 1, padc);
	} else {
		// print any needed pad characters before first digit
		while (--width > 0)
			putch(padc, putdat);
	}

	// then print this (the least significant) digit
	putch("0123456789abcdef"[num % base], putdat);
}
```

可以重新编译调试，查看发现

```
6828 decimal is XXX octal!
```

变成

```
6828 decimal is 15254 octal!
```

说明我们结果是正确的

### Q1、printf.c与console.c之间的接口问题

printf.c中使用了cputchar()函数，而console.c中也会使用cprintf()，打印信息到终端上。

### Q2、解释下面代码

```c
      if (crt_pos >= CRT_SIZE) {
              int i;
              memmove(crt_buf, crt_buf + CRT_COLS, (CRT_SIZE - CRT_COLS) * sizeof(uint16_t));
              for (i = CRT_SIZE - CRT_COLS; i < CRT_SIZE; i++)
                      crt_buf[i] = 0x0700 | ' ';
              crt_pos -= CRT_COLS;
      }
```

这里向上翻看crt_pos保存的是光标的位置。CRT_COLS和CRT_SIZE等是宏，表示显示器一行的字长和总共最大容纳的字数

memmove是标准库函数`void *memmove(void *dest, const void *src, size_t n);`表示以src为源地址，向dest方向移动，移动单位为n个字节。在这里，也就是向首地址移动。

所以综合起来看，主要功能就是当光标指到最后一行的时候（屏幕满时），将文字向上移动一行。最后的for循环是，将新行用`0x0700 | ' '`填充(应该是颜色色块)，之后再将光标向上移动一行。完成总体的移动。

### Q3、cprintf("x=%d y=%d", 3);会得到什么

这里我们看到cprintf是没有做异常处理的，所以，要从va_arg源码的角度考虑

```c
#define va_arg(ap, type) \
    (ap += sizeof(type), *((type *)(ap - sizeof(type))))
```

从上面我们可以看到，va_arg其实就是对ap的地址进行操作，也就是将它向右加一个单位来移动ap；返回逗号表达式的右项，也就是加完之后，后退一个单位得到这个地址中的结果。

所以这里分析，对于上述的输出，主要要看在这个，3之后跟着的下一个（int*）内存中储存的是什么。

**这里我们也可以直接调试源代码得到**

```shell
---在函数i386_init处打上断点，然后调试就可以----
b i386_init
```

### Q4、如果改变GCC的调用时的压栈顺序要怎么改变cprintf()的使用

- 使用时倒着输参数
- 更改ap的增长顺序（涉及修改标准库了，可以用函数打桩）

## Exercise9、内核初始化栈相关

- 从entry.S中可以找到这样几句话

  ```assembly
  # Set the stack pointer
  	movl	$(bootstacktop),%esp
  ```

  这里将bootstacktop的地址传入esp，也即栈指针（bootstacktop可以在.data区中找到）


- 我们也可以调试查看bootstacktop或者esp的值

  ```shell
  (gdb) print $esp
  $1 = (void *) 0xf0110000 <entry_pgtable>
  ```

  从kernel.asm反汇编文件中，可以看到，和我们调试得到的结果是一样的，所以可以确定，栈顶指针初始化时，指向0xf0110000（当然是虚拟地址，物理地址减去偏移量0xf0000000）

  ```shell
  	movl	$(bootstacktop),%esp
  f0100034:	bc 00 00 11 f0       	mov    $0xf0110000,%esp
  ```

- *Everything below that location in the region reserved for the stack is free.*可以知道，栈顶向下增长

## Exercise10、调用test_trace的过程

仍然通过调试来查看

```shell
----打上断点
(gdb) b test_backtrace
Breakpoint 3 at 0xf0100040: file kern/init.c, line 13.
------向下执行
(gdb) si
=> 0xf0100044 <test_backtrace+4>:	push   %ebp
0xf0100044	13	{
(gdb) si
=> 0xf0100045 <test_backtrace+5>:	mov    %esp,%ebp
0xf0100045 in test_backtrace (x=-267386628) at kern/init.c:13
13	{
(gdb) print $ebp
$4 = (void *) 0xf010fff8
(gdb) print $esp
$5 = (void *) 0xf010ffd8

```

可以看到，这里堆栈指针和ebp的值

我们也可以使用info stack来查看调用关系。

```
(gdb) info stack
#0  0xf0100045 in test_backtrace (x=-267386628) at kern/init.c:13
#1  0xf010fff8 in bootstack ()
#2  0xf01000fc in i386_init () at kern/init.c:39
#3  0x00000005 in ?? ()
#4  0xf010003e in relocated () at kern/entry.S:80
```

查看esp向下的50个空间（下面只选取重要的部分）

可以看到，在0xf010fff8处存有test_backtrace(5)的返回地址0xf010003e

```shell
(gdb) x/50x $esp
0xf010ffd8:	0xf010fff8	0xf01000fc	0x00000005	0x00001aac
0xf010ffe8:	0x00000640	0x00000000	0x00000000	0x00010094
0xf010fff8:	0x00000000	0xf010003e	0x00000003	0x00001003
```




# summary
## 启动过程

