#lab1
***
###练习1
1.OS原理和知识点
```
需要了解的内容有
1.BIOS的启动过程
2.bootloader启动过程
3.保护模式和分段模式
4.ELF文件格式
5.函数堆栈
6.中断与异常
```
2.ucore.img如何生成？
```
定义了一些需要的符号

然后定义GCCPREFIX,QEMU,USELLVM,CC,CCFLAGS,HOSTCC,HOSTCCFLAGS等符号
这是和编译器有关的参数

然后是一些INCLUDE,CFLAGS,LIBDIR这些和包含的库有关的符号
然后是kernel的一些符号：KINCLUDE,KSRDIR,KCFLAGS

然后是kernel的定义
kernel = $(call totarget,kernel)
这一句的作用是给kernel加个目录的前缀

kernel : tools/kernel.ld
定义了kernel的先决条件，这个kernel是Makefile的生成目标

接下来是这一段
$(kernel): $(KOBJS)
	@echo + ld $@
	$(V)$(LD) $(LDFLAGS) -T tools/kernel.ld -o $@ $(KOBJS)
	@$(OBJDUMP) -S $@ > $(call asmfile,kernel)
	@$(OBJDUMP) -t $@ | $(SED) '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $(call symfile,kernel)

先定义目标的先决条件是KOBJS
然后输出 + ld $@
这里$@指代的就是$(kernel)

然后
$(V)$(LD) $(LDFLAGS) -T tools/kernel.ld -o $@ $(KOBJS)
ld是连接命令，连接生成$(kernel)

然后是objdump -S 生成反汇编代码
然后是objdump -t 输出符号表

接下来是
$(bootblock): $(call toobj,$(bootfiles)) | $(call totarget,sign)
	@echo + ld $@
	$(V)$(LD) $(LDFLAGS) -N -e start -Ttext 0x7C00 $^ -o $(call toobj,bootblock)
	@$(OBJDUMP) -S $(call objfile,bootblock) > $(call asmfile,bootblock)
	@$(OBJCOPY) -S -O binary $(call objfile,bootblock) $(call outfile,bootblock)
	@$(call totarget,sign) $(call outfile,bootblock) $(bootblock)
这一点和之前大同小异，目标是生成bootblock

最后是生成ucoreimg
$(UCOREIMG): $(kernel) $(bootblock)
	$(V)dd if=/dev/zero of=$@ count=10000
	$(V)dd if=$(bootblock) of=$@ conv=notrunc
	$(V)dd if=$(kernel) of=$@ seek=1 conv=notrunc

先用0填满了10000个block，一个block有512B
把bootblock填进了一个block(512B)
把kernel填进了第二个block(512B)

```
3.一个被系统认为是符合规范的硬盘主引导扇区的特征是什么？
```
由以下部分组成
跳转指令：跳转到启动代码（平台相关）
文件卷头:文件系统描述信息
启动代码：跳转到加载指令
结束标志：0x55 0xAA
```

***
###练习2
1.从CPU加电后执行的第一条指令开始，单步跟踪BIOS的执行
```
在Makefile中加上这一段
lab1-mon: $(UCOREIMG)
	$(V)$(TERMINAL) -e "$(QEMU) -S -s -d in_asm -D $(BINDIR)/q.log -monitor stdio -hda $< -serial null"
	$(V)sleep 2
	$(V)$(TERMINAL) -e "gdb -q -x tools/lab1init"

然后lab1init的内容是
file bin/kernel
target remote:1234
set architecture i8086

然后进入执行make lab1-mon
单步使用si指令就可以单步跟踪BIOS了
```

2.在初始化位置0x7c00设置地址断点，测试断点正常。
```
执行 b *0x7c00即在 0x7c00处设了断点
然后执行到那里就可以了，测试了断点正常
```
3.从0x7c00开始跟踪代码运行，将单步跟踪反汇编得到的代码与bootasm.s和bootblock.asm对比
```
按照之前的步骤，执行到0x7c00处
执行x /10i $pc，得到：
=> 0x7c00:	cli    
   0x7c01:	cld    
   0x7c02:	xor    %eax,%eax
   0x7c04:	mov    %eax,%ds
   0x7c06:	mov    %eax,%es
   0x7c08:	mov    %eax,%ss
   0x7c0a:	in     $0x64,%al
   0x7c0c:	test   $0x2,%al
   0x7c0e:	jne    0x7c0a
   0x7c10:	mov    $0xd1,%al
和bootasm.s和bootblock.asm对比相同
```

4.自己找一个bootloader或内核中的代码位置，设置断点并进行调试。
```
利用
b cons_init 指令
在初始化console的函数进行单步跟踪
```


***
###练习3
0.首先奖段寄存器置0
1.为何开启A20,以及如何开启A20
```
在实模式下要访问高端内存区，这个开关必须打开
在保护模式下，由于使用21位地址线，如果A20恒等于0，那么系统智能奇数兆的内存了，所以在保护模式下，也必须打开

步骤：
	1.等待8042 input buffer为空
	2.向64端口输入0d1h,表示要写Output buffer
	3.等待8042 input buffer为空
	4.向60端口输入0xdfh,将A20置1
```
2.如何初始化GDT表
```
	lgdt gdtdesc #这句话将gdt载入
```
3.如何使能和进入保护模式
```
	movl %cr0, %eax
    orl $CR0_PE_ON, %eax
    movl %eax, %cr0
将CR0的PE置为1，即开启了保护模式
```

4.设置段寄存器
5.转到保护模式，调用bootmain

***

###练习4
1.bootloader如何读取硬盘扇区的？
```
读取硬盘扇区使用readseg函数：

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

首先是计算结束的虚地址
然后va对齐
然后计算所要读取的块数
然后依次调用readsect函数

然后是readsect函数的实现
static void
readsect(void *dst, uint32_t secno) {
    // wait for disk to be ready
    waitdisk();

    outb(0x1F2, 1);                         // count = 1
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
2.bootloader如何加载ELF格式的OS？
```
void
bootmain(void) {
    // read the 1st page off disk  
    readseg((uintptr_t)ELFHDR, SECTSIZE * 8, 0);
    //这一句是读取相应地elf header

    // is this a valid ELF?
    if (ELFHDR->e_magic != ELF_MAGIC) {
        goto bad;
    }
    //判断如果header不合法的话调到bad执行，输出错误信息

    struct proghdr *ph, *eph;
    //ph和eph分别表示program header 和它的结束地址

    // load each program segment (ignores ph flags)
    ph = (struct proghdr *)((uintptr_t)ELFHDR + ELFHDR->e_phoff);
    eph = ph + ELFHDR->e_phnum;
    for (; ph < eph; ph ++) {
        readseg(ph->p_va & 0xFFFFFF, ph->p_memsz, ph->p_offset);
    }
    //读取kernel

    // call the entry point from the ELF header
    // note: does not return
    ((void (*)(void))(ELFHDR->e_entry & 0xFFFFFF))();
    //调用相应的函数，即调用ucore

bad:
    outw(0x8A00, 0x8A00);
    outw(0x8A00, 0x8E00);

    /* do nothing */
    while (1);
}

```

***

###练习5
```
使用如下代码
print_stackframe(void){
	uint32_t ebp = read_ebp();
	uint32_t eip = read_eip();
	int i=0;
	for (i=0; i<STACKFRAME_DEPTH;i++){
		cprintf("ebp:0x%08x eip:0x%08x ",ebp,eip);
		cprintf("args:0x%08x 0x%08x 0x%08x 0x%08x",((uint32_t*)ebp+2)[0],((uint32_t*)ebp+2)[1],((uint32_t*)ebp+2)[2],((uint32_t*)ebp+2)[3]);
		cprintf("\n");
		print_debuginfo(eip-1);

		eip = ((uint32_t*)ebp+1)[0];
		if (ebp==0) break;
		ebp = ((uint32_t*)(ebp))[0];
	}
首先读取ebp，eip
然后逐层往上

输出ebp，eip
然后ebp+2，开始存储了参数(如果有的话)，输出
然后输出debug信息，eip是函数的返回地址，那么eip-1，就应该是落在上一条指令的范围内
而print_debuginfo(addr)这个函数是根据你给的地址，找出它的函数信息（用二分的方法查找）
然后继续递归下去，eip等于上一个函数的返回地址，ebp等于上一层的ebp

最后结果如下：
ebp:0x00007b08 eip:0x001009a7 args:0x00010094 0x00000000 0x00007b38 0x00100092
    kern/debug/kdebug.c:306: print_stackframe+22
ebp:0x00007b18 eip:0x00100ca2 args:0x00000000 0x00000000 0x00000000 0x00007b88
    kern/debug/kmonitor.c:125: mon_backtrace+10
ebp:0x00007b38 eip:0x00100092 args:0x00000000 0x00007b60 0xffff0000 0x00007b64
    kern/init/init.c:48: grade_backtrace2+33
ebp:0x00007b58 eip:0x001000bb args:0x00000000 0xffff0000 0x00007b84 0x00000029
    kern/init/init.c:53: grade_backtrace1+38
ebp:0x00007b78 eip:0x001000d9 args:0x00000000 0x00100000 0xffff0000 0x0000001d
    kern/init/init.c:58: grade_backtrace0+23
ebp:0x00007b98 eip:0x001000fe args:0x0010331c 0x00103300 0x0000130a 0x00000000
    kern/init/init.c:63: grade_backtrace+34
ebp:0x00007bc8 eip:0x00100055 args:0x00000000 0x00000000 0x00000000 0x00010094
    kern/init/init.c:28: kern_init+84
ebp:0x00007bf8 eip:0x00007d68 args:0xc031fcfa 0xc08ed88e 0x64e4d08e 0xfa7502a8
    <unknow>: -- 0x00007d67 --
ebp:0x00000000 eip:0x00007c4f args:0xf000e2c3 0xf000ff53 0xf000ff53 0xf000ff53
    <unknow>: -- 0x00007c4e --
}
```

***
###练习6
1.中断向量表一个表项占多少字节？其中哪几位代表中断处理代码入口？
```
一个表项占了8个字节。
0-1字节和6-7字节表示偏移
2-3字节表示段选择子
合起来表示中断代码入口
```

2.完善trap.c中的idt_init
```
	extern uintptr_t __vectors[];
	int i=0;
	for (i=0;i<256;i++){
		if (i==T_SYSCALL){
			SETGATE(idt[i], 1, GD_KTEXT, __vectors[i], DPL_USER);  //syscall
		}
		else
		if (i<=31 && i!=2){
			SETGATE(idt[i], 1, GD_KTEXT, __vectors[i], DPL_KERNEL);}  //trap(exception)
		else
			SETGATE(idt[i], 0, GD_KTEXT, __vectors[i], DPL_KERNEL);	  //interrupt
	}

	//load idt
	lidt(&idt_pd);

程序如上
如果是系统调用，使用用户态(陷阱门)
如果是异常，使用内核态
如果是中断，也是内核态
```

3.完善trap.c中的trap
```
ticks++;
if (ticks%TICK_NUM==0){
    print_ticks();
    ticks-=TICK_NUM;
}

每次时钟中断，增加一下计数次，每TICK_NUM次，输出ticks

```

###与参考答案的对比：
练习五中我的实现
```
print_stackframe(void){
	uint32_t ebp = read_ebp();
	uint32_t eip = read_eip();
	int i=0;
	for (i=0; i<STACKFRAME_DEPTH;i++){
		cprintf("ebp:0x%08x eip:0x%08x ",ebp,eip);
		cprintf("args:0x%08x 0x%08x 0x%08x 0x%08x",((uint32_t*)ebp+2)[0],((uint32_t*)ebp+2)[1],((uint32_t*)ebp+2)[2],((uint32_t*)ebp+2)[3]);
		cprintf("\n");
		print_debuginfo(eip-1);

		eip = ((uint32_t*)ebp+1)[0];
		if (ebp==0) break;
		ebp = ((uint32_t*)(ebp))[0];
	}

参考答案的实现：
  uint32_t ebp = read_ebp(), eip = read_eip();

    int i, j;
    for (i = 0; ebp != 0 && i < STACKFRAME_DEPTH; i ++) {
        cprintf("ebp:0x%08x eip:0x%08x args:", ebp, eip);
        uint32_t *args = (uint32_t *)ebp + 2;
        for (j = 0; j < 4; j ++) {
            cprintf("0x%08x ", args[j]);
        }
        cprintf("\n");
        print_debuginfo(eip - 1);
        eip = ((uint32_t *)ebp)[1];
        ebp = ((uint32_t *)ebp)[0];
    }

我是根据上面注释的提示做的，和参考答案大同小异
```

练习六：
```
我的实现
	extern uintptr_t __vectors[];
	int i=0;
	for (i=0;i<256;i++){
		if (i==T_SYSCALL){
			SETGATE(idt[i], 1, GD_KTEXT, __vectors[i], DPL_USER);  //syscall
		}
		else
		if (i<=31 && i!=2){
			SETGATE(idt[i], 1, GD_KTEXT, __vectors[i], DPL_KERNEL);}  //trap(exception)
		else
			SETGATE(idt[i], 0, GD_KTEXT, __vectors[i], DPL_KERNEL);	  //interrupt
	}
    
    SETGATE(idt[T_SWITCH_TOK], 0, GD_KTEXT, __vectors[T_SWITCH_TOK], DPL_USER);

	//load idt
	lidt(&idt_pd);
参考答案的实现:
    extern uintptr_t __vectors[];
    int i;
    for (i = 0; i < sizeof(idt) / sizeof(struct gatedesc); i ++) {
        SETGATE(idt[i], 0, GD_KTEXT, __vectors[i], DPL_KERNEL);
    }
	// set for switch from user to kernel
    SETGATE(idt[T_SWITCH_TOK], 0, GD_KTEXT, __vectors[T_SWITCH_TOK], DPL_USER);
	// load the IDT
    lidt(&idt_pd);

我的实现区分了异常和中断的区别，而参考答案没有
```