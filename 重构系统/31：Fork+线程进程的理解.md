

# 31：Fork+线程进程的理解





进程控制块，其内存是由内核分配的。

```c
/* 进程或线程的pcb,程序控制块 */
struct task_struct {
   uint32_t* self_kstack;	 // 各内核线程都用自己的内核栈
   pid_t pid;
   enum task_status status;
   char name[16];
   uint8_t priority;
   uint8_t ticks;	   // 每次在处理器上执行的时间嘀嗒数
/* 此任务自上cpu运行后至今占用了多少cpu嘀嗒数,
 * 也就是此任务执行了多久*/
   uint32_t elapsed_ticks;
/* general_tag的作用是用于线程在一般的队列中的结点 */
   struct list_elem general_tag;				    
/* all_list_tag的作用是用于线程队列thread_all_list中的结点 */
   struct list_elem all_list_tag;
   uint32_t* pgdir;              // 进程自己页表的虚拟地址
   struct virtual_addr userprog_vaddr;   // 用户进程的虚拟地址
   struct mem_block_desc u_block_desc[DESC_CNT];   // 用户进程内存块描述符
   int32_t fd_table[MAX_FILES_OPEN_PER_PROC];	// 已打开文件数组
   uint32_t cwd_inode_nr;	 // 进程所在的工作目录的inode编号
   int16_t parent_pid;		 // 父进程pid
   uint32_t stack_magic;	 // 用这串数字做栈的边界标记,用于检测栈的溢出
};
```

一个PCB可以代表进程或线程，用户进程的空间是规定好的，0~3GB用以存放代码区，数据区，堆，用户栈，其用户栈是可以预测到的，也是编译器规划好的。但是内核栈就不一样了，可能是因为每个线程对于内核空间是共享的，故一个指定的虚拟内核空间实际上就对应于一块实际制定的内核空间，而用户进程里面一个制定的虚拟用户空间并不唯一对应一个用户空间。（就比如进程a的0xc0001的物理内存不是进程b的0xc0001的物理内存，但是线程a内核栈指针的0xc0000051和线程b内核栈指针的0xc0000051所对应的物理内存是同一块） 故也许这就是内核栈需要维护，用户栈不需要维护的原因吧。









实现线程有两种方式一个是用户线程一个是内核线程，我们实现的是内核线程，对于内核线程来说，线程用到的栈就是内核栈，各内核线程都用自己的内核栈，线程内核栈分为两个：**中断栈intr_stack**（此结构用于中断发生时保护程序（线程或进程）的上下文环境，进程或线程被外部中断或软中断打断时，会按照此结构压入上下文寄存器，intr_exit中的出栈操作时此结构的逆操作，此栈在线程自己的内核栈中位置固定，所以在页的最顶端）和**线程栈thread_stack**（线程自己的栈，用于存储线程中待执行的函数，此结构在线程自己的内核栈中位置不固定，仅用在switch_to时保存线程环境，实际位置取决于实际运行情况）

```c
/***********   中断栈intr_stack   ***********
 * 此结构用于中断发生时保护程序(线程或进程)的上下文环境:
 * 进程或线程被外部中断或软中断打断时,会按照此结构压入上下文
 * 寄存器,  intr_exit中的出栈操作是此结构的逆操作
 * 此栈在线程自己的内核栈中位置固定,所在页的最顶端
********************************************/
struct intr_stack {
    uint32_t vec_no;	 // kernel.S 宏VECTOR中push %1压入的中断号
    uint32_t edi;
    uint32_t esi;
    uint32_t ebp;
    uint32_t esp_dummy;	 // 虽然pushad把esp也压入,但esp是不断变化的,所以会被popad忽略
    uint32_t ebx;
    uint32_t edx;
    uint32_t ecx;
    uint32_t eax;
    uint32_t gs;
    uint32_t fs;
    uint32_t es;
    uint32_t ds;

/* 以下由cpu从低特权级进入高特权级时压入 */
    uint32_t err_code;		 // err_code会被压入在eip之后
    void (*eip) (void);
    uint32_t cs;
    uint32_t eflags;
    void* esp;
    uint32_t ss;
};


/***********  线程栈thread_stack  ***********
 * 线程自己的栈,用于存储线程中待执行的函数
 * 此结构在线程自己的内核栈中位置不固定,
 * 用在switch_to时保存线程环境。
 * 实际位置取决于实际运行情况。
 ******************************************/
struct thread_stack {
   uint32_t ebp;
   uint32_t ebx;
   uint32_t edi;
   uint32_t esi;

/* 线程第一次执行时,eip指向待调用的函数kernel_thread 
其它时候,eip是指向switch_to的返回地址*/
   void (*eip) (thread_func* func, void* func_arg);

/*****   以下仅供第一次被调度上cpu时使用   ****/

/* 参数unused_ret只为占位置充数为返回地址 */
   void (*unused_retaddr);
   thread_func* function;   // 由Kernel_thread所调用的函数名
   void* func_arg;    // 由Kernel_thread所调用的函数所需的参数
};

```





拷贝父进程本身所占资源给子进程

```c
/* 拷贝父进程本身所占资源给子进程 */
static int32_t copy_process(struct task_struct* child_thread, struct task_struct* parent_thread) {
   /* 内核缓冲区,作为父进程用户空间的数据复制到子进程用户空间的中转 */
   void* buf_page = get_kernel_pages(1);
   if (buf_page == NULL) {
      return -1;
   }

   /* a 复制父进程的pcb、虚拟地址位图、内核栈到子进程 */
   if (copy_pcb_vaddrbitmap_stack0(child_thread, parent_thread) == -1) {
      return -1;
   }

   /* b 为子进程创建页表,此页表仅包括内核空间 */
   child_thread->pgdir = create_page_dir();
   if(child_thread->pgdir == NULL) {
      return -1;
   }

   /* c 复制父进程进程体及用户栈给子进程 */
   copy_body_stack3(child_thread, parent_thread, buf_page);

   /* d 构建子进程thread_stack和修改返回值pid */
   build_child_stack(child_thread);

   /* e 更新文件inode的打开数 */
   update_inode_open_cnts(child_thread);

   mfree_page(PF_KERNEL, buf_page, 1);
   return 0;
}
```

内核空间对于各个线程来说是共享的，但是各个线程的内核栈不是共享的，各自有各自的内核栈。进程与线程的关系是进程是资源容器，线程是资源使用者。进程与线程的 区别是线程没有自己独享的资源，因此没有自己的地址空间，它要依附在进程的地址空间中从而借助进程的资源运行。说白了就是线程没有自己的页表，而进程有。 **线程是操作系统能够进行运行调度的最小单位。 就是一段代码可以作为整体受到调度的区域，本质上就是一段作为整体能够受到调度的代码块，通过访问进程给的资源去运行，而这些资源就包括代码区和数据区，而用于控制线程运行状态的便是线程栈**



首先看步骤a:copy_pcb_vaddrbitmap_stack

```c
/* 将父进程的pcb、虚拟地址位图拷贝给子进程 */
static int32_t copy_pcb_vaddrbitmap_stack0(struct task_struct* child_thread, struct task_struct* parent_thread) {
/* a 复制pcb所在的整个页,里面包含进程pcb信息及特级0极的栈,里面包含了返回地址, 然后再单独修改个别部分 */
   memcpy(child_thread, parent_thread, PG_SIZE);
   child_thread->pid = fork_pid();
   child_thread->elapsed_ticks = 0;
   child_thread->status = TASK_READY;
   child_thread->ticks = child_thread->priority;   // 为新进程把时间片充满
   child_thread->parent_pid = parent_thread->pid;
   child_thread->general_tag.prev = child_thread->general_tag.next = NULL;
   child_thread->all_list_tag.prev = child_thread->all_list_tag.next = NULL;
   block_desc_init(child_thread->u_block_desc);
/* b 复制父进程的虚拟地址池的位图 */
   uint32_t bitmap_pg_cnt = DIV_ROUND_UP((0xc0000000 - USER_VADDR_START) / PG_SIZE / 8 , PG_SIZE);
   void* vaddr_btmp = get_kernel_pages(bitmap_pg_cnt);
   if (vaddr_btmp == NULL) return -1;
   /* 此时child_thread->userprog_vaddr.vaddr_bitmap.bits还是指向父进程虚拟地址的位图地址
    * 下面将child_thread->userprog_vaddr.vaddr_bitmap.bits指向自己的位图vaddr_btmp */
   memcpy(vaddr_btmp, child_thread->userprog_vaddr.vaddr_bitmap.bits, bitmap_pg_cnt * PG_SIZE);
   child_thread->userprog_vaddr.vaddr_bitmap.bits = vaddr_btmp;
   /* 调试用 */
   ASSERT(strlen(child_thread->name) < 11);	// pcb.name的长度是16,为避免下面strcat越界
   strcat(child_thread->name,"_fork");
   return 0;
}
```

我们设计的PCB占用一个页，故将父进程页面的数据复制到子进程页面，里面包括了PCB信息和**内核栈数据**（注意是复制所有数据而不是单纯复制内核栈指针，内核栈指针后来是要改的，父进程和子进程并不共享一个内核栈，因为接下来会分叉，子进程和父进程调用的函数可能不一样涉及到的内核函数也可能不一样，更不能公用一个内核栈），这是整体复制，然后再稍微修改一下个别部分。然后此时子进程的位图指针指向的还是父进程的位图空间，子进程和父进程只共享内核空间，其他的数据内存都不共享，故子进程需要重新分配一个虚拟内存池位图。





至此，我们只是单纯的复制了PCB结构的数据，接下来为子进程创建页表，从而使得子进程能够得到虚拟空间，以能够存储父进程的资源（代码区，数据区，用户栈，堆） ，直接创建页表即可，一个进程有了页表就代表其有了虚拟空间（开始创的页表只含有内核空间的项目，到后来要用到用户空间就要慢慢一个个的创建PTE或者PDE），也就代表它有能力获得物理内存。（虚拟内存位图是用来管理的）



至此，子进程获得了自己的PCB数据，也获得了在内存中存储的能力（获取内存的能力），那么就要开始将父进程的数据复制到子进程 copy_body_stack3，遍历父进程的所有页，查找所有已有数据的页 通过内核区的缓存buf_page（内核空间是共享的）去过渡转存到子进程的空间中去。



至此，子进程有了自己的PCB数据，有存储的能力，也有了父进程的资源，那么现在就应该要到了fork返回的时候了，fork返回本质上是cs：ip执行到某个地方，更直白地说，就是控制指令流，那么这个时候就到了线程出场了，进程把资源都准备好了就等待调度器来调度线程啦，线程就要开始运行了，子进程是由调度器schedule调度执行的，它要用到switch_to函数，而switch_to函数是要通过找线程栈thread_stack中恢复上下文，因此我们要想办法构建合适的thread_stack。很显然thread_stack就是用来恢复上下文的，说简单点我们就是要在thread_stack栈中为esi,edi,ebx,ebp安排位置，其中最重要的是thread_stack里面的ebp，由于我们是现在是处于中断，所以我们必须要把ebp设置为pcb中偏移量为0的地方，即task_struct中的self_kstack（其实就是中断栈，里面保存了中断恢复上下文环境的数据），将来switch_to要用它作为栈顶，并且执行一系列的pop来恢复上下文，这是第一部分的设置即恢复上下文，第二部分的设置则是在thread里面把ret设置为intr_exit的地址，即退出中断的入口，这就保证了子进程被调度时，可以直接从中断返回，也就是实现了从fork之后的代码处继续执行的目的。



至此子进程被调度后也开始继续执行代码了。以上就是线程fork核心的全流程。





封装下实现内核级sys_fork()

```c
/* fork子进程,内核线程不可直接调用 */
pid_t sys_fork(void) {
   struct task_struct* parent_thread = running_thread();
   struct task_struct* child_thread = get_kernel_pages(1);    // 为子进程创建pcb(task_struct结构)
   if (child_thread == NULL) {
      return -1;
   }
   ASSERT(INTR_OFF == intr_get_status() && parent_thread->pgdir != NULL);

   if (copy_process(child_thread, parent_thread) == -1) {
      return -1;
   }

   /* 添加到就绪线程队列和所有线程队列,子进程由调试器安排运行 */
   ASSERT(!elem_find(&thread_ready_list, &child_thread->general_tag));
   list_append(&thread_ready_list, &child_thread->general_tag);
   ASSERT(!elem_find(&thread_all_list, &child_thread->all_list_tag));
   list_append(&thread_all_list, &child_thread->all_list_tag);
   
   return child_thread->pid;    // 父进程返回子进程的pid
}


```



至于调度的细节，有一个：线程的运行嘛，创建线程，加入就绪队列，等待中断调用即可（可以认为中断是一个主动的过程）。

easy!





###### 第一个进程init和线程init()

由于执行init（）函数的内核线程和init进程的进程标识符都是1，它们又都叫init，因此init（）函数和init进程容易造成概念上的模糊不清。

1、init（）函数是内核代码的一部分，在内核态运行，是独立的可执行代码的一部分。
2、init进程在Linux操作系统中是一个具有特殊意义的进程，它是由内核启动并运行的第一个用户进程，因此它不是运行在内核态，而是运行在用户态。它的代码不是内核本身的一部分，而是存放在硬盘上可执行文件的映象中，和其他用户进程没有什么两样。

进程是Linux内核最基本的抽象之一，它是处于执行期的程序，或者说“进程=程序+执行”。但是进程并不仅局限于一段可执行代码(代码段)，它还包括进程需要的其他资源，例如打开的文件、挂起的信号量、内存管理、处理器状态、一个或者多个执行线程和数据段等。Linux内核通常把进程叫作是任务(task)，因此进程控制块(processing control block，PCB)也被命名为struct task_struct。



线程只是进程的一部分，进程代表一个整体有独立地址在任意时刻可以运行可以不在运行，不运行时它也存在，需要用时就直接跳到该进程代码区。而线程只是一个运行的抽象，它只代表一块运行的指令流，没有任何存储空间和独立地址。

Linux内核在启动时会有一个init_task进程，它是系统所有进程的“鼻祖”，称为0号进程或idle进程，当系统没有进程需要调度时，调度器就会去执行idle进程。idle进程在内核启动( start_kemel()函数）时静态创建，所有的核心数据结构都预先静态赋值。