## BOCHS



`bochs 的硬件调试体现在： （1）调试时可以查看页表、gdt、idt 等数据结构； （2）可以查看栈中数据； （3）可以反汇编任意内存； （4）实模式、保护模式互相变换时提醒； （5）中断发生时提醒`



大体上 bochs 的调试命令分为“Debugger control”类、“Execution control”类、“Breakpoint management” 类、“CPU and memory contents”类。

![image-20220701172217181](D:/TYPIC/image-20220701172217181.png)

一.查看内存：在“CPU and memory contents”类中，有x,xp命令。它们的区别是 x 命令后接线性地址，xp 命令后接 physical 物理地址。说明一下，bochs 中用到的“字”并不是 2 字节，而是 4 字节。所以如果不指 定数据单位大小，默认以 4 字节为单位来显示。例如 xp 0x7c00，将显示从 0x7c00 开始的 4 个字节

![image-20220701172350270](D:/TYPIC/image-20220701172350270.png)



`由于默认 xp 以 4 字节来显示，所以 xp 中斜杠后面指定的数字 2，最终会让 xp 显示 8 个字节。提醒 一下，咱们 bochs 模拟的是 x86 平台，它是小端字节序。咱们只看 1 个 4 字节，先从低地址看，最低位是 ea，`

![image-20220701172428864](D:/TYPIC/image-20220701172428864.png)

```asm
u/1 0xffff0  表示以默认显示字节的1倍反汇编显示0xffff0内存
```































































附录

`y “Debugger control”类
q|quit|exit，左边三个命令任意一个都能退出调试状态，关闭虚拟机，一般用 q 最简单。
set 是指令族，咱们通常用 set 设置寄存器的值，这个较常用。
例如 set reg = val。可以设置的寄存器包括通用寄存器和段寄存器。
（2）也可以设置每次停止执行时，是否反汇编指令：set u on|off。
show 是指令族，有很多子功能，咱们常用的就下面这 3 个。
（1）show mode 
每次 CPU 变换模式时就提示，模式是指保护模式、实模式，比如从实模式进入到保护模式时会有提示。
（2）show int 
每次有中断时就提示，同时显示三种中断类型，这三种中断类型包括“softint”、“extint”和“iret”。
可以单独显示某类中断，如执行 show softint 只显示软件主动触发的中断，show extint 则只显示来自
外部设备的中断，show iret 只显示 iretd 指令有关的信息。
（3）show call 
每次有函数调用发生时就会提示。
traceon|off 如果此项设为 on，每次执行一条指令，bochs 都会将反汇编的代码打印到控制台，这样在
单步调试时免得看源码了。
u|disasm [/num] [start] [end] 
将物理地址 start 到 end 之间的代码反汇编，如果不指定地址，则反汇编 EIP 指向的内存。num 指定
反汇编的指令数。
setsize = 16|32|64 在使用反汇编命令时，用来告诉调试器段的大小。
set 指令也会设置在停止时是否反汇编命令。前面 set 命令中有说过。
ctrl+c 中断执行，回到 bochs 控制台。
y“Execution control”类
c|cont|continue，左边列出的三个命令都意为向下持续执行，若没断点则一直运行下去。最常用的是 c。
s|step [count] 执行 count 条指令，count 是指定单步执行的指令数，若不指定，count 默认为 1。此指
令若遇到函数调用，则会进入函数中去执行。最常用的是 s。
p|n|next 执行 1 条指令，若待执行的指令是函数调用，不管函数内有多少指令，把整个函数当作一个
整体来执行。最常用的是 n。
y“Breakpoint management”类
以地址打断点：
vb|vbreak [seg：off] 以虚拟地址添加断点，程序执行到此虚拟地址时停下来，注意虚拟地址是“段：
段内偏移”的形式。最常用的是 vb。
lb|lbreak [addr]以线性地址添加断点，程序执行到此线性地址时停下来。最常用的是 lb。
pb|pbreak|b|break [addr]以物理地址添加断点。程序执行到此物理地址时停下来。b 比较常用。
异步社区会员 databus(17602509427) 专享 尊重版权
3.4 bochs 调试方法
117
以指令数打断点：
sb [delta] delta 表示增量，意味再执行 delta 条指令程序就中断。
sba [time] CPU 从运行开始，执行第 time 条指令时中断，从 0 开始的指令数。
以读写 IO 打断点：
watch 也有子命令，常用的是这两个。
watch r|read [phy_addr] 设置读断点，如果物理地址 phy_addr 有读操作则停止运行。
watch w|write [phy_addr] 设置写断点，如果物理地址 phy_addr 有写操作则停止运行。此命令非常有用，
如果某块内存不知何时被改写了，可以设置此中断。
watch 显示所有读写断点。
unwatch 清除所有断点。
unwatch [phy_addr] 清除在此地址上的读写断点。
blist 显示所有断点信息，功能等同于 info b。
bpd|bpe [n] 禁用断点（break point disable）/启用断点（break point enable），n 是断点号，可以用 blist
命令先检查出来。
d|del|delete [n] 删除某断点，n 是断点号，可以用 blist 命令先检查出来。D 最常用。
y“CPU and memory contents”类
x /nuf [line_addr] 显示线性地址的内容。n、u、f 是三个参数，都是可选的，如果没有指定，则 n 为 1，
u 是 4 字节，f 是十六进制。解释如下。
n 显示的单元数
u 每个显示单元的大小，u 可以是下列之一：
（1）b 1 字节；
（2）h 2 字节；
（3）w 4 字节；
（4）g 8 字节。
f 显示格式，f 可以是下列之一：
（1）x 按照十六进制显示；
（2）d 十进制显示；
（3）u 按照无符号十进制显示；
（4）o 按照八进制显示；
（5）t 按照二进制显示；
（6）c 按照字符显示；
（7）s 按照 ASCIIz 显示；
（8）i 按照 instr 显示。
xp /nuf [phy_addr] 显示物理地址 phy_addr 处的内容，注意和 x 的区别，x 是线性地址。
setpmem [phy_addr] [size] [val] 设置以物理内存 phy_addr 为起始，连续 size 个字节的内容为 val。此命令非常
有用，在某些情况下不易调试时，可以在程序中通过某个地址的值来判断分支，需要用 setpmem 来配合。
注意啦，size 最多只能设置 4 个字节宽度的数据，如果 size 大于 4 便会报错：Error：setpmem： bad length 
value = 8。size 小于等于 4 是正确的，setpmem 0x7c00 4 0x9。
r|reg|regs|registers 任意四个命令之一便可以显示 8 个通用寄存器的值+eflags 寄存器+eip 寄存器。r 是
我常用的查看寄存器的命令。
ptime 显示 Bochs 自启动之后，总执行指令数。其实这个命令不常用，感兴趣的同学可以用 ptime 和
“sb 指令数”来验证结果是否正确。
print-stack [num] 显示堆栈，num 默认为 16，表示打印的栈条目数。输出的栈内容是栈顶在上，低地
址在上，高地址在下。这和栈的实际扩展方向相反，这一点请注意。


?|calc 内置的计算器。
info 是个指令族，执行 help info 时可查看其所有支持的子命令，如下：
info pb|pbreak|b|break 查看断点信息，等同于 blist。
info CPU 显示 CPU 所有寄存器的值，包括不可见寄存器。
info fpu 显示 FPU 状态。
info idt 显示中断向量表 IDT。
info gdt [num]显示全局描述符表 GDT，如果加了 num，只显示 gdt 中第 num 项描述符。
info ldt 显示局部描述符表 LDT。
info tss 显示任务状态段 TSS。
info ivt [num]显示中断向量表 IVT。和 gdt 一样，如果指定了 num，则只会显示第 num 项的中断向量。


info flags|eflags 显示状态寄存器，其实在用 r 命令显示寄存器值时也会输出 eflags 的状态，还会输出
通用寄存器的值，我通常会用 r 来看。
sreg 显示所有段寄存器的值。
dreg 显示所有调试寄存器的值。
creg 显示所有控制寄存器的值。
info tab 显示页表中线性地址到物理地址的映射。
page line_addr 显示线性地址到物理地址间的映射。`