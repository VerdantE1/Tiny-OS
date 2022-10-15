## MBR的初试编写

MBR是BOOT程序在进行检查硬件，硬件初始化，建立中断向量表之后最后一项工作即查找MBR文件，其标志是扇区结尾有魔数0x55,0xaa。

MBR要合理规划地址，高级语言不能用，显然用汇编更方便（想用机器指令也行/坏笑）



要保证几点要求：保证文件大小是512字节且510，511字节是魔术0x55,0xaa，不足的由0填充。就此即可成为MBR文件，先简单测试下。在屏幕上输入字符串“1 MBR”



提示几点，<1>标号如code_start被汇编器认为是一个地址下面的code_start就是jmp `$`

 <2>`$`是伪指令，其属于隐式地藏在本行代码前的标号，也就是编译器给当前行安排的地址。`$$`也是如此，汇编器认为是本section的起始地址。

```asm
code_start：
 jmp $ 
…… 
```

<3>如果用了vstart关键字，用来修饰该section，那么若用vstart之后想获得本section在文件中的真实偏移量（真实地址）如何做？section.节名.start.如果没有定义section，nasm默认全部代码为一个section,起始地址为0



<4>nasm简单用法

nasm -f  <format> <filename>  [-o <output> ]

<5> message db "1 MBR" 定义字符串之后message代表该字符串的地址，利用es:bp进行定位到该字符串调用int 0x10的13号子功能打印字符串。寄存器相当于是参数

<6>记得初始化段寄存器

代码

```asm
;初始化段寄存器，主引导程序，定义其位置位于0x7c00
SECTION MBR vstart=0x7c00
	mov ax,cs
	mov ds,ax
	mov es,ax
	mov ss,ax
	mov fs,ax
	mov gs,ax
	mov sp,0x7c00
;每个程序分配函数变量都需要栈,初始化结束。

;清屏操作,调用INT 0X10 ,功能号0x06,功能描述：上卷窗口
;AH 功能号=0x06,AL =上卷行数(若为0则表示全部)
;BH=上卷的属性
;(CL,CH) = 窗口左上角（x,y）位置，（DL，DH）=窗口右下角（x,y）位置
;无返回值：
	mov ax,0x600
	mov bx,0x700
	mov cx,0x0              ;左上角
	mov dx,0x184f           ;右下角（80,25），0x18为24,0x4f为79,一个16位连续数表示xy,采用小端表示法

	int 0x10;


;获取光标位置,3号子功能
	mov ah,0x3
	mov bh,0
	
	int 0x10                ;输出：ch=光标开始行，cl=光标结束行，dh=光标所在行号，dl=光标所在列号

;;;;;;;;;	获取光标结束	;;;;;;;;;;

;;;;;;;;;	打印字符串	;;;;;;;;;;

	mov ax,message 		;message is the  string of  head address
	mov bp,ax    		;es:bp为串首地址，es此时和cs一致，开头已经为sreg初始化
     		
     	mov cx,5
	mov ax,0x1301   	;ah=01 : option:show string,curso following
	mov bx,0x2

	int 0x10
;;;;;;;;  Over ;;;;;;;;;
	jmp $

	message db "1 MBR"
	times 510-($-$$) db 0
	db 0x55,0xaa
	

;6.29完成
```

nasm -o mbr.bin mbr.S

nasm -I ../include/  -o ./loader.bin ../loader.S



最后介绍下linux的dd命令

```shell
if=FILE 
read from FILE instead of stdin 
此项是指定要读取的文件。

of=FILE 
write to FILE instead of stdout 
此项是指定把数据输出到哪个文件。

bs=BYTES 
read and write BYTES bytes at a time (also see ibs=,obs=) 
此项指定块的大小，dd 是以块为单位来进行 IO 操作的，得告诉人家块是多大字节。此项是统计配置
了输入块大小 ibs 和输出块大小 obs。这两个可以单独配置。

count=BLOCKS 
copy only BLOCKS input blocks 
此项是指定拷贝的块数。

seek=BLOCKS 
skip BLOCKS obs-sized blocks at start of output 
此项是指定当我们把块输出到文件时想要跳过多少个块。

conv=CONVS 
convert the file as per the comma separated symbol list 
此项是指定如何转换文件。

append append mode (makes sense only for output; conv=notrunc suggested) 
这句话建议在追加数据时，conv 最好用 notrunc 方式，也就是不打断文件。
齐了，dd 的介绍就到这了，赶紧试验一下这个神奇的工具吧。

dd if=/your_path/mbr.bin of=/your_path/bochs/hd60M.img bs=512 count=1 conv=notrunc

path=/home/buzzlight/bochs
```

