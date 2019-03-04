# uCore Lab1 实验报告

## 练习1：理解通过make生成执行文件的过程

执行 `make "V=" > make_commands.sh`，能够得到 `make` 执行的所有命令历史。观察可以得到，执行的过程分为以下的阶段。

### 编译内核

命令类似于：
bash
```
gcc -Ikern/mm/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Ikern/debug/ -Ikern/driver/ -Ikern/trap/ -Ikern/mm/ -c kern/mm/pmm.c -o obj/kern/mm/pmm.o
```

这些命令调用 `gcc` 编译内核的各个 C 源码为 `.o` 目标文件，生成在 `obj/kern` 目录的各个子目录下。同时也编译了两个支持库（字符串、格式化）生成在 `obj/lib/` 下。

编译器选项 `-fno-builtin` 关闭 GCC 内置的函数，`-Wall` 打开所有警告，`-ggdb, -gstabs` 生成 GDB 格式的调试信息，`-m32` 生成 x86 代码，`-nostdinc` 不使用 `libstdc` 的头文件目录，`-fno-stack-protector` 关闭栈溢出保护代码，`-I` 指定头文件搜索目录，`-c` 指定编译为目标文件，`-o` 指定输出文件名。  

### 链接内核

在生成了所有目标文件后，使用 `ld` 进行链接（中间省略了大量文件名）：

```bash
ld -m    elf_i386 -nostdlib -T tools/kernel.ld -o bin/kernel  obj/kern/init/init.o (...) obj/libs/printfmt.o
```

链接器选项 `-m` 指定目标平台和文件格式，`-nostdlib` 指定不链接标准库，`-T` 指定链接脚本，`-o` 指定输出文件，之后的参数都是输入文件。最终得到可执行的内核文件 `bin/kernel`。

### 编译与链接 Bootloader

```bash
gcc -Iboot/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Os -nostdinc -c boot/bootasm.S -o obj/boot/bootasm.o
gcc -Iboot/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Os -nostdinc -c boot/bootmain.c -o obj/boot/bootmain.o
gcc -Itools/ -g -Wall -O2 -c tools/sign.c -o obj/sign/tools/sign.o
ld -m    elf_i386 -nostdlib -N -e start -Ttext 0x7C00 obj/boot/bootasm.o obj/boot/bootmain.o -o obj/bootblock.o
```

与上述的类似，这些命令生成了 `obj/bootblock.o` 为 bootloader 映像文件。其中链接器参数 `-N` 表示 `text` 与 `data` 段都可读写，并在文件中加入 `OMAGIC` 标记；`-e` 指定入口点， `-Ttext` 指定 `text` 段起始地址。

### 生成磁盘镜像

首先将 `bootloader.o` 中的 `text` 段复制出来：

```bash
objcopy -S -O binary obj/bootblock.o obj/bootblock.out
```

而后编译生成（本机运行的）磁盘扇区镜像生成工具 `sign`，并用它生成带签名的引导扇区镜像。

```bash
gcc -Itools/ -g -Wall -O2 -c tools/sign.c -o obj/sign/tools/sign.o
gcc -g -Wall -O2 obj/sign/tools/sign.o -o bin/sign
bin/sign obj/bootblock.out obj/bootblock
```

而后创建磁盘镜像，而后将其引导扇区和内核写入其中：

```bash
dd if=/dev/zero of=bin/ucore.img count=10000
dd if=bin/bootblock of=bin/ucore.img conv=notrunc
dd if=bin/kernel of=bin/ucore.img seek=1 conv=notrunc
```

注意到 `dd` 的计数默认为扇区（512字节），因此磁盘镜像共有 10000 个扇区，第0扇区为 bootloader，从第1扇区起是内核镜像。

通过对 `sign` 工具的源码分析，可知硬盘主引导扇区的大小一定需要为 512 字节，并且最后两个字节为 ``0x55 0xAA`。

## 练习2：使用qemu执行并调试lab1中的软件

由于我的系统没有 GUI，因此不能直接使用 `Makefile` 中的指令。经过测试，在运行之前需要设置下列环境变量。

```bash
export QEMU="qemu-system-i386 -nographic -monitor none"
```

此时使用 `make qemu`， QEMU 能够正常等待 gdb 的连接。我不使用 `make debug`，因为其中硬编码了过多系统相关的命令（如 `gnome-terminal`）。

### 单步跟踪BIOS的执行

在 GDB 中执行：

```gdb
file bin/kernel
target remote 127.0.0.1:1234
```

执行 `info reg` 可以看到目前的寄存器情况，`CS=0xf000, EIP=0xfff0`，即当前地址为 `0xfffffff0`。反编译当前命令：

```gdb
gdb-peda$ x/i 0xfffffff0
   0xfffffff0:  jmp    0x3630:0xf000e05b
gdb-peda$ si
Warning: not running or target is remote
0x0000e05b in ?? ()
gdb-peda$ si
Warning: not running or target is remote
0x0000e062 in ?? ()
```

此即为单步执行。使用 `x/i` 也可以跟踪当前的指令。

### 设置实地址断点并反编译：

```gdb
gdb-peda$ b *0x7c00
Breakpoint 1 at 0x7c00
gdb-peda$ c
Continuing.
Warning: not running or target is remote

Breakpoint 1, 0x00007c00 in ?? ()
gdb-peda$ x/10i 0x7c00
=> 0x7c00:      cli
   0x7c01:      cld
   0x7c02:      xor    eax,eax
   0x7c04:      mov    ds,eax
   0x7c06:      mov    es,eax
   0x7c08:      mov    ss,eax
   0x7c0a:      in     al,0x64
   0x7c0c:      test   al,0x2
   0x7c0e:      jne    0x7c0a
   0x7c10:      mov    al,0xd1

```

可以看到成功设置了断点并中断了执行。反汇编可以得到上面的代码，与 `bootasm.S` 第16行开始的代码一致，与 `bootblock.asm` 第14行开始的代码也是一致的。可以看到这些代码主要工作为关闭中断、清空段寄存器、打开A20地址线等，将在下面分析。

我们在打印内核信息的 `print_kerninfo` 函数前设置断点并观察：

```gdb
gdb-peda$ break print_kerninfo
Breakpoint 2 at 0x100a32: file kern/debug/kdebug.c, line 218.
gdb-peda$ c
Continuing.
Warning: not running or target is remote

Breakpoint 2, print_kerninfo () at kern/debug/kdebug.c:218
218     print_kerninfo(void) {
gdb-peda$ disas print_kerninfo
Dump of assembler code for function print_kerninfo:
=> 0x00100a32 <+0>:     push   ebp
   0x00100a33 <+1>:     mov    ebp,esp
   0x00100a35 <+3>:     push   ebx
   0x00100a36 <+4>:     sub    esp,0x4
......
```

而查看当前的 `CS=0x8, EIP=0x100a32`，这与函数入口地址是一致的，说明断点成功被触发。

## 练习3：分析bootloader进入保护模式的过程

从 `bootasm.S` 中可以找到进行这一过程的代码，有以下的阶段：

### 初始化 (L16-L23)

首先进行上电后的一些列初始化工作，使用 `cli` 禁止中断，`cld` 清空方向标志位，并清空 `ax, ds, es, ss` 段寄存器。

### 开启 A20 地址线 (L29-L43)

为了打开 A20 地址线以使用完整的 4GB 地址空间，需要向 8042 控制器发出命令。方法为：

1. 读取 8042 状态，直到其不再忙
2. 向其控制端口写入 `0xd1`，并等待其不再忙
3. 向其数据端口写入 `0xdf`

上述命令的具体含义可见文档。

### 配置 GDT (L49, L78-L86)

GDT 表本身起始地址为 `.gdt`，其中配置了数据和代码两个段，分别具有 RX 和 W 权限，起始地址都是 0（即直接映射到物理地址空间）。

GDT 属性（基址、长度）起始地址为 `.gdtdesc`，使用 `lgdt gdtdesc` 加载 GDT。

### 开启保护模式并执行32位代码 (L50-L56)

首先将 `CR0` 寄存器中 `PE` 位置为 1，后进行长跳转到 GDT 第一个表项（代码段）的 `protcseg` 偏移，事实上就是 `protcseg` 的地址处，开始执行 32 位代码。

### 设置段寄存器

进入保护模式后，将各个数据段寄存器置为数据段对应的段选择器（在此处为 GDT 第二个表项）。而后初始化栈寄存器，即可跳转到 C 代码。

## 练习4：分析bootloader加载ELF格式的OS的过程

### Bootloader 读取硬盘扇区

在保护模式下，不可使用 BIOS 提供的中断读取硬盘。因此，`bootmain.c` 中实现了 `readsect` 函数，直接与 IDE 控制器通信，进行扇区的读取。具体的流程为：

1. 等待磁盘空闲（从 `0x1F7` 端口读取一位）
2. 向磁盘控制器 `0x1F2` 端口写入读取数量（1）
3. 向磁盘控制器 `0x1F3` 开始的4个端口写入扇区编号（LBA编号）和主从属性
4. 向磁盘控制器 `0x1F7` 端口发送读取指令 `0x20`
5. 等待磁盘空闲，后从 `0x1F0` 端口读取长度为扇区大小的数据到指定地址

### Bootloader 加载 ELF 文件过程

ELF 文件头格式为：

```c
typedef struct {
	uint32_t 	e_ident[4];     /* Magic number and other info */
	uint16_t 	e_type;			/* Object file type */
	uint16_t 	e_machine;		/* Architecture */
	uint32_t 	e_version;		/* Object file version */
	uint32_t 	e_entry;		/* Entry point virtual address */
	uint32_t 	e_phoff;		/* Program header table file offset */
	uint32_t 	e_shoff; 		/* Section header table file offset */
	uint32_t 	e_flags;		/* Processor-specific flags */
	uint16_t 	e_ehsize;		/* ELF header size in bytes */
	uint16_t 	e_phentsize;	/* Program header table entry size */
	uint16_t 	e_phnum;		/* Program header table entry count */
	uint16_t 	e_shentsize;	/* Section header table entry size */
	uint16_t 	e_shnum;		/* Section header table entry count */
	uint16_t 	e_shstrndx;		/* Section header string table index */
} elf32_ehdr;
```

结构体中标识了程序节的数量（`e_phnum`）、单个大小（`e_phentsize`）和偏移量（`e_phoff`）。类似地，每个程序节都包含该节在文件中的偏移、长度以及加载位置，据此读取每一个节到指定的地址，并跳转到指定的入口地址（`e_entry`）即可。

## 练习5：实现函数调用堆栈跟踪函数

这个练习要求完成 `kern/debug/kdebug.c` 中的 `print_stackframe` 函数，以实现当前调用栈帧打印的功能。

根据文件中的注释，我们可以知道，首先需要通过 `get_eip` 和 `get_ebp` 函数来获取当前的这两个寄存器值。而后对于每一层栈帧，都要打印以下内容：

* EIP 和 EBP 的值
* 当前函数的调用参数 （始于 EBP 后第二个字，共四个字）
* 当前函数名称和代码行号（通过 `print_debuginfo` 函数搜索 ELF 文件结构实现）
  
而后，需要向回滚上一个栈帧后重复上述流程。新的 EIP 为当前 EBP 后第一个字，即函数返回地址；新的 EBP 即为当前 EBP 指向的字。需要注意的是，当 EBP 减少到 0 时，应该停止打印。

得到的最后一行有意义信息为：

```text
ebp:0x00007bf8 eip:0x00007d6e args:0xc031fcfa 0xc08ed88e 0x64e4d08e 0xfa7502a8
    <unknow>: -- 0x00007d6d --
```

这是最早的栈帧，猜想其为 `bootmain` 函数，而根据反汇编的结果，可以看到：

```objdump
bootmain(void) {
......
    // call the entry point from the ELF header
    // note: does not return
    ((void (*)(void))(ELFHDR->e_entry & 0xFFFFFF))();
    7d62:       a1 18 00 01 00          mov    0x10018,%eax
    7d67:       25 ff ff ff 00          and    $0xffffff,%eax
    7d6c:       ff d0                   call   *%eax
}

static inline void
outw(uint16_t port, uint16_t data) {
    asm volatile ("outw %0, %1" :: "a" (data), "d" (port));
    7d6e:       ba 00 8a ff ff          mov    $0xffff8a00,%edx
......
```

`EIP` 指向的的确是 `bootmain` 最后跳转语句后的第一条语句，而 `EBP` 是它的栈基址。后面四个数值本应该是其参数，但是在没有传送参数的情况下，就是 `EBP` 后第二个字开始的四个字的内容，也就是 `0x7c00` 开始的四个字，恰好对应下面的汇编代码。

```
7c00:       fa                      cli
7c01:       fc                      cld
7c02:       31 c0                   xor    %eax,%eax
7c04:       8e d8                   mov    %eax,%ds
7c06:       8e c0                   mov    %eax,%es
7c08:       8e d0                   mov    %eax,%ss
7c0a:       e4 64                   in     $0x64,%al
7c0c:       a8 02                   test   $0x2,%al
7c0e:       75 fa                   jne    7c0a <seta20.1>
```

## 练习6：完善中断初始化和处理

中断描述符表（IDT）每个表项大小为8字节。中断服务程序入口的段选择子在31-16位，段偏移量的第15-0位在15-0位，第31-16位在63-48位。

为了填充 IDT 表，首先需要声明一个外部符号 `extern uintptr_t __vectors[]` 存放中断服务程序的地址，而后循环调用 `SETGATE` 宏设置 IDT 表的每一项（共256项）。其中偏移为 `T_SYSCALL` 的项目的 DPL 需要设置为 3（用户级），并且设置为陷阱门；其他项目的 DPL 均为 0 （系统级），并且设置为中断门。所有中断服务程序的段选择子都设置为 `KERNEL_CS`。最后调用提供的 `lidt` 函数向处理器载入 IDT 表。

在 `kern/trap/trap.c` 中，`trap_dispatch` 函数用于统一的中断处理。在 `IRQ_TIMER`，即定时中断的处理中，每次累加一个计时器 `tick`，当是 `TICK_NUM` 整数倍时，调用 `print_tick` 函数，即可定时输出节拍。

## 扩展练习 Challenge 1

本扩展联系要求实现切换当前特权级的两个系统调用：`T_SWITCH_TOU` 和 `T_SWITCH_TOK`，关键在于要求切换完成后，发起调用的函数还能正常执行，这就要求我们在中断处理程序中对 trapframe 做一些处理。

### 内核态至用户态

由于发生中断时没有发生特权级别的切换，所以 trapframe 被直接放置在当前的栈上，并且 `tf` 中的 `tf_esp` 所在的位置实际上是调用者的 `ESP` 原本指向的位置，`tf_ss` 所在的位置实际上是调用者的返回地址的一部分。而由于 `iret` 导致特权级别切换，如果直接使用提供的 trapframe，会导致调用者状态被破坏。因此，我在处理函数的栈上（也就是当前栈的最下方）将提供的 trapframe 复制了一份，修改相应的特权级别（包括段选择子和 IOPL），并将 `tf_esp` 指向调用者 `ESP` 的位置（`tf + offsetof(tf_esp)`）。最后，将存储的异常栈位置指向这个临时栈，就完成了切换。

### 用户态至内核态

此时发生的中断导致了特权级别的切换，因此 trapframe 结构体被完整放置在内核栈 `stack0` 上。由于中断返回不改变特权态，如果不进行任何处理，会导致 `ESP` 相比原来向下移动两个字，从而发生栈不平衡，导致调用者无法正常返回。因此，我选择直接在用户栈（也就是 `tf_esp` 指向的位置）上放置一个当前 `tf` 的拷贝，但是拷贝时向上移动两个字，并不覆盖原有的内容（类似于上面到内核态的初始情况）。而后，修改特权级别，并类似地重新指向临时栈地位置即可。

### 其余处理

两个切换函数的实现只需要使用 `int` 指令触发相应的中断。同时，在 IDT 的初始化中，需要将 `T_SWITCH_TOK` 的特权级设置为 `DPL_USER`，以允许用户态程序调用触发这个中断。

## 扩展练习 Challenge 2

本扩展练习要求在按下键盘的按键时切换到相应的特权级。由于上面已经实现了对应的中断处理，代码完全可以复用。只需要键盘中断的处理例程中，在检测到对应的按键时跳转到相应的代码标号处即可。