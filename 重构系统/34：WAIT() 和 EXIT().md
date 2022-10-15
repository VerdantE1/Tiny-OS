## 34：WAIT() 和 EXIT()

![image-20220822103925468](C:\Users\Administrator\Desktop\备份\TYPIC\image-20220822103925468.png)

###### CRT简介

大伙儿知道，根据 C 调用约定，参数是通过栈来传递的，在同一个进程中，由于栈已经存在了，参 数可以直接压在栈中，被调函数便可以从栈中获取参数了，这使得咱们处理内部命令时很容易。获取参数 是在执行命令之前，要想把获取到的参数传递给某个命令，该命令所属的进程必须先有栈。但外部命令的 执行，实质上是加载一个用户程序的过程，shell 执行外部命令前，外部命令（用户进程）尚未创建，它的 栈当然也不存在了，参数就没法传递了吗？这该怎么办呢？

C 标准库是与操作系统平台无关的，它诞生之初就是为了实现用户程序跨操作系统平台而规约的标准 接口，使用户进程无论在哪个操作系统上调用同样的函数接口，执行的结果都是一样的。 C 运行库也称为 CRT（C RunTime library），它是与操作系统息息相关的，因为谁也不愿意重复造轮子， 故它的实现也基于 C 标准库，因此 CRT 属于 C 标准库的扩展。CRT 多是补充 C 标准库中没有的功能，为 适配本操作系统环境而定制开发的。因此 CRT 并不通用，只适用于在本操作系统上运行的程序。

CRT 都做了什么呢？很多，最主要的就是初始化运行环境，在进入 main 函数之前为用户进程准备条件，传递参数等，待条件准备好后再调用用户进程的 main 函数，这样用户进程才能顺利地跑起来。当用 户进程结束时，CRT 还要负责回收用户进程的资源。其实想想这也是必然的，main 函数是用户自己写的， 无论代码多少，总有结束那天（死循环不算），如果 main 执行到了边界，此时没有固定的代码执行，程序 不就“飞”了吗，也就是说处理器会越过边界自动向下取指令，cs:eip 寄存器中的值肯定就不对了，因此 程序不知道会跑哪里去了，处理器一直会执行到抛异常为止，操作系统也就失去了处理器的控制权，整个 计算机系统瘫痪了，这就是咱们经常在程序的最后添加死循环 代码“while(1)”的原因。综上所述，main 函数一定是被 call 指令调用的，必须有去有回，目的是当用户进程执行完用户所 写的 main 函数后能够执行固定的代码—系统调用 exit 或 _exit，这样用户程序陷入内核，使处理器的控制权重新回到操 作系统手中

```asm
[bits 32]
extern	 main
extern	 exit 
section .text
global _start
_start:
   ;下面这两个要和execv中load之后指定的寄存器一致
   push	 ebx	  ;压入argv
   push  ecx	  ;压入argc
   call  main

   ;将main的返回值通过栈传给exit,gcc用eax存储返回值,这是ABI规定的
   push  eax
   call exit
   ;exit不会返回

```

_start是ld默认的启动入口，可见main指定的参数先传入ebx,ecx，然后开始执行CRT，先压入ebx,ecx再调用call main。main执行完后，将返回值给eax,又再次回到了CRT,CRT再将返回值压入栈中，最后call exit完成回收资源及PCB并让用户程序陷入内核，**使处理器的控制权重新回到了操作系统手中。**

**而上述的传参数到ebx,ecx是由execv完成的。** 在文件 exec.c 中我们已经把新进程的参数压入内核栈中相应的寄存器，sys_execv 执行完成从 intr_exit 返回后，寄存器 ebx 是参数数组 argv 的地址，寄存器 ecx 是参数个数 argc。因此在第 7～8 行将它们 压入栈，此时的栈是用户栈，这就是之前咱们所说的，自己和自己协调好就行了：在 sys_execv 中，往 0 特 权级栈中哪个寄存器写入参数，此处就从哪个寄存器中获取参数，然后再压入用户栈为用户进程准备参数。

内核也是从_start后跳入main开始执行初始化，如果想执行用户程序就fork一个进程。



子进程的返回值不是手递手直接交给父进程的，进程都是独立的地址空间，即使是父子进程，他们之间也是相互独立不可互访的，因为这就是与线程的区别，进程想要互相通信必须要借用内核（无论是管道，消息队列，还是共享内存等进程间通信形式，无一例外都是借助内核这个中间人），子进程的返回值肯定首先要交给内核，然后父进程通过wait向内核要子进程的返回值。

另外，进程在调用exit时就表示进程生命周期结束了，其占用的资源可以被回收了，因此进程在调用exit后，内核会把该进程占用的大部分资源回收除了PCB，比如内存，页表等，但肯定不能将PCB回收，原因是里面存储着子进程的遗言，必须交付给父进程，父进程受到子进程的遗言后才能回收子进程的PCB，必须要交给父进程，父进程收到子进程的遗言后才能回收子进程的PCB。否则子进程的PCB会始终存在。



###### 僵尸进程和孤儿进程

僵尸进程就是针对子进程的返回值是否提交给父进程而提出的，父进程不调用wait，就无法获得子进程的返回值，从而内核就无法回收子进程PCB所占的空间，因此就会在队列中占据一个进程表项。僵尸进程是没有进程体的，但是有PCB存在。

父进程提前退出，它的所有子进程还在运行，没有一个执行了exit，因为它们的生命周期尚未结束，还在运行中个个都有进程体，这些进程就称为孤儿进程，当父进程提前退出时，这时候所有的子进程会被init进程收养（第一个进程，也是内核初始化的进程），init进程便是这些进程的新父进程，当子进程退出时，init便为其收尸。





###### WAIT和EXIT作用

总结： exit 是由子进程调用的，表面上功能是使子进程结束运行并传递返回值给内核，本质上是内核在幕后 会将进程除 pcb 以外的所有资源都回收。wait 是父进程调用的，表面上功能是使父进程阻塞自己，直到子 进程调用 exit 结束运行，然后获得子进程的返回值，本质上是内核在幕后将子进程的返回值传递给父进程 并会唤醒父进程，然后将子进程的 pcb 回收





```c
/* 等待子进程调用exit,将子进程的退出状态保存到status指向的变量.
 * 成功则返回子进程的pid,失败则返回-1 */
pid_t sys_wait(int32_t* status) {
   struct task_struct* parent_thread = running_thread();

   while(1) {
      /* 优先处理已经是挂起状态的任务 */
      struct list_elem* child_elem = list_traversal(&thread_all_list, find_hanging_child, parent_thread->pid);
      /* 若有挂起的子进程 */
      if (child_elem != NULL) {
	 struct task_struct* child_thread = elem2entry(struct task_struct, all_list_tag, child_elem);
	 *status = child_thread->exit_status; 

	 /* thread_exit之后,pcb会被回收,因此提前获取pid */
	 uint16_t child_pid = child_thread->pid;

	 /* 2 从就绪队列和全部队列中删除进程表项*/
	 thread_exit(child_thread, false); // 传入false,使thread_exit调用后回到此处
	 /* 进程表项是进程或线程的最后保留的资源, 至此该进程彻底消失了 */

	 return child_pid;
      } 

      /* 判断是否有子进程 */
      child_elem = list_traversal(&thread_all_list, find_child, parent_thread->pid);
      if (child_elem == NULL) {	 // 若没有子进程则出错返回
	 return -1;
      } else {
      /* 若子进程还未运行完,即还未调用exit,则将自己挂起,直到子进程在执行exit时将自己唤醒 */
	 thread_block(TASK_WAITING); 
      }
   }
}


/* 子进程用来结束自己时调用 */
void sys_exit(int32_t status) {
   struct task_struct* child_thread = running_thread();
   child_thread->exit_status = status; 
   if (child_thread->parent_pid == -1) {
      PANIC("sys_exit: child_thread->parent_pid is -1\n");
   }

   /* 将进程child_thread的所有子进程都过继给init */
   list_traversal(&thread_all_list, init_adopt_a_child, child_thread->pid);

   /* 回收进程child_thread的资源 */
   release_prog_resource(child_thread); 

   /* 如果父进程正在等待子进程退出,将父进程唤醒 */
   struct task_struct* parent_thread = pid2thread(child_thread->parent_pid);
   if (parent_thread->status == TASK_WAITING) {
      thread_unblock(parent_thread);
   }

   /* 将自己挂起,等待父进程获取其status,并回收其pcb */
   thread_block(TASK_HANGING);
}
```

