### OS Lab 1 Report
#### 计55  陈齐斌  2015011403

### 练习 1
#### 1.1
- 在如下代码中可以看到，UCOREIMG的make首先需要kernel以及bootblock，然后进行了

```makefile
# create ucore.img
UCOREIMG    := $(call totarget,ucore.img)

$(UCOREIMG): $(kernel) $(bootblock)
                   $(V)dd if=/dev/zero of=$@ count=10000
                   $(V)dd if=$(bootblock) of=$@ conv=notrunc
                   $(V)dd if=$(kernel) of=$@ seek=1 conv=notrunc
```

- 而make kernel的代码段为如下，可以看出首先需要 tools/kernel.ld 以及 KOBJS

```makefile
$(kernel): tools/kernel.ld

$(kernel): $(KOBJS)
    @echo + ld $@
    $(V)$(LD) $(LDFLAGS) -T tools/kernel.ld -o $@ $(KOBJS)
    @$(OBJDUMP) -S $@ > $(call asmfile,kernel)
    @$(OBJDUMP) -t $@ | $(SED) '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $(call symfile,kernel)
```

- 而bootblock代码段如下，可见首先需要bootfiles以及sign.

```makefile
# create bootblock
bootfiles = $(call listf_cc,boot)
$(foreach f,$(bootfiles),$(call cc_compile,$(f),$(CC),$(CFLAGS) -Os -nostdinc))

bootblock = $(call totarget,bootblock)

$(bootblock): $(call toobj,$(bootfiles)) | $(call totarget,sign)
    @echo + ld $@
    $(V)$(LD) $(LDFLAGS) -N -e start -Ttext 0x7C00 $^ -o $(call toobj,bootblock)
    @$(OBJDUMP) -S $(call objfile,bootblock) > $(call asmfile,bootblock)
    @$(OBJCOPY) -S -O binary $(call objfile,bootblock) $(call outfile,bootblock)
    @$(call totarget,sign) $(call outfile,bootblock) $(bootblock)
```

- 之后的三句命令以及含义分别为

```makefile
# 生成10000个全零块的文件
$(V)dd if=/dev/zero of=$@ count=10000
# 将bootblock写到该文件的开头，notrunc意思是不要输出不要被文件的大小所截断
$(V)dd if=$(bootblock) of=$@ conv=notrunc
# seek=1 表示从第二个块开始写，写的内容(if)为kernel
$(V)dd if=$(kernel) of=$@ seek=1 conv=notrunc
```

#### 1.2
- 根据提示，查看`sign.c`中第31, 32行， 了解到特征是最后两个字节分别为0x55, 0xAA

### 练习 2

#### 2.1 从CPU加电后执行的第一条指令开始，单步跟踪BIOS的执行。

1. 首先修改 lab1/tools/gdbinit，为:
```
set architecture i8086
target remote :1234
```

2 在 lab1目录下，执行make操作
```
# 清理之前生成的完整文件
make clean
make debug
```

3 在 gdb 中输入si
```
si
```
即可单步跟踪BIOS了。
![](http://otukr87eg.bkt.clouddn.com/5fe929be93341f13668918ca3879a016.jpg)


#### 2.2 在初始化位置0x7c00 设置实地址断点，测试断点正常。

1. 在tools/gdbinit结尾加上

```
set architecture i8086  //设置当前调试的CPU是8086
b *0x7c00  //在0x7c00处设置断点。此地址是bootloader入口点地址，可看boot/bootasm.S的start地址处
c          //continue简称，表示继续执行
x /2i $pc  //显示当前eip处的汇编指令
set architecture i386  //设置当前调试的CPU是80386
```

2. 同样执行make clean，make debug，进入gdb可以看到

```
Breakpoint 2, 0x00007c00 in ?? ()
=> 0x7c00:      cli
   0x7c01:      cld
   0x7c02:      xor    %eax,%eax
   0x7c04:      mov    %eax,%ds
   0x7c06:      mov    %eax,%es
   0x7c08:      mov    %eax,%ss
   0x7c0a:      in     $0x64,%al
   0x7c0c:      test   $0x2,%al
   0x7c0e:      jne    0x7c0a
   0x7c10:      mov    $0xd1,%al
```

### 练习 3
#### 3.1 请分析bootloader是如何完成从实模式进入保护模式的。

- 进入`bootasm.S`前有 `%cs=0 %ip=7c00`
```asm
  # 这一步的作用为将flag以及段寄存器置零
  .code16
      cli
      cld
      xorw %ax, %ax
      movw %ax, %ds
      movw %ax, %es
      movw %ax, %ss
```
- 以下部分为将A20使能。为了后向兼容，在实模式下只能访问1M内存空间，这一步使得4G地址都能访问到。
```asm
    #  Enable A20:
    #  For backwards compatibility with the earliest PCs, physical
    #  address line 20 is tied low, so that addresses higher than
    #  1MB wrap around to zero by default. This code undoes this.
    seta20.1:
    inb $0x64, %al                                  # Wait for not busy(8042 input buffer empty).
    testb $0x2, %al
    jnz seta20.1

    movb $0xd1, %al                                 # 0xd1 -> port 0x64
    outb %al, $0x64                                 # 0xd1 means: write data to 8042's P2 port

    seta20.2:
    inb $0x64, %al                                  # Wait for not busy(8042 input buffer empty).
    testb $0x2, %al
    jnz seta20.2

    movb $0xdf, %al                                 # 0xdf -> port 0x60
    outb %al, $0x60                                 # 0xdf = 11011111, means set P2's A20 bit(the 1 bit) to 1
```

- 之后初始化gdt表并通过将cr0寄存器pe位置1，这样就开启了保护模式
```asm
    # Switch from real to protected mode, using a bootstrap GDT
    # and segment translation that makes virtual addresses
    # identical to physical addresses, so that the
    # effective memory map does not change during the switch.
    lgdt gdtdesc
    movl %cr0, %eax
    orl $CR0_PE_ON, %eax
    movl %eax, %cr0
```

- 以下语句通过长跳转更新了cs的基地址，并且设置段寄存器，建立栈
```asm
        # Jump to next instruction, but in 32-bit code segment.
        # Switches processor into 32-bit mode.
        ljmp $PROT_MODE_CSEG, $protcseg

    .code32                                             # Assemble for 32-bit mode
    protcseg:
        # Set up the protected-mode data segment registers
        movw $PROT_MODE_DSEG, %ax                       # Our data segment selector
        movw %ax, %ds                                   # -> DS: Data Segment
        movw %ax, %es                                   # -> ES: Extra Segment
        movw %ax, %fs                                   # -> FS
        movw %ax, %gs                                   # -> GS
        movw %ax, %ss                                   # -> SS: Stack Segment

        # Set up the stack pointer and call into C. The stack region is from 0--start(0x7c00)
        movl $0x0, %ebp
        movl $start, %esp
```
- 此时已经成功向保护模式转换，可以进入主方法，即 `call bootmain`。

### 练习 4
#### 4.1 通过阅读bootmain.c，了解bootloader如何加载ELF文件。通过分析源代码和通过qemu来运行并调试
- readsect 为最底层的读磁盘操作，函数的两个参数含义为，secno为扇区编号，dst为存放读出数据的内存地址。
```c
    /* readsect - read a single sector at @secno into @dst */
    static void
    readsect(void *dst, uint32_t secno) {
        // wait for disk to be ready
        waitdisk();

        outb(0x1F2, 1);                         // count = 1
        // 以下四条指令一起指定扇区编号
        outb(0x1F3, secno & 0xFF);
        outb(0x1F4, (secno >> 8) & 0xFF);
        outb(0x1F5, (secno >> 16) & 0xFF);
        outb(0x1F6, ((secno >> 24) & 0xF) | 0xE0);
        outb(0x1F7, 0x20);                      // cmd 0x20 - read sectors

        // wait for disk to be ready
        waitdisk();

        // read a sector
        insl(0x1F0, dst, SECTSIZE / 4);
    }
```

- readseg 对 readsect 进行了封装，用来读取指定长度的数据。
```c
    /* *
     * readseg - read @count bytes at @offset from kernel into virtual address @va,
     * might copy more than asked.
     * */
    static void
    readseg(uintptr_t va, uint32_t count, uint32_t offset) {
        uintptr_t end_va = va + count;

        // round down to sector boundary
        va -= offset % SECTSIZE;

        // translate from bytes to sectors; kernel starts at sector 1
        uint32_t secno = (offset / SECTSIZE) + 1;

        // If this is too slow, we could read lots of sectors at a time.
        // We'd write more to memory than asked, but it doesn't matter --
        // we load in increasing order.
        for (; va < end_va; va += SECTSIZE, secno ++) {
            readsect((void *)va, secno);
        }
    }
```

- `bootmain` 函数中包含了加载ELF格式的os，使用了上面的读磁盘的函数。具体见代码中的中文注释
```c
    /* bootmain - the entry of bootloader */
    void
    bootmain(void) {
        // read the 1st page off disk，即读取ELF头
        readseg((uintptr_t)ELFHDR, SECTSIZE * 8, 0);

        // is this a valid ELF?
        if (ELFHDR->e_magic != ELF_MAGIC) {
            goto bad;
        }

        struct proghdr *ph, *eph;

        // load each program segment (ignores ph flags)
        ph = (struct proghdr *)((uintptr_t)ELFHDR + ELFHDR->e_phoff);
        eph = ph + ELFHDR->e_phnum;

        // 这一步为加载ELF文件的数据到内存，依据ELF头部中读到的描述表。
        for (; ph < eph; ph ++) {
            readseg(ph->p_va & 0xFFFFFF, ph->p_memsz, ph->p_offset);
        }

        // call the entry point from the ELF header
        // note: does not return
        ((void (*)(void))(ELFHDR->e_entry & 0xFFFFFF))();
        // 最后根据ELF头部存储的入口信息，找到内核入口。
    bad:
        outw(0x8A00, 0x8A00);
        outw(0x8A00, 0x8E00);

        /* do nothing */
        while (1);
    }
```

### 练习 5

- `print_stackframe` 代码实现如下

```c
void print_stackframe(void) {
    uint32_t ebp = read_ebp(), eip = read_eip();
    for (int i = 0; i < STACKFRAME_DEPTH && ebp != 0; ++i)
    {
        cprintf("ebp:0x%08x eip:0x%08x ", ebp, eip);
        cprintf("args:");
        uint32_t *args = (uint32_t *)ebp + 2;
        for (int j = 0; j < 4; ++j)
        {
            cprintf("0x%08x ", args[j]);
        }
        cprintf("\n");
        print_debuginfo(eip - 1);
        eip = ((uint32_t *)ebp)[1];
        ebp = ((uint32_t *)ebp)[0];
    }
}
```

- 为防止意外出现死循环，使用循环深度用i迭代而不是直接while (ebp != 0)，进行ebp的回溯，按照格式输出信息后，调用`print_debuginfo`函数输出一些函数名的信息，直到最后到达栈顶判断退出。
- 为打印地址，使用了%08x的格式化输出，输出补零的8位16进制。

### 练习 6 完善中断初始化和处理

#### 6.1
- 中断描述符表（也可简称为保护模式下的中断向量表）中一个表项占8个字节。其中哪几位代表中断处理代码的入口？
    - 表中一个表项占用8字节，其中2-3字节是段选择子，因此处理代码入口为0-1字节和6-7字节两者concat。

#### 6.2 请编程完善kern/trap/trap.c中对中断向量表进行初始化的函数idt_init。在idt_init函数中，依次对所有中断入口进行初始化。使用mmu.h中的SETGATE宏，填充idt数组内容。每个中断的入口由tools/vectors.c生成，使用trap.c中声明的vectors数组即可。
- 首先声明外部定义的符号 `extern uintptr_t __vectors[];` 得到中断服务程序的入口
- 之后迭代idt中的256项，依次调用SETGATE宏，并将T_SWITCH_TOK处设为DPL_USER，代码如下
```c
    for (int i = 0; i < 256; i++) {
        SETGATE(idt[i], 0, GD_KTEXT, __vectors[i], DPL_KERNEL);
    }
    SETGATE(idt[T_SWITCH_TOK], 0, GD_KTEXT, __vectors[T_SWITCH_TOK], DPL_USER);
```
 - 最后根据提示需要让cpu知道中断描述表的位置，使用指令lidt，语句为 `lidt(&idt_pd);`

#### 6.3 请编程完善trap.c中的中断处理函数trap，在对时钟中断进行处理的部分填写trap函数中处理时钟中断的部分，使操作系统每遇到100次时钟中断后，调用print_ticks子程序，向屏幕上打印一行文字”100 ticks”。
- 这一步比较简单，使用 `kern/driver/clock.c` 中定义的 ticks 变量，每一次处理中断就把它+1，每 TICK_NUM 调用一下现有的函数 `print_ticks()`

### 与参考答案的区别
- 实验中与参考答案的区别主要在第1、2两个练习，由于之前只会使用最简单的Makefile用法，做起来比较吃力。实验2的gdb中查看汇编等操作同。
- 练习6.1中，初始化中断向量表是，参考答案中的循环上界使用 `sizeof(idt) / sizeof(struct gatedesc)` 计算得到，而我直接用了idt数组的元素个数256。由于for循环每次循环都需要判断是否达到上界，因此认为应直接用256。不过由于是静态数组可能编译器直接会加优化，这样两种情况就一样了。
