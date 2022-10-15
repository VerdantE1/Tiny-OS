# MBR完善

前置：例子

1.汇编语言能直接操纵寄存器存储参数，用哪个寄存器存储参数没硬性规定

2.nasm编译器预处理：%include "文件名"

3.nasm编译器宏定义：宏名 equ 值

4.cmp al,0x08 是和0x08做减法运算，结果一些性质存储在标志寄存器中

有条件跳转指令所用的不同条件

![image-20220702150240496](D:/TYPIC/image-20220702150240496.png)

5.loop根据cx寄存器的特定功能判断是否继续循环

6.mul 操作数。操作数可以是内存或寄存器，乘法运算要有两个参数，被乘数被隐藏在al或ax寄存器中，如果操作数是8位被乘数就是al，乘积是16位，结果在ax。若操作数是16位，被乘数是ax，乘积就是32位，积的高16位在另外一个参数寄存器中，低16位在ax中。

7.实模式下的CPU偏移地址寻址能力为16位，故bx所指引的内存块是小于64KB，所以要知道所载入的程序是多大，如果大于65535则会从bx初始位置开始覆盖。

8.ret指令将栈中的返回地址重新加载到程序计数器中。恢复之前的执行顺序。

9.字体

![image-20220702153452393](D:/TYPIC/image-20220702153452393.png)

10.硬盘控制器的端口

![](D:/TYPIC/image-20220702155224443.png)

11.device寄存器和status寄存器结构

![image-20220702155256931](D:/TYPIC/image-20220702155256931.png)

12.in out命令硬性规定只能用寄存器/内存交换数据

# 思路

[^加载内核loader整体思路]:草稿

<img src="D:/TYPIC/IMG_20220702_150922.jpg" alt="IMG_20220702_150922" style="zoom: 25%;" />





代码：eax存的是磁盘的LBA地址，在0x900



首先查看要使用的盘的相关信息，可以知道该盘为主盘，用主通道。从内存为0x1f0的地址为该端口（寄存器）地址。

<img src="D:/TYPIC/%5DOXE%7D%7DT6UIYVH3SAIQ642A3.png" alt="img" style="zoom: 67%;" />

再看上图使用相匹配的端口地址，我们要用主通道。

开始实现，第一步，往主通道的sector count寄存器中写入待操作的扇区数

```asm
	mov dx,0x1f2   ;端口地址赋予dx
	mov al,cl   ;8位传输
	out dx,al   
	mov eax,esi  ;恢复ax
```

第二步，将LBA存入控制器的寄存器中

```asm
;LBA 地址 7～0 位写入端口 0x1f3 
  mov dx,0x1f3 
  out dx,al 
 
 ;LBA 地址 15～8 位写入端口 0x1f4 
 mov cl,8 
 shr eax,cl 
 mov dx,0x1f4 
 out dx,al 
 
 ;LBA 地址 23～16 位写入端口 0x1f5 
 shr eax,cl 
 mov dx,0x1f5 
 out dx,al 
 
 shr eax,cl 
 and al,0x0f ;lba 第 24～27 位
 or al,0xe0 ; 设置 7～4 位为 1110,表示 lba 模式
 mov dx,0x1f6 
 out dx,al 
```

第三步，在command寄存器中输入命令，0x20代表读命令

```asm
 mov dx,0x1f7 
 mov al,0x20 
 out dx,al 
```



第四步，检查硬盘

```asm
.not_ready: 
 nop 
 in al,dx 
 and al,0x88 ;第 4 位为 1 表示硬盘控制器已准备好数据传输
 ;第 7 位为 1 表示硬盘忙
 cmp al,0x08 
 jnz .not_ready ;若未准备好,继续等
```



第五步，读取数据

```asm
 mov ax, di 
 mov dx, 256 
 mul dx 
 mov cx, ax 
; di 为要读取的扇区数,一个扇区有 512 字节,每次读入一个字
 ; 共需 di*512/2 次,所以 di*256 
 mov dx, 0x1f0 
 .go_on_read: 
 in ax,dx 
 mov [bx],ax 
 add bx,2 
 loop .go_on_read 
 ret 
```







所有代码

```asm
  ;主引导程序
  ;------------------------------------------------------------ 
  %include "boot.inc" 
  SECTION MBR vstart=0x7c00 
  mov ax,cs 
  mov ds,ax 
  mov es,ax 
  mov ss,ax 
  mov fs,ax 
  mov sp,0x7c00 
  mov ax,0xb800 
  mov gs,ax 
  
  ;清屏
  ;利用 0x06 号功能，上卷全部行，则可清屏
  ; ----------------------------------------------------------- 
  ;INT 0x10 功能号：0x06 功能描述：上卷窗口
  ;------------------------------------------------------ 
  ;输入：
  ;AH 功能号= 0x06 
  ;AL = 上卷的行数（如果为 0，表示全部）
  ;BH = 上卷行属性
  ;(CL,CH) = 窗口左上角的(X,Y)位置
  ;(DL,DH) = 窗口右下角的(X,Y)位置
  ;无返回值：

  mov ax, 0600h 
  mov bx, 0700h 
  mov cx, 0 ; 左上角: (0, 0) 
  mov dx, 184fh ; 右下角: (80,25), 
  ; 因为 VGA 文本模式中，一行只能容纳 80 个字符，共 25 行
  ; 下标从 0 开始，所以 0x18=24，0x4f=79 
  int 10h ; int 10h 
 
  ; 输出字符串:MBR 
  mov byte [gs:0x00],'1' 
  mov byte [gs:0x01],0xA4 
  
  mov byte [gs:0x02],' ' 
  mov byte [gs:0x03],0xA4 
  
  mov byte [gs:0x04],'M' 
  mov byte [gs:0x05],0xA4 
 ;A 表示绿色背景闪烁,4 表示前景色为红色
  
  mov byte [gs:0x06],'B' 
  mov byte [gs:0x07],0xA4 
  
  mov byte [gs:0x08],'R' 
  mov byte [gs:0x09],0xA4 
 
  mov eax,LOADER_START_SECTOR ; 起始扇区 lba 地址
  mov bx,LOADER_BASE_ADDR ; 写入的地址
  mov cx,1 ; 待读入的扇区数
  
  call rd_disk_m_16 ; 以下读取程序的起始部分(一个扇区) 
  
  jmp LOADER_BASE_ADDR 
  
  ;------------------------------------------------------------------------------- 
  ;功能:读取硬盘 n 个扇区
  rd_disk_m_16: 
 ;------------------------------------------------------------------------------- 
  ; eax=LBA 扇区号
  ; bx=将数据写入的内存地址
  ; cx=读入的扇区数
  mov esi,eax ;备份 eax 
  mov di,cx ;备份 cx 
  ;读写硬盘: 
  ;;;;;;;;;;第 1 步:设置要读取的扇区数
  mov dx,0x1f2 
  mov al,cl 
  out dx,al ;读取的扇区数

  mov eax,esi ;恢复 ax 
 
 ;;;;;;;;;;第 2 步:将 LBA 地址存入 0x1f3 ～ 0x1f6 

 ;LBA 地址 7～0 位写入端口 0x1f3 
  mov dx,0x1f3 
  out dx,al 
 
 ;LBA 地址 15～8 位写入端口 0x1f4 
 mov cl,8 
 shr eax,cl 
 mov dx,0x1f4 
 out dx,al 
 
 ;LBA 地址 23～16 位写入端口 0x1f5 
 shr eax,cl 
 mov dx,0x1f5 
 out dx,al 
 
 shr eax,cl 
 and al,0x0f ;lba 第 24～27 位
 or al,0xe0 ; 设置 7～4 位为 1110,表示 lba 模式
 mov dx,0x1f6 
 out dx,al 
 

 ;;;;;;;;;;;;;第 3 步:向 0x1f7 端口写入读命令,0x20 
 mov dx,0x1f7 
 mov al,0x20 
 out dx,al 
 
 ;;;;;;;;;;;;第 4 步:检测硬盘状态
 .not_ready: 
 nop 
 in al,dx 
 and al,0x88 ;第 4 位为 1 表示硬盘控制器已准备好数据传输
 ;第 7 位为 1 表示硬盘忙
 cmp al,0x08 
 jnz .not_ready ;若未准备好,继续等
 
 ;;;;;;;;;第 5 步:从 0x1f0 端口读数据
 mov ax, di 
 mov dx, 256 
 mul dx 
 mov cx, ax 
; di 为要读取的扇区数,一个扇区有 512 字节,每次读入一个字
 ; 共需 di*512/2 次,所以 di*256 
 mov dx, 0x1f0 
 .go_on_read: 
 in ax,dx 
 mov [bx],ax 
 add bx,2 
 loop .go_on_read 
 ret 
 
 times 510-($-$$) db 0 
 db 0x55,0xaa 
```

