# 实验三——操作系统的引导

## 1. 改写 `bootsect.s` 主要完成如下功能：

`bootsect.s` 能在屏幕上打印一段提示信息

```
XXX is booting...
```
完成屏幕显示的关键代码如下：

```! 首先读入光标位置
    mov    ah,#0x03
    xor    bh,bh
    int    0x10

    ! 显示字符串“XXXos is running...”
    mov    cx,#25            ! 要显示的字符串长度
    mov    bx,#0x0007        ! page 0, attribute 7 (normal)
    mov    bp,#msg1
    mov    ax,#0x1301        ! write string, move cursor
    int    0x10

inf_loop:
    jmp    inf_loop        ! 后面都不是正经代码了，得往回跳呀
    ! msg1处放置字符串

msg1:
    .byte 13,10            ! 换行+回车
    .ascii "XXX os is running..."
    .byte 13,10,13,10            ! 两对换行+回车
    !设置引导扇区标记0xAA55
    .org 510
boot_flag:
    .word 0xAA55            ! 必须有它，才能引导
```
修改`bootsect.s`中`msg1`中打印的内容,将长度修改为`#28`

```
msg1:
	.byte 13,10
	.ascii "chensiyu is booting..."
	.byte 13,10,13,10
```
用 cd 命令进入 linux-0.11\boot，使用如下命令进行编译：
```
   $ as86 -0 -a -o bootsect.o bootsect.s
   $ ld86 -0 -s -o bootsect bootsect.o
```
通过dd工具移除32位的头，转成Image文件,然后运行
```
$ dd bs=1 if=bootsect of=Image skip=32

```
![](./ima/1.png)

## 2. 改写 `setup.s` 主要完成如下功能：

`bootsect.s` 能完成 `setup.s` 的载入，并跳转到 `setup.s` 开始地址执行。而 `setup.s` 向屏幕输出一行

```
Now we are in SETUP
```
首先编写一个 setup.s ，该 setup.s 可以就直接拷贝前面的 bootsect.s （可能还需要简单的调整）， 然后将其中的显示的信息改为：
```
Now we are in SETUP
```
接下来需要编写 bootsect.s 中载入 setup.s 的关键代码。原版 bootsect.s 中下面的代码就是做这个的。
```
load_setup:
    mov    dx,#0x0000               !设置驱动器和磁头(drive 0, head 0): 软盘0磁头
    mov    cx,#0x0002               !设置扇区号和磁道(sector 2, track 0):0磁头、0磁道、2扇区
    mov    bx,#0x0200               !设置读入的内存地址：BOOTSEG+address = 512，偏移512字节
    mov    ax,#0x0200+SETUPLEN      !设置读入的扇区个数(service 2, nr of sectors)，
                                    !SETUPLEN是读入的扇区个数，Linux 0.11设置的是4，
                                    !我们不需要那么多，我们设置为2
    int    0x13                     !应用0x13号BIOS中断读入2个setup.s扇区
    jnc    ok_load_setup            !读入成功，跳转到ok_load_setup: ok - continue
    mov    dx,#0x0000               !软驱、软盘有问题才会执行到这里。我们的镜像文件比它们可靠多了
    mov    ax,#0x0000               !否则复位软驱 reset the diskette
    int    0x13
    jmp    load_setup               !重新循环，再次尝试读取
ok_load_setup:
                                    !接下来要干什么？当然是跳到setup执行。
```
参考上一个任务进行编译并得到image文件运行  
```
as86 -0 -a -o bootsect.o bootsect.s
ld86 -0 -s -o bootsect bootsect.o

as86 -0 -a -o setup.o setup.s  
ld86 -0 -s -o setup setup.o

dd bs=1 if=bootsect of=Image skip=32
dd bs=1 if=setup of=Image skip=32 seek=512

```
![](./ima/2.png)
## 3. setup.s 获取基本硬件参数
下面是将硬件参数取出来放在内存 0x90000 的关键代码。
```
mov    ax,#INITSEG
mov    ds,ax        !设置ds=0x9000
mov    ah,#0x03     !读入光标位置
xor    bh,bh
int    0x10         !调用0x10中断
mov    [0],dx       !将光标位置写入0x90000.

!读入内存大小位置
mov    ah,#0x88
int    0x15
mov    [2],ax

!从0x41处拷贝16个字节（磁盘参数表）
mov    ax,#0x0000
mov    ds,ax
lds    si,[4*0x41]
mov    ax,#INITSEG
mov    es,ax
mov    di,#0x0004
mov    cx,#0x10
rep            !重复16次
movsb
```
按照实验手册上的提示写出打印16位数的汇编过程，使用call指令来调用。将打印的数放在dx寄存器中：
```
print_hex:      ! 以 16 进制方式打印栈顶的16位数
    mov cx,#4   ! 4 个十六进制数字
    mov dx,(bp) ! 将(bp)所指的值放入 dx 中，如果 bp 是指向栈顶的话
print_digit:
    rol dx,#4       ! 循环以使低 4 比特用上 !! 取 dx 的高 4 比特移到低 4 比特处。就是将dx整体向左移动4位，然后高4位依次移入低4位，
    mov ax,#0xe0f   ! ah=0xe功能号表示要打印字符，al=0x0f低8个字节。
    and al,dl       ! 取 dl 的低 4 比特值。
    add al,#0x30    ! 给 al 数字加上十六进制 0x30
    cmp al,#0x3a    ! 判断al是否大于等于0x3a，如果大于等于0x3a，那么al就是a~f。
    jl  outp        ! 是一个不大于十的数字
    add al,#0x07    ! 是a～f，要多加 7
outp:
    int 0x10
    loop print_digit
    ret
```
运行后查看输出
![](./ima/3.png)
查看bochs/bochsrc.bxrc文件,打印信息基本相符
![](./ima/4.png)
## 4.请找出 x86 计算机启动过程中，被硬件强制，软件必须遵守的两个“多此一举”的步骤（多找几个也无妨），说说它们为什么多此一举，并设计更简洁的替代方案。
1. 启动时进入实模式
原因：x86 CPU 启动时会自动进入实模式，这是为了向下兼容 16 位程序，比如 8086/8088 时代的程序。但对于现代处理器和操作系统来说，实模式的功能有限，它只能访问 1M 以下的内存，且没有内存保护等机制。现代处理器和操作系统完全可以直接在保护模式下运行，进入实模式显得多此一举。
替代方案：处理器在启动时不向下兼容实模式，直接进入 32 位或 64 位的保护模式，这样可以充分利用现代处理器的功能，如更大的内存寻址空间、内存保护机制等，提高系统的安全性和性能。

2. BIOS 的加电自检（POST）
原因：BIOS 在启动时会进行加电自检，以检测硬件是否正常工作，包括检测 CPU、内存、显卡、I/O 设备等。然而，现代硬件设备通常具有较好的稳定性和自我检测机制，而且操作系统也可以在运行过程中对硬件进行更全面、更高效的检测和管理。因此，BIOS 的加电自检在一定程度上增加了启动时间，显得有些多余。
替代方案：可以简化 BIOS 的加电自检过程，只进行最基本的硬件检测，如检测 CPU 是否正常工作、内存是否存在严重错误等。对于其他硬件设备的检测，可以交给操作系统在启动过程中进行，或者通过硬件自身的固件来完成。这样可以减少启动时间，提高系统的启动效率。
