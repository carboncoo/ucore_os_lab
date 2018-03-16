# Lab1 Report

## 练习1

- [练习1.1] 操作系统镜像文件 ucore.img 是如何一步一步生成的?(需要比较详细地解释 Makefile 中每一条相关命令和命令参数的含义,以及说明命令导致的结果)

答：Makefile中生成ucore.img的代码段如下

```makefile
# create ucore.img 
UCOREIMG    := $(call totarget,ucore.img)

$(UCOREIMG): $(kernel) $(bootblock)
    $(V)dd if=/dev/zero of=$@ count=10000
    $(V)dd if=$(bootblock) of=$@ cvonv=notrunc
    $(V)dd if=$(kernel) of=$@ seek=1 conv=notrunc

$(call create_target,ucore.img)
```

第一句表示在bin目录下生成ucore.img文件

可以看到，ucore需要bootblock和kernel两个部分

---

生成bootblock的代码段如下

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

$(call create_target,bootblock)
```

在这一段代码中，首先使用宏批量生成了bootasm.o,bootmain.o的相关makefile代码，实际执行的编译代码为

```bash
gcc -Iboot/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Os -nostdinc -c boot/bootasm.S -o obj/boot/bootasm.o
+ cc boot/bootmain.c
gcc -Iboot/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Os -nostdinc -c boot/bootmain.c -o obj/boot/bootmain.o

```

其中使用的相关命令参数有

-I 添加搜索头文件的路径

-fno-builtin  除非用__builtin_前缀，否则不进行builtin函数的优化

-Wall 是打开警告开关

-ggdb  生成可供gdb使用的调试信息。

-m32  生成适用于32位环境的代码。

-gstabs  生成stabs格式的调试信息。

-nostdinc  不使用标准库。

-fno-stack-protector  不生成用于检测缓冲区溢出的代码。

-Os  为减小代码大小而进行优化。

生成sign的相关makefile代码为

```makefile
# create 'sign' tools
$(call add_files_host,tools/sign.c,sign,sign)
$(call create_target_host,sign,sign)
```

实际执行的编译代码为

```shell
gcc -Itools/ -g -Wall -O2 -c tools/sign.c -o obj/sign/tools/sign.o
gcc -g -Wall -O2 obj/sign/tools/sign.o -o bin/sign
```

使用下面的命令连接生成bootblock.o

```shell
ld -m    elf_i386 -nostdlib -N -e start -Ttext 0x7C00 obj/boot/bootasm.o obj/boot/bootmain.o -o obj/bootblock.o

```

其中使用的相关命令参数有

-m    elf_i386 模拟为i386上的连接器

-N  设置代码段和数据段均可读写

-e <entry>  指定入口

-Ttext  制定代码段开始位置

使用下面的命令将bootblock.o拷贝到bootblock.out

```shell
objcopy -S -O binary obj/bootblock.o obj/bootblock.out
```

之后使用sign工具处理bootblock.out，生成bootblock
```shell
bin/sign obj/bootblock.out bin/bootblock
```

---

生成kernel的代码段如下

```makefile
# create kernel target
kernel = $(call totarget,kernel)

$(kernel): tools/kernel.ld

$(kernel): $(KOBJS)
    @echo + ld $@
    $(V)$(LD) $(LDFLAGS) -T tools/kernel.ld -o $@ $(KOBJS)
    @$(OBJDUMP) -S $@ > $(call asmfile,kernel)
    @$(OBJDUMP) -t $@ | $(SED) '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $(call symfile,kernel)

$(call create_target,kernel)
```

kernel需要kernel.ld以及init.o readline.o stdio.o kdebug.o kmonitor.o panic.o clock.o console.o intr.o picirq.o trap.o trapentry.o vectors.o pmm.o  printfmt.o string.o等一系列.o文件

使用下面的命令可以生成上面提到的一系列.o文件

```makefile
$(call add_files_cc,$(call listf_cc,$(KSRCDIR)),kernel,$(KCFLAGS))
```

具体的编译命令基本上一致，如下举例所示

```bash
gcc -Ilibs/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/  -c libs/string.c -o obj/libs/string.o
```

之后，使用以下命令连接生成kernel

```bash
ld -m    elf_i386 -nostdlib -T tools/kernel.ld -o bin/kernel  obj/kern/init/init.o obj/kern/libs/stdio.o obj/kern/libs/readline.o obj/kern/debug/panic.o obj/kern/debug/kdebug.o obj/kern/debug/kmonitor.o obj/kern/driver/clock.o obj/kern/driver/console.o obj/kern/driver/picirq.o obj/kern/driver/intr.o obj/kern/trap/trap.o obj/kern/trap/vectors.o obj/kern/trap/trapentry.o obj/kern/mm/pmm.o  obj/libs/string.o obj/libs/printfmt.o
```

最后，执行最初的生成ucore.img的命令

```bash
dd if=/dev/zero of=bin/ucore.img count=10000
记录了10000+0 的读入
记录了10000+0 的写出
dd if=bin/bootblock of=bin/ucore.img conv=notrunc
记录了1+0 的读入
记录了1+0 的写出
dd if=bin/kernel of=bin/ucore.img seek=1 conv=notrunc
记录了146+1 的读入
记录了146+1 的写出
```

这一段首先创建了一个有10000个块的文件，每个块默认512字节，用0填充，即位ucore.img；之后将bootblock写到第一个块中，之后将kernel写入。这样ucore.img就生成完毕了。

- [练习1.2]

一个磁盘主引导扇区只有512字节。且最后的两个字节为0x55AA。

## 练习2

- [练习2.1] 从 CPU 加电后执行的第一条指令开始,单步跟踪 BIOS 的执行。

首先修改Makefile文件,添加一个新的target`debug-init`

```makefile
debug-init: $(UCOREIMG)
	$(V)$(TERMINAL) -e "$(QEMU) -S -s -d in_asm -D $(BINDIR)/q.log -monitor stdio -hda $< -serial null"
	$(V)sleep 2
	$(V)$(TERMINAL) -e "gdb -q -x tools/gdbinit"
```

其中第一条指令含义为，新启动一个gnome终端并运行debug状态的qemu，并将打印的汇编代码输出到`q.log`文件中。
同时将文件`tools/gdbinit`的内容修改为：

```
file bin/kernel
target remote :1234
set architecture i8086
break kern_init
```

该文件是gdb在初始化时执行的命令，首先加载`bin/kernel`的符号，再对qemu的1234端口进行连接，并设置模拟的CPU架构为`i8086`，便于进一步调试。

经过实验，gdb终端在访问`$pc`寄存器/变量时，并没有考虑段机制中`CS`寄存器的偏移，而qemu提供的终端考虑到了这一点，可以很方便地查看当前的运行状态。CPU初始时处于32位实模式，第一条运行的指令为：

    (qemu) x $pc
    0xfffffff0:  ljmp   $0xf000,$0xe05b

经过长跳转之后，CPU进入16位实模式，BIOS指令大致为：

```
(qemu) x /50i $pc
0x000fe05b:  cmpl   $0x0,%cs:0x6c48
0x000fe062:  jne    0xfd2e1
0x000fe066:  xor    %dx,%dx
0x000fe068:  mov    %dx,%ss
0x000fe06a:  mov    $0x7000,%esp
0x000fe070:  mov    $0xf3691,%edx
0x000fe076:  jmp    0xfd165
0x000fe079:  push   %ebp
...
```