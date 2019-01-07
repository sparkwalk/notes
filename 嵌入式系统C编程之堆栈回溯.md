 

# 前言

​     在嵌入式系统C语言开发调试过程中，常会遇到各类异常情况。一般可按需添加打印信息，以便观察程序执行流或变量值是否异常。然而，打印操作会占用CPU时间，而且代码中添加过多打印信息时会显得很凌乱。此外，即使出错打印已非常详尽，但仍难以完全预防和处理段违例(Segment Violation)等错误。在没有外部调试器(如gdb server)可用或无法现场调试的情况下，若程序能在突发崩溃时自动输出函数的调用堆栈信息(即堆栈回溯)，那么对于排错将会非常有用。

​     本文主要介绍嵌入式系统C语言编程中，发生异常时的堆栈回溯方法。文中涉及的代码运行环境如下：

![img](https://images0.cnblogs.com/blog/569008/201409/011936570161878.jpg) 

​     本文假定读者已具备函数调用栈、信号处理等方面的知识。相关性文章也可参见：

​     [《C语言函数调用栈(一)》](http://www.cnblogs.com/clover-toeic/p/3755401.html)

​     [《C语言函数调用栈(二)》](http://www.cnblogs.com/clover-toeic/p/3756668.html)

​     [《C语言函数调用栈(三)》](http://www.cnblogs.com/clover-toeic/p/3757091.html)

​     [《嵌入式系统C编程之错误处理》](http://www.cnblogs.com/clover-toeic/p/3919857.html)

 

 

# 一  原理

​     通常，在多级函数调用过程中，处理器会将调用函数指令的下一条地址压入堆栈。通过分析当前栈帧，找到上层函数在堆栈中的栈帧地址，再分析上层函数的栈帧，进而找到再上层函数的栈帧地址……如此回溯直至最顶层函数。这就组成一条函数执行的路径轨迹(调用顺序)。

​     以Intel x86架构为例，由于帧基指针(BP)所指向的内存中存储上一层函数调用时的BP值，而在每层函数调用中都能通过当前BP值向栈底方向偏移得到返回地址。如此递归，可逐层向上找到最顶层函数。

​     在GDB里，使用bt命令可获取函数调用栈。若要通过代码获取当前函数调用栈，可借助glibc库提供的backtrace系列函数。由于不同处理器堆栈布局不同，堆栈回溯由编译器内建函数__buildin_frame_address和__buildin_return_address实现，涉及工具glibc和gcc。若编译器不支持该功能，也可自行实现，其步骤如下(以Intel x86架构为例)：

​     1) 获得当前函数的BP；

​     2) 通过BP偏移获得主调函数的IP(返回地址)；

​     3) 通过当前BP指向的内容，获得主调函数BP地址；

​     4) 循环执行以上步骤直至到达栈底。

​     glibc2.1及以上版本提供backtrace等GNU扩展函数以获取当前线程的函数调用堆栈，其原型声明在头文件<execinfo.h>内。

int **backtrace**(void **buffer, int size);

​     该函数获取当前线程的调用堆栈，并以指针(实为返回地址)列表形式存入参数buffer缓冲区中。参数size指定buffer中可容纳的void*元素数目。该函数返回是实际获取的元素数，且不超过size大小。若返回值小于size，则buffer中保存完整的堆栈信息；若返回值等于size，则堆栈信息可能已被删减(最早的那些栈帧返回地址被丢弃)。

char ** **backtrace_symbols**(void *const *buffer, int size);

​     该函数将backtrace函数获取的信息转换为一个字符串数组。参数buffer应指向backtrace函数获取的地址数组，参数size为该数组中的元素个数(backtrace函数返回值)。

​     该函数返回一个指向字符串数组的指针，数组元素个数与buffer数组相同(即为size)。每个字符串包含一个对应buffer数组元素的可打印描述信息，如函数名、偏移地址和实际的返回地址(16进制)。

​     该函数的返回值指向函数内部通过malloc所申请的动态内存，因此调用者必须使用free函数来释放该内存。若不能为字符串申请足够的内存，则该函数返回NULL。

​     目前，只有在使用ELF二进制格式的程序和库的系统中才能获取函数名和偏移地址。在其他系统中，仅能获取16进制的返回地址。此外，可能需要向链接器传递额外的标志，以支持函数名功能(如在使用GNU ld的系统中，需要传递-rdynamic选项来通知链接器将所有符号添加到动态符号表中)。

void **backtrace_symbols_fd**(void *const *buffer, int size, int fd);

​     该函数与backtrace_symbols函数功能相同，但不向调用者返回字符串数组，而是将结果写入文件描述符为fd的文件中，每条信息字符串对应一行。该函数不会为字符串存储申请动态内存，因此适用于堆内存可能被破坏的情况(此时buffer也应为静态或自动存储空间)。

​     举例如下：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
 1 #include <stdio.h>
 2 #include <stdlib.h>
 3 #include <unistd.h>
 4 #include <execinfo.h>
 5 
 6 static void StackTrace(void){
 7     void *pvTraceBuf[10];
 8     int dwTraceSize = backtrace(pvTraceBuf, 10);
 9     backtrace_symbols_fd(pvTraceBuf, dwTraceSize, STDOUT_FILENO);
10 }
11 
12 void FuncC(void){ StackTrace(); }
13 static void FuncB(void){ FuncC(); }
14 void FuncA(void){ FuncB(); }
15 int main(void){
16     FuncA();
17     return 0;
18 }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

​     编译运行结果如下：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
1 [wangxiaoyuan_@localhost test1]$ gcc -Wall -rdynamic -o StackTrace StackTrace.c
2 [wangxiaoyuan_@localhost test1]$ ./StackTrace
3 ./StackTrace[0x80485f9]
4 ./StackTrace(FuncC+0xb)[0x8048623]
5 ./StackTrace[0x8048630]
6 ./StackTrace(FuncA+0xb)[0x804863d]
7 ./StackTrace(main+0x16)[0x8048655]
8 /lib/libc.so.6(__libc_start_main+0xdc)[0x552e9c]
9 ./StackTrace[0x8048521]
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

​     当若干主调函数中的某个以错误的参数调用给定函数时，通过在该函数内检查参数并调用StackTrace()函数，即可方便地定位出错的主调函数。

​     使用backtrace系列函数获取堆栈回溯信息时，需要注意以下几点：

​     1) 某些编译器优化可能对获取有效的调用堆栈造成干扰。

​     若忽略帧基指针(-fomit-frame-pointer)，回溯时将无法正确解析堆栈内容。优化级别非0时(如-O2)可能改变函数调用关系；尾调用(Tail-call)优化会替换栈帧内容，这些也会影响回溯结果。

​     2) 内联函数和宏定义没有栈帧结构。

​     3) 静态函数名无法被内部解析，因其无法被动态链接访问。此时可使用外部工具addr2line解析。

​     4) 若内存垃圾导致堆栈自身被破坏，则无法进行回溯。

​     若自行实现堆栈回溯功能，可调用dladdr()函数来解析返回地址所对应的文件名和函数名等信息。

#include <dlfcn.h>int **dladdr**(void *addr, Dl_info *info);

​     该函数出错时(共享库libdl.so目标文件段中不存在该地址)返回0，成功时返回非0值。

​     Dl_info结构定义如下：

```
1 typedef struct{
2     const char *dli_fname;  /* Filename of defining object */
3     void *dli_fbase;        /* Load address of that object */
4     const char *dli_sname;  /* Name of nearest lower symbol */
5     void *dli_saddr;        /* Exact value of nearest symbol */
6 }Dl_info;
```

​     使用dladdr()函数时，需加上-rdynamic编译选项和-ldl链接选项。

​     更进一步，可将堆栈回溯置于信号处理程序中。这样，当程序突然崩溃时，当前进程接收到内核发送的信号后，在信号处理程序中自动输出进程的执行信息、当前寄存器内容及函数调用关系等。

​     通常使用sigaction()函数检查或修改与指定信号相关联的处理动作(或同时执行这两种操作)：

#include <signal.h>int **sigaction**( int signo, const struct sigaction *restrict act, struct sigaction *restrict oact);

​     该函数成功时返回0，否则返回-1并设置errno值。参数signo为待检测或修改其具体动作的信号编号。若act指针非空，则修改其动作；若oact指针非空，则系统经由oact指针返回该信号的上个动作。sigaction结构的sa_flags字段指定对信号进行处理的各个选项。当设置为SA_SIGINFO标志时，表示信号附带的信息可传递到信号处理函数中。此时，应按下列方式调用信号处理程序：

void **handler**(int signo, siginfo_t *info, void *context);

​     siginfo_t结构包含信号产生原因的有关信息，需针对不同信号选取有意义的属性。其中，si_signo(信号编号)、si_errno(errno值)和si_code(信号产生原因)定义针对所有信号。其余属性只有部分信息对特定信号有用。例如，si_addr指示触发故障的内存地址(尽管该地址可能并不准确)，仅对SIGILL、SIGFPE、SIGSEGV和SIGBUS 信号有意义。si_errno字段包含错误编号，对应于引发信号产生的条件，并由实现定义(Linux中通常不使用该属性)。

​     信号处理程序的context参数是无类型指针，可被强制转换为ucontext_t结构，用于标识信号产生时的进程上下文(如CPU寄存器)。该结构定义在头文件<ucontext.h>内，且包含mcontext_t类型的uc_mcontext字段(该字段保存特定于机器的寄存器上下文)。

​     注意，即使指定信号处理函数，若不设置SA_SIGINFO标志，信号处理函数同样不能得到信号传递过来的附加信息(info和context)，在信号处理函数中访问这些信息都将导致段错误。

 

 

# 二  实现

​     本节将实现基于信号处理的用户态进程堆栈回溯功能。该实现假定未忽略帧基指针。

​     注意，若只需向上回溯一层函数，如查看某函数被哪些函数直接调用，则可对其进行简单封装。假定被调函数名为FuncTraced，可将其声明和定义中的名称改为FuncTraced1，然后封装名为FuncTraced的宏。该宏内部输出定位信息并调用FuncTraced1()函数，如：

```
1 extern void FuncTraced1(void);
2 #define FuncTraced() do{ \
3     printf("[%s<%d>]Call FuncTraced!\n", __FILE__, __LINE__); \
4     FuncTraced1(); \
5 }while(0)
```

​     示例中原FuncTraced()函数无返回值，若有则封装方式略有不同。

## 2.1 数据定义

​     定义如下宏：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
 1 #ifndef __i386
 2     #warning "Possibly Non-x86 Platform!"
 3 #endif
 4 
 5 #if defined(REG_RIP)
 6     #define REG_IP     REG_RIP   //指令指针(保存返回地址)
 7     #define REG_BP     REG_RBP   //帧基指针
 8     #define REG_FMT    "%016lx"
 9 #elif defined(REG_EIP)
10     #define REG_IP     REG_EIP
11     #define REG_BP     REG_EBP
12     #define REG_FMT    "%08x"
13 #else
14     #warning "Neither REG_RIP nor REG_EIP is defined!"
15     #define REG_FMT    "%08x" 
16 #endif
17 
18 #define BTR_FILE_LEN    512   //保存堆栈回溯结果的文件路径最大长度
19 #ifndef BTR_FILE        //保存堆栈回溯结果的基本文件名
20     #define BTR_FILE         "btr" 
21 #endif
22 #ifndef BTR_FILE_PATH   //保存堆栈回溯结果的文件路径(默认为当前路径)
23     #define BTR_FILE_PATH    "."  //"..//var//tmp"
24 #endif
25 
26 #ifndef MAX_BTR_LEVEL   //函数回溯的最大层数
27     #define MAX_BTR_LEVEL    20
28 #endif
29 
30 //用户调用SHOW_STACK宏可触发堆栈回溯
31 #ifndef BTR_SIG   //触发堆栈回溯的用户信号
32     #define BTR_SIG          SIGUSR1
33 #endif
34 #define SHOW_STACK()     do{raise(BTR_SIG);}while(0)
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

​     其中，REG_IP、REG_BP分别为x86处理器的指令指针和帧基指针寄存器编号，REG_FMT宏指定寄存器内容的输出格式。BTR_FILE等文件相关的宏指定保存堆栈回溯结果时文件路径和名称。当程序运行于嵌入式单板时，当前路径可能没有写入权限，此时用户可自定义BTR_FILE_PATH宏。

​     定义如下全局变量：

```
1 static FILE *gpStraceFd = NULL;  //输出文件描述符(置为stderr时输出到终端，否则将输出存入文件)
2 typedef VOID (*SignalHandleFunc)(INT32S dwSignal);
3 static SignalHandleFunc gfpCustSigHandler = NULL; //用户自定义的信号处理函数指针
```

## 2.2 函数接口

​     首先定义一组私有函数。这些内部使用的函数已尽可能保证参数安全性，故省去参数校验处理。

​     SpecifyStraceOutput()函数指定堆栈回溯结果的输出方式：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
 1 /******************************************************************************
 2 * 函数名称:  SpecifyStraceOutput
 3 * 功能说明:  指定回溯结果输出方式
 4 ******************************************************************************/
 5 static FILE *SpecifyStraceOutput(VOID)
 6 {
 7 #ifdef __BTR_TO_FILE
 8     time_t tTime;
 9     CHAR szFileName[BTR_FILE_LEN];
10     szFileName[0] = '\0';
11     if(time(&tTime) != -1)
12     {
13         struct tm *ptTime = localtime(&tTime);
14         snprintf(szFileName, sizeof(szFileName), "%s/[%d]%d%02d%02d_%02d%02d%02d.%s",
15                  BTR_FILE_PATH, getpid(), (ptTime->tm_year+1900), (ptTime->tm_mon+1),
16                  ptTime->tm_mday, ptTime->tm_hour, ptTime->tm_min, ptTime->tm_sec, BTR_FILE);
17     }
18     else
19     {
20         snprintf(szFileName, sizeof(szFileName), "%s/%s", BTR_FILE_PATH, BTR_FILE);
21     }
22 
23     FILE *pFile = fopen(szFileName, "w+");
24     if(NULL == pFile)
25     {
26         fprintf(stderr, "Cannot open File '%s'(%s)\n!", szFileName, strerror(errno));
27         return -1;
28     }
29     return pFile;
30 #else
31     return stderr;
32 #endif
33 }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

​     当__BTR_TO_FILE编译选项打开时，堆栈回溯结果输出到指定目录下的文件内。若成功获取当前时间，则该文件名为"[进程号]年月日_时分秒.btr"，否则名为"btr"。当__BTR_TO_FILE编译选项关闭时，堆栈回溯结果直接输出到终端设备屏幕上。

​     注意，SpecifyStraceOutput()函数返回的文件描述符类型为FILE*。标准流stdin/stdout/stderr均为该类型，用于带缓冲的高级I/O函数(如fread/fwrite/fclose等)；而STDIN_FILENO/ STDOUT_FILENO/STDERR_FILENO的类型为int，用于低级I/O调用(如read/write/close等)。

​     信号处理函数SigHandler()依次输出接收到的信号信息、堆栈寄存器内容及堆栈回溯信息：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
 1 /******************************************************************************
 2 * 函数名称:  SigHandler
 3 * 功能说明:  信号处理函数
 4 * 输入参数:  INT32S dwSigNo      :信号名
 5             siginfo_t *tSigInfo :信号产生原因等信息
 6             VOID *pvContext     :信号传递时的进程上下文
 7 * 输出参数:  NA
 8 * 返 回 值:  VOID
 9 ******************************************************************************/
10 static VOID SigHandler(INT32S dwSigNo, siginfo_t *tSigInfo, VOID *pvContext)
11 {
12     fprintf(gpStraceFd, "\nStart of Stack Trace>>>>>>>>>>>>>>>>>>>>>>>>>>\n");
13 
14     fprintf(gpStraceFd, "Process (%d) receive signal %d\n", getpid(), dwSigNo);
15 
16     fprintf(gpStraceFd, "<Signal Information>:\n" );
17     fprintf(gpStraceFd, "\tSigNo:     %-2d(%s)\n", tSigInfo->si_signo, OmciStrSigNo(tSigInfo->si_signo)); //strsignal(dwSigNo)
18     fprintf(gpStraceFd, "\tErrNo:     %-2d(%s)\n", tSigInfo->si_errno, strerror(tSigInfo->si_errno));
19     fprintf(gpStraceFd, "\tSigCode:   %-2d\n", tSigInfo->si_code);
20     fprintf(gpStraceFd, "\tRaised at: %p[Unreliable]\n", tSigInfo->si_addr);
21 
22     fprintf(gpStraceFd, "<Register Content>: \n\t" );
23     INT32U dwIdx = 0;
24     ucontext_t *ptContext = (ucontext_t*)pvContext;
25     for(dwIdx = 0; dwIdx < NGREG; dwIdx++)
26     {
27         fprintf(gpStraceFd, REG_FMT" ", ptContext->uc_mcontext.gregs[dwIdx]);
28         if(0 == ((dwIdx+1)%4)) //每行输出4个寄存器值
29             fprintf(gpStraceFd, "\n\t");
30     }
31     fprintf(gpStraceFd, "\n");
32 
33 #if defined(REG_RIP) || defined(REG_EIP)
34     dwIdx = 0;
35     VOID *pvIp = (VOID*)ptContext->uc_mcontext.gregs[REG_IP];
36     VOID **ppvBp = (VOID**)ptContext->uc_mcontext.gregs[REG_BP];
37     fprintf(gpStraceFd, "<Stack Trace(Customized)>:\n");
38     while(ppvBp != &pvIp)
39     {
40         Dl_info tDlInfo;
41         if(!dladdr(pvIp, &tDlInfo))
42             break;
43         fprintf(gpStraceFd, "\t[%2d] (%s) [0x%08x] (%s)+0x%02x\n", ++dwIdx,
44                 tDlInfo.dli_fname, (INT32U)pvIp,
45                 (tDlInfo.dli_sname != NULL) ? tDlInfo.dli_sname : "<STATIC>",
46                 ((INT32U)pvIp - (INT32U)tDlInfo.dli_saddr));
47 
48         if((NULL == ppvBp) || (tDlInfo.dli_sname && !strcmp(tDlInfo.dli_sname, "main")))
49             break;
50         pvIp = ppvBp[1];          //帧基指针向高地址偏移1个单位(4字节)为返回地址
51         ppvBp = (VOID**)(*ppvBp); //帧基指针所指向的空间存放主调函数栈帧的帧基指针
52     }
53 #else
54     fprintf(gpStraceFd, "<Stack Trace(Standard)>:\n");
55 
56     VOID *pvTraceBuf[MAX_BTR_LEVEL];
57     INT32U dwTraceSize = backtrace(pvTraceBuf, MAX_BTR_LEVEL);
58     CHAR **ppTraceInfos = backtrace_symbols(pvTraceBuf, dwTraceSize);
59     if(!ppTraceInfos || !(*ppTraceInfos))
60         exit(EXIT_FAILURE);
61 
62     for(dwIdx = 0; dwIdx < dwTraceSize; dwIdx++)
63         fprintf(gpStraceFd, "\t%s\n", ppTraceInfos[dwIdx]);
64 
65     free(ppTraceInfos);
66 #endif
67 
68     fprintf(gpStraceFd, "End of Stack Trace<<<<<<<<<<<<<<<<<<<<<<<<<<<<\n\n");
69 
70     if(gfpCustSigHandler != NULL)
71         gfpCustSigHandler(dwSigNo);
72 
73     exit(EXIT_FAILURE);
74 }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

​     其中，si_signo可由入参dwSigNo代替，si_errno句也可省略。因为输出格式需要使用制表符('\t')，故直接使用backtrace_symbols()函数。若更侧重安全性，则可换用backtrace_symbols_fd()函数。

​     若REG_RIP或REG_EIP宏已定义，则根据x86寄存器编号获取主调函数的指令指针IP和帧基指针BP，然后回溯栈帧并自定义输出信息。否则，调用backtrace()相关函数输出回溯信息。

​     注意，REG_XXP等宏的定义位于头文件<ucontext.h>内。若要使用这些宏，需先定义_GNU_SOURCE宏。该宏位于头文件<features.h>(被<ucontext.h>所包含)内，用于控制诸如_ISOC99_SOURCE、_POSIX_SOURCE等功能测试宏，指示是否包含对应标准的特性。_GNU_SOURCE宏必须在所有标准头文件之前包含，即定义在源文件首行，或指定为编译选项。定义该宏后，可使用很多非标准的GNU/ Linux扩展函数。

​     堆栈回溯时静态函数名不可见，因此自定义输出中将其显示为<STATIC>。此外，直接输出时文件描述符为stderr(不带缓冲)，若改为stdout(行缓冲)则"\t%s\n"格式控制将不能正常显示。

​     进程调用exit()函数退出时，内核将关闭进程中已打开的所有文件描述符。因此，SigHandler()函数中未显式调用fclose(gpStraceFd)。

​     读者也可根据栈帧的布局(入参向低地址偏移依次为返回地址和帧基指针)，自行获取IP/BP指针。如：

```
1 VOID **GetEbp(INT32U dwDummy)
2 {
3     VOID **ebp = (VOID **)&dwDummy - 2;
4     return (*ebp);
5 }
```

​     则SigHandler()函数中对ppvBp的赋值可改为：

```
1 VOID **ppvBp = getEbp(dwIdx); //或
2 VOID **ppvBp = (VOID **)&dwSigNo - 2;
```

​     注意，此时获得的寄存器指针指向SigHandler()函数栈帧，while循环内应先执行pvIp = ppvBp[1]再解析地址。

​     OmciStrSigNo()函数基于NameParser来解析信号名：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
 1 #define NAME_MAP_ENTRY(name)    {name, #name}
 2 static T_NAME_PARSER gSigNameMap[] = {
 3     NAME_MAP_ENTRY(SIGHUP),
 4     NAME_MAP_ENTRY(SIGINT),
 5     NAME_MAP_ENTRY(SIGQUIT),
 6     NAME_MAP_ENTRY(SIGILL),
 7     NAME_MAP_ENTRY(SIGTRAP),
 8     NAME_MAP_ENTRY(SIGABRT), //SIGABRT(ANSI) = SIGIOT(4.2 BSD)
 9     NAME_MAP_ENTRY(SIGBUS),
10     NAME_MAP_ENTRY(SIGFPE),
11     NAME_MAP_ENTRY(SIGKILL),
12     NAME_MAP_ENTRY(SIGUSR1),
13     NAME_MAP_ENTRY(SIGSEGV),
14     NAME_MAP_ENTRY(SIGUSR2),
15     NAME_MAP_ENTRY(SIGPIPE),
16     NAME_MAP_ENTRY(SIGALRM),
17     NAME_MAP_ENTRY(SIGTERM),
18     NAME_MAP_ENTRY(SIGSTKFLT),
19     NAME_MAP_ENTRY(SIGCHLD),  //SIGCHLD(POSIX) = SIGCLD(System V)
20     NAME_MAP_ENTRY(SIGCONT),
21     NAME_MAP_ENTRY(SIGSTOP),
22     NAME_MAP_ENTRY(SIGTSTP),
23     NAME_MAP_ENTRY(SIGTTIN),
24     NAME_MAP_ENTRY(SIGTTOU),
25     NAME_MAP_ENTRY(SIGURG),
26     NAME_MAP_ENTRY(SIGXCPU),
27     NAME_MAP_ENTRY(SIGXFSZ),
28     NAME_MAP_ENTRY(SIGVTALRM),
29     NAME_MAP_ENTRY(SIGPROF),
30     NAME_MAP_ENTRY(SIGWINCH),
31     NAME_MAP_ENTRY(SIGIO),     //SIGIO(4.2 BSD) = SIGPOLL(System V)
32     NAME_MAP_ENTRY(SIGPWR),
33     NAME_MAP_ENTRY(SIGSYS)
34 };
35 //信号值字符串化
36 CHAR *OmciStrSigNo(INT32S dwSigNo)
37 {
38     return NameParser(gSigNameMap, ARRAY_SIZE(gSigNameMap), dwSigNo, "UnkownSigNo");
39 }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

​     NameParser()函数实现参见[《C语言表驱动法编程实践》](http://www.cnblogs.com/clover-toeic/p/3730362.html)一文，读者也可自行实现解析函数。

​     若不想依赖函数解析信号名，也可使用如下宏定义：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
 1 #define SIG_NAME(eSigNo) \
 2     ((eSigNo) == SIGHUP     ?    "SIGHUP"      : \
 3     ((eSigNo) == SIGINT     ?    "SIGINT"      : \
 4     ((eSigNo) == SIGQUIT    ?    "SIGQUIT"     : \
 5     ((eSigNo) == SIGILL     ?    "SIGILL"      : \
 6     ((eSigNo) == SIGTRAP    ?    "SIGTRAP"     : \
 7     ((eSigNo) == SIGABRT    ?    "SIGABRT(ANSI)/SIGIOT(4.2 BSD)"     : \
 8     ((eSigNo) == SIGBUS     ?    "SIGBUS"      : \
 9     ((eSigNo) == SIGFPE     ?    "SIGFPE"      : \
10     ((eSigNo) == SIGKILL    ?    "SIGKILL"     : \
11     ((eSigNo) == SIGUSR1    ?    "SIGUSR1"     : \
12     ((eSigNo) == SIGSEGV    ?    "SIGSEGV"     : \
13     ((eSigNo) == SIGUSR2    ?    "SIGUSR2"     : \
14     ((eSigNo) == SIGPIPE    ?    "SIGPIPE"     : \
15     ((eSigNo) == SIGALRM    ?    "SIGALRM"     : \
16     ((eSigNo) == SIGTERM    ?    "SIGTERM"     : \
17     ((eSigNo) == SIGSTKFLT  ?    "SIGSTKFLT"   : \
18     ((eSigNo) == SIGCHLD    ?    "SIGCHLD(POSIX)/SIGCLD(System V)"   : \
19     ((eSigNo) == SIGCONT    ?    "SIGCONT"     : \
20     ((eSigNo) == SIGSTOP    ?    "SIGSTOP"     : \
21     ((eSigNo) == SIGTSTP    ?    "SIGTSTP"     : \
22     ((eSigNo) == SIGTTIN    ?    "SIGTTIN"     : \
23     ((eSigNo) == SIGTTOU    ?    "SIGTTOU"     : \
24     ((eSigNo) == SIGURG     ?    "SIGURG"      : \
25     ((eSigNo) == SIGXCPU    ?    "SIGXCPU"     : \
26     ((eSigNo) == SIGXFSZ    ?    "SIGXFSZ"     : \
27     ((eSigNo) == SIGVTALRM  ?    "SIGVTALRM"   : \
28     ((eSigNo) == SIGPROF    ?    "SIGPROF"     : \
29     ((eSigNo) == SIGWINCH   ?    "SIGWINCH"    : \
30     ((eSigNo) == SIGIO      ?    "SIGIO(4.2 BSD)/SIGPOLL(System V)"  : \
31     ((eSigNo) == SIGPWR     ?    "SIGPWR"      : \
32     ((eSigNo) == SIGSYS     ?    "SIGSYS"      : \
33      "Unknown" )))))))))))))))))))))))))))))))
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

​     InstallFaultTrap()为程序异常时安装的信号捕获函数：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
 1 /******************************************************************************
 2 * 函数名称:  InstallFaultTrap
 3 * 功能说明:  安装出错时的信号捕获函数
 4 * 输入参数:  SignalHandleFunc fpCustSigHandler :用户自定义的信号处理函数
 5 * 输出参数:  NA
 6 * 返 回 值:  INT32S
 7 ******************************************************************************/
 8 static INT32S InstallFaultTrap(SignalHandleFunc fpCustSigHandler)
 9 {
10     gfpCustSigHandler = fpCustSigHandler;
11 
12     struct sigaction tSigAction;
13     memset(&tSigAction, 0, sizeof(tSigAction));
14     tSigAction.sa_sigaction = SigHandler;
15     sigemptyset(&tSigAction.sa_mask);
16     tSigAction.sa_flags = SA_SIGINFO;
17 
18     //检查可能导致进程终止的信号
19     INT32S dwRet = 0;
20     if((dwRet = sigaction(SIGSEGV, &tSigAction, NULL)) < 0)
21         fprintf(stderr, "[%s]Sigaction failed for SIGSEGV(%d, %s)!\n", FUNC_NAME, errno, strerror(errno));
22 
23     if((dwRet = sigaction(SIGQUIT, &tSigAction, NULL)) < 0)
24         fprintf(stderr, "[%s]Sigaction failed for SIGQUIT(%d, %s)!\n", FUNC_NAME, errno, strerror(errno));
25 
26     if((dwRet = sigaction(SIGILL, &tSigAction, NULL)) < 0)
27         fprintf(stderr, "[%s]Sigaction failed for SIGILL(%d, %s)!\n", FUNC_NAME, errno, strerror(errno));
28 
29     if((dwRet = sigaction(SIGTRAP, &tSigAction, NULL)) < 0)
30         fprintf(stderr, "[%s]Sigaction failed for SIGTRAP(%d, %s)!\n", FUNC_NAME, errno, strerror(errno));
31 
32     if((dwRet = sigaction(SIGABRT, &tSigAction, NULL)) < 0)
33         fprintf(stderr, "[%s]Sigaction failed for SIGABRT(%d, %s)!\n", FUNC_NAME, errno, strerror(errno));
34 
35     if((dwRet = sigaction(SIGFPE, &tSigAction, NULL)) < 0)
36         fprintf(stderr, "[%s]Sigaction failed for SIGFPE(%d, %s)!\n", FUNC_NAME, errno, strerror(errno));
37 
38     if((dwRet = sigaction(SIGBUS, &tSigAction, NULL)) < 0)
39         fprintf(stderr, "[%s]Sigaction failed for SIGBUS(%d, %s)!\n", FUNC_NAME, errno, strerror(errno));
40 
41     if((dwRet = sigaction(SIGXFSZ, &tSigAction, NULL)) < 0)
42         fprintf(stderr, "[%s]Sigaction failed for SIGXFSZ(%d, %s)!\n", FUNC_NAME, errno, strerror(errno));
43 
44     if((dwRet = sigaction(SIGXCPU, &tSigAction, NULL)) < 0)
45         fprintf(stderr, "[%s]Sigaction failed for SIGXCPU(%d, %s)!\n", FUNC_NAME, errno, strerror(errno));
46 
47     if((dwRet = sigaction(SIGSYS, &tSigAction, NULL)) < 0)
48         fprintf(stderr, "[%s]Sigaction failed for SIGSYS(%d, %s)!\n", FUNC_NAME, errno, strerror(errno));
49 
50     if((dwRet = sigaction(BTR_SIG, &tSigAction, NULL)) < 0)
51         fprintf(stderr, "[%s]Sigaction failed for %s(%d, %s)!\n", FUNC_NAME,
52                 OmciStrSigNo(BTR_SIG), errno, strerror(errno));
53 
54     return dwRet;
55 }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

​     用户可通过fpCustSigHandler回调函数额外地输出特定的自定义信息。

​     通常，并不期望用户显式地初始化堆栈回溯功能。因此，提供__BTR_AUTO_INIT编译选项以支持自动初始化(AutoInitBacktrace)：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
 1 /******************************************************************************
 2 * 函数名称:  AutoInitBacktrace
 3 * 功能说明:  自动初始化堆栈回溯功能
 4 * 输入参数:  VOID
 5 * 输出参数:  NA
 6 * 返 回 值:  INT32S
 7 * 注意事项:  该函数在main()函数之前执行，无需用户显式调用
 8 ******************************************************************************/
 9 #ifdef __BTR_AUTO_INIT
10 static VOID __attribute((constructor)) AutoInitBacktrace(VOID)
11 {
12     gpStraceFd = SpecifyStraceOutput();
13     InstallFaultTrap(NULL);
14 }
15 #endif
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

​     其中，声明为gcc(constructor)属性的函数将在main()函数之前被执行，而声明为gcc(destructor)属性的函数则在_after_ main()退出时执行。

​     若用户想额外输出自定义信息，则需要显式调用MannInitBacktrace()函数进行手工初始化。该函数调用时可指定fpCustSigHandler回调函数：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
 1 /******************************************************************************
 2 * 函数名称:  MannInitBacktrace
 3 * 功能说明:  手工初始化堆栈回溯功能
 4 * 输入参数:  SignalHandleFunc fpCustSigHandler :用户自定义的信号处理函数
 5 * 输出参数:  NA
 6 * 返 回 值:  VOID
 7 * 注意事项:  fpCustSigHandler符合signal()函数原型，用户可借此额外地输出
 8             特定的自定义信息
 9 ******************************************************************************/
10 VOID MannInitBacktrace(SignalHandleFunc fpCustSigHandler)
11 {
12     gpStraceFd = SpecifyStraceOutput();
13     InstallFaultTrap(fpCustSigHandler);
14 }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

 

 

# 三  测试

​     本节将对上文实现的用户态进程堆栈回溯功能进行测试。测试函数如下：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
 1 VOID Func1(VOID){
 2     SHOW_STACK();
 3     return;
 4 }
 5 VOID Func2(VOID){
 6     Func1();
 7     printf("%s\n", 0x123);
 8     return;
 9 }
10 VOID BtrTest(VOID){
11     Func2();
12     printf("%d\n", 5/0);
13     return;
14 }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

​     指定的编译选项为：

```
1 CFLAGS += -D__BTR_AUTO_INIT -rdynamic –ldl #-D__BTR_TO_FILE
2 CFLAGS += -DMAX_BTR_LEVEL=10
3 CFLAGS += -fno-omit-frame-pointer
```

​     执行结果如下：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
 1 Start of Stack Trace>>>>>>>>>>>>>>>>>>>>>>>>>>
 2 Process (18390) receive signal 10
 3 <Signal Information>:
 4         SigNo:     10(SIGUSR1)
 5         ErrNo:     0 (Success)
 6         SigCode:   -6
 7         Raised at: 0x47d6[Unreliable]
 8 <Register Content>: 
 9         00000033 00000000 0000007b 0000007b 
10         006c8ff4 00535ca0 bfb62228 bfb6221c 
11         000047d6 0000000a 000047d6 00000000 
12         00000000 00000000 00480402 00000073 
13         00000202 bfb6221c 0000007b 
14 <Stack Trace(Standard)>:
15         ./OmciExec [0x804a770]
16         [0x480440]
17         ./OmciExec(Func1+0x12) [0x804ad4e]
18         ./OmciExec(Func2+0xb) [0x804ad5b]
19         ./OmciExec(BtrTest+0xb) [0x804ad7c]
20         ./OmciExec(main+0x16) [0x804eec0]
21         /lib/libc.so.6(__libc_start_main+0xdc) [0x552e9c]
22         ./OmciExec [0x8049f31]
23 End of Stack Trace<<<<<<<<<<<<<<<<<<<<<<<<<<<<
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

​     若注释掉Func1()函数中的SHOW_STACK()语句，则执行结果如下：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
 1 Start of Stack Trace>>>>>>>>>>>>>>>>>>>>>>>>>>
 2 Process (18429) receive signal 11
 3 <Signal Information>:
 4         SigNo:     11(SIGSEGV)
 5         ErrNo:     0 (Success)
 6         SigCode:   1 
 7         Raised at: 0x123[Unreliable]
 8 <Register Content>: 
 9         00000033 00000000 0000007b 0000007b 
10         00000123 bf9a5114 bf9a50ec bf9a4acc 
11         0067eff4 00579999 00000003 00000123 
12         0000000e 00000004 005ad1ab 00000073 
13         00010206 bf9a4acc 0000007b 
14 <Stack Trace(Standard)>:
15         ./OmciExec [0x804a740]
16         [0xedc440]
17         /lib/libc.so.6(_IO_printf+0x33) [0x582e83]
18         ./OmciExec(Func2+0x1f) [0x804ad30]
19         ./OmciExec(BtrTest+0xb) [0x804ad3d]
20         ./OmciExec(main+0x16) [0x804ee80]
21         /lib/libc.so.6(__libc_start_main+0xdc) [0x552e9c]
22         ./OmciExec [0x8049f01]
23 End of Stack Trace<<<<<<<<<<<<<<<<<<<<<<<<<<<<
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

​     若指定编译选项为：

```
1 CFLAGS += -D__BTR_AUTO_INIT -rdynamic -ldl
2 CFLAGS += -D_GNU_SOURCE
3 CFLAGS += -fno-omit-frame-pointer
```

​     则部分执行结果如下：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
 1 <Register Content>: 
 2         00000033 00000000 0000007b 0000007b 
 3         00000123 bfbe8694 bfbe866c bfbe804c 
 4         0067eff4 00579999 00000003 00000123 
 5         0000000e 00000004 005ad1ab 00000073 
 6         00010206 bfbe804c 0000007b 
 7 <Stack Trace(Customized)>:
 8         [ 1] (/lib/libc.so.6) [0x005ad1ab] (strlen)+0x0b
 9         [ 2] (/lib/libc.so.6) [0x00582e83] (_IO_printf)+0x33
10         [ 3] (./OmciExec) [0x0804adfb] (Func2)+0x1f
11         [ 4] (./OmciExec) [0x0804ae08] (BtrTest)+0x0b
12         [ 5] (./OmciExec) [0x0804f154] (main)+0x2a
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

 

 

# 四  参考

backtrace系列函数用法参考<http://www.kernel.org/doc/man-pages/online/pages/man3/backtrace.3.html>

sigaction函数用法参考<http://man7.org/linux/man-pages/man2/sigaction.2.html> 