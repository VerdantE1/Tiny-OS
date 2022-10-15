# 32:实现shell+一些命令原理





核心模块

```c
#define MAX_ARG_NR 16	   // 加上命令名外,最多支持15个参数

/* 存储输入的命令 */
static char cmd_line[MAX_PATH_LEN] = {0};
char final_path[MAX_PATH_LEN] = {0};      // 用于洗路径时的缓冲

/* 用来记录当前目录,是当前目录的缓存,每次执行cd命令时会更新此内容 */
char cwd_cache[MAX_PATH_LEN] = {0};

/* 输出提示符 */
void print_prompt(void) {
   printf("[rabbit@localhost %s]$ ", cwd_cache);
}



/* 从键盘缓冲区中最多读入count个字节到buf。*/
static void readline(char* buf, int32_t count) {
   assert(buf != NULL && count > 0);
   char* pos = buf;

   while (read(stdin_no, pos, 1) != -1 && (pos - buf) < count) { // 在不出错情况下,直到找到回车符才返回
      switch (*pos) {
       /* 找到回车或换行符后认为键入的命令结束,直接返回 */
	 case '\n':
	 case '\r':
	    *pos = 0;	   // 添加cmd_line的终止字符0
	    putchar('\n');
	    return;

	 case '\b':
	    if (cmd_line[0] != '\b') {		// 阻止删除非本次输入的信息
	       --pos;	   // 退回到缓冲区cmd_line中上一个字符
	       putchar('\b');
	    }
	    break;

	 /* ctrl+l 清屏 */
	 case 'l' - 'a': 
	    /* 1 先将当前的字符'l'-'a'置为0 */
	    *pos = 0;
	    /* 2 再将屏幕清空 */
	    clear();
	    /* 3 打印提示符 */
	    print_prompt();
	    /* 4 将之前键入的内容再次打印 */
	    printf("%s", buf);
	    break;

	 /* ctrl+u 清掉输入 */
	 case 'u' - 'a':
	    while (buf != pos) {
	       putchar('\b');
	       *(pos--) = 0;
	    }
	    break;

	 /* 非控制键则输出字符 */
	 default:
	    putchar(*pos);
	    pos++;
      }
   }
   printf("readline: can`t find enter_key in the cmd_line, max num of char is 128\n");
}




/* 分析字符串cmd_str中以token为分隔符的单词,将各单词的指针存入argv数组 */
static int32_t cmd_parse(char* cmd_str, char** argv, char token) {
   assert(cmd_str != NULL);
   int32_t arg_idx = 0;
   while(arg_idx < MAX_ARG_NR) {
      argv[arg_idx] = NULL;
      arg_idx++;
   }
   char* next = cmd_str;
   int32_t argc = 0;
   /* 外层循环处理整个命令行 */
   while(*next) {
      /* 去除命令字或参数之间的空格 */
      while(*next == token) {
	 next++;
      }
      /* 处理最后一个参数后接空格的情况,如"ls dir2 " */
      if (*next == 0) {
	 break; 
      }
      argv[argc] = next;

     /* 内层循环处理命令行中的每个命令字及参数 */
      while (*next && *next != token) {	  // 在字符串结束前找单词分隔符
	 next++;
      }

      /* 如果未结束(是token字符),使tocken变成0 */
      if (*next) {
	 *next++ = 0;	// 将token字符替换为字符串结束符0,做为一个单词的结束,并将字符指针next指向下一个字符
      }
   
      /* 避免argv数组访问越界,参数过多则返回0 */
      if (argc > MAX_ARG_NR) {
	 return -1;
      }
      argc++;
   }
   return argc;
}

```

上有从键盘中读入缓冲区，快捷键原理，分析cmd命令到arg数组。

linux中的cmd命令解析是将路径中的各种参数全部存到arg,argc+1代表有几个参数即 arg是以0开始的下标。

argv是全局变量，为了以后exec的程序可访问参数







Linux的shell或WINDOWS中的cmd中命令分为两大类，一种是外部命令，一种是内部命令。外部命令是指该命令是个存储在文件系统上的外部程序，执行该命令实际上是从文件系统上加载该程序到内存后运行的进程，也就是说外部命令会以进程的方式执行。如ls命令，它通常存储在/bin/ls。而如cd这种实际上都是由bash提供，即内核的shell.c文件中提供，因此称他们为BASH_BUILTNS。





（1）内部命令都以前缀“buildin_”+“命令名”的形式命名，如 cd 命令的函数是 buildin_cd。

 （2）形参均是 argc 和 argv，argc 是参数数组 argv 中参数的个数。 

（3）函数实现是调用同功能的系统调用实现的，如函数 buildin_cd 是调用系统调用 chdir 完成的。

 （4）在进行系统调用前调用函数 make_clear_abs_path 把路径转换为绝对路径。

















##### 让SHELL调用外部命令

Linux中执行命令，是bash先fork一个子进程，然后调用exec去执行命令，更严格的说，是执行外部命令时bash才会fork出子进程并调用exec从磁盘上加载外部命令对应的程序，然后执行该程序，从而实现了外部命令的执行。

```c
 } else {      // 如果是外部命令,需要从磁盘上加载
	 int32_t pid = fork();
	 if (pid) {	   // 父进程
	    int32_t status;
	    int32_t child_pid = wait(&status);          // 此时子进程若没有执行exit,my_shell会被阻塞,不再响应键入的命令
	    if (child_pid == -1) {     // 按理说程序正确的话不会执行到这句,fork出的进程便是shell子进程
	       panic("my_shell: no child\n");
	    }
	    printf("child_pid %d, it's status: %d\n", child_pid, status);
	 } else {	   // 子进程
	    make_clear_abs_path(argv[0], final_path); //得到可执行文件argv[0]的绝对路径加载到final_path，然后将argv[0]重新指向final_path
	    argv[0] = final_path;
	    /* 先判断下文件是否存在 */
	    struct stat file_stat;
	    memset(&file_stat, 0, sizeof(struct stat));
	    if (stat(argv[0], &file_stat) == -1) {
	       printf("my_shell: cannot access %s: No such file or directory\n", argv[0]);
	       exit(-1);
	    } else {
	       execv(argv[0], argv);
	    }
	 }
      }
```

execv是解析磁盘上的elf文件，然后加载到当前进程。









**其实命令也是相当于一个新的进程,由父进程fork的一个子进程，子进程复制父进程的所有当前资源副本（内容相同，地址不同），然后execv（path,argv[]）复制该要运行的进程体并将参数传递给其栈，最后return都会exit将状态给eax,CRT将状态传给父进程，最后由内核清理子进程的PCB**

```c
#include "syscall.h"
#include "stdio.h"
#include "string.h"
int main(int argc, char** argv) {
   if (argc > 2) {
      printf("cat: argument error\n");
      exit(-2);
   }

   if (argc == 1) {
      char buf[512] = {0};
      read(0, buf, 512);
      printf("%s",buf);
      exit(0);
   }

   int buf_size = 1024;
   char abs_path[512] = {0};
   void* buf = malloc(buf_size);
   if (buf == NULL) { 
      printf("cat: malloc memory failed\n");
      return -1;
   }
   if (argv[1][0] != '/') {
      getcwd(abs_path, 512);
      strcat(abs_path, "/");
      strcat(abs_path, argv[1]);
   } else {
      strcpy(abs_path, argv[1]);
   }
   int fd = open(abs_path, O_RDONLY);
   if (fd == -1) { 
      printf("cat: open: open %s failed\n", argv[1]);
      return -1;
   }
   int read_bytes= 0;
   while (1) {
      read_bytes = read(fd, buf, buf_size);
      if (read_bytes == -1) {
         break;
      }
      write(1, buf, read_bytes);
   }
   free(buf);
   close(fd);
   return 66;
}
```







