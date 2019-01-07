# [C语言函数调用栈(三)](https://www.cnblogs.com/clover-toeic/p/3757091.html)



 

# 6 调用栈实例分析

​     本节通过代码实例分析函数调用过程中栈帧的布局、形成和消亡。

 

## 6.1 栈帧的布局

​     示例代码如下：



[![复制代码](http://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
 1 //StackReg.c
 2 #include <stdio.h>
 3 
 4 //获取函数运行时寄存器%ebp和%esp的值
 5 #define FETCH_SREG(_ebp, _esp)     do{\
 6     asm volatile( \
 7         "movl %%ebp, %0 \n" \
 8         "movl %%esp, %1 \n" \
 9         : "=r" (_ebp), "=r" (_esp) \
10     ); \
11 }while(0)
12 //也可使用gcc扩展register void *pvEbp __asm__ ("%ebp"); register void *pvEsp __asm__ ("%esp");获取，
13 // pvEbp和pvEsp指针变量的值就是FETCH_SREG(_ebp, _esp)中_ebp和_esp的值
14 
15 #define PRINT_ADDR(x)     printf("[%s]: &"#x" = %p\n", __FUNCTION__, &x)
16 #define PRINT_SREG(_ebp, _esp)     do{\
17     printf("[%s]: EBP      = 0x%08x\n", __FUNCTION__, _ebp); \
18     printf("[%s]: ESP      = 0x%08x\n", __FUNCTION__, _esp); \
19     printf("[%s]: (EBP)    = 0x%08x\n", __FUNCTION__, *(int *)_ebp); \
20     printf("[%s]: (EIP)    = 0x%08x\n", __FUNCTION__, *((int *)_ebp + 1)); \
21     printf("[%s]: &"#_esp"  = %p\n", __FUNCTION__, &_esp); \
22     printf("[%s]: &"#_ebp"  = %p\n", __FUNCTION__, &_ebp); \
23 }while(0)
24 
25 void tail(int paraTail){
26     int locTail = 0;
27     int ebpReg, espReg;
28 
29     FETCH_SREG(ebpReg, espReg);
30     PRINT_SREG(ebpReg, espReg);
31     PRINT_ADDR(paraTail);
32     PRINT_ADDR(locTail);
33 }
34 int middle(int paraMid1, int paraMid2, int paraMid3){
35     int ebpReg, espReg;
36     tail(paraMid1);
37 
38     FETCH_SREG(ebpReg, espReg);
39     PRINT_SREG(ebpReg, espReg);
40     PRINT_ADDR(paraMid1);
41     PRINT_ADDR(paraMid2);
42     PRINT_ADDR(paraMid3);
43     return 1;
44 }
45 int main(void){
46     int ebpReg, espReg;
47     int locMain = middle(1, 2, 3);
48 
49     FETCH_SREG(ebpReg, espReg);
50     PRINT_SREG(ebpReg, espReg);
51     PRINT_ADDR(locMain);
52     return 0;
53 }
```

[![复制代码](http://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

​     该程序每个函数都嵌入汇编代码，以获取各函数运行时刻EBP和ESP寄存器的值。每个函数都打印出EBP寄存器所指向内存地址处的值，以及位于其后的函数返回地址。图7给出程序的编译和运行结果。

![img](https://images0.cnblogs.com/i/569008/201405/281428362913446.jpg) 

图7 StackReg运行结果

​     为便于理解输出结果中数据间的关系，将其转化为图8所示。图左还示出栈的增长方向和栈的内存地址。黑色箭头和寄存器名表示当前栈帧，否则用灰色表示。图中表示tail函数内所看到的栈布局，其中完整示出tail和middle函数的栈帧结构，以及main函数的部分。注意，形参1、2、3(常量)不在栈内。

 ![img](https://images0.cnblogs.com/i/569008/201405/281430276813048.jpg)

图8 StackReg栈帧布局

​     通常每个函数都有自己的栈帧。各栈帧中存放前一个调用函数的栈帧基址，通过该地址域将所有主调函数与被调函数的栈帧以链表形式连在一起。函数调用级数越多，占用的栈空间也越大，因此应小心使用递归函数。

 

## 6.2 栈帧的形成

​     为方便讲解，获取StackReg示例程序所对应的汇编代码片段，如图9所示。在汇编代码中，最左列为指令在内存中的地址，栈帧中的返回地址(return address)即指此类地址。最右列为待执行的汇编指令语句，中间列为该指令在代码段中的16进制表示，可见push %ebp指令仅占一个字节(0x55)。每次CPU执行都要先读取%eip寄存器值，然后定位到%eip指向的汇编指令内存地址，读取该指令并执行。读取指令会使%eip寄存器值增加相应指令的长度(字节数)，执行指令后%eip值为下条待执行指令的跳转地址。

 ![img](https://images0.cnblogs.com/i/569008/201405/281434017595647.jpg)

图9 StackReg汇编片段

​     假设程序运行在main刚调用middle函数时，观察栈帧布局如何变化。程序进入middle函数所运行的第一条指令位于内存地址0x804847c处，在运行该指令之前的栈帧结构如图10所示。此时EBP指向main函数栈帧的头部，而ESP所指向的内存中存放程序返回到main函数的指令位置(0x080485c5)。

![img](https://images0.cnblogs.com/i/569008/201405/281435513844523.jpg) 

图10 StackReg运行中栈帧结构-1

​     被调函数在调用后获得程序的控制权，接着需完成3项工作：建立自己的栈帧，为局部变量分配空间，按需保存寄存器%ebx、%esi和%edi的值。

​     内存地址0x804847c～0x804847f的指令用于形成middle函数的栈帧。第一条指令(位于地址0x804847c处，简称<指令804847c>)将主调函数main的栈帧基址保存到栈上(压栈操作)，该地址用于从被调函数堆栈返回到主调函数main中。正是各函数内的这一操作，使得所有栈帧连在一起成为一条链。

​     <指令804847d>将%esp寄存器的值赋值给%ebp寄存器，此时%ebp寄存器中存放当前函数的栈帧基址，以便根据偏移量访问堆栈中的参数或变量。这样便可腾出%esp寄存器以作他用，并在需要时根据%ebp值从当前函数栈顶直接返回栈底。

​     <指令804847f>对%esp进行减操作，即将%esp向低地址处移动40(0x28)个字节，以便在栈上腾出空间来存放局部变量和临时变量。

​     运行完上述三条指令后，middle函数的栈帧就已形成，如图11所示。图中还示出该函数内的局部变量ebpReg和espReg在栈帧中的位置。

![img](https://images0.cnblogs.com/i/569008/201405/281438094941263.jpg) 

图11 StackReg运行中栈帧结构-2

​     随后，将执行middle函数体。执行过程中帧基指针EBP保持不变，通过该指针加偏移量即可访问函数实参、局部变量和临时存储内容。即使middle函数内调用其他函数(如tail)，甚至递归调用middle自身，只要在这些子调用返回时恢复EBP，就可继续用EBP加偏移量的方式访问实参等信息。

​     <指令804848d>和<指令804848f>是middle函数中内嵌的汇编代码，用于获取此时%ebp和%esp寄存器的值。<指令8048491>将%ebp寄存器值放入局部变量ebpReg中，<指令8048494>则将%esp寄存器值放入局部变量espReg中。其中，0xfffffffc(%ebp)等于(%ebp - 4)，表示在帧基指针向低地址偏移四字节的地址处存储的内容(偏移量用补码表示，负值表示向低地址偏移)。

​     <指令8048482>和<指令8048485>将main函数中传递来的第一个变量paraMid1值拷贝到%esp寄存器所指向的内存中，为调用tail函数准备实参。此时栈空间如图12所示。

![img](https://images0.cnblogs.com/i/569008/201405/281439460416634.jpg) 

图12 StackReg运行中栈帧结构-3

​     <指令8048488>调用tail函数，该调用将返回地址(EIP指令指针寄存器的内容)压入栈中，调用该指令后的栈空间如图13所示。压栈的返回地址是0x804848d，从图9中可看出该地址指向middle函数内调用tail函数的后一条指令，当tail函数返回时将从该地址处继续运行程序。调用<指令8048488>也意味着进入tail函数的栈帧，tail函数采用与middle函数相同方式的建立自己的栈帧。前面图8所示正是tail函数建立栈帧时的内存布局。

 ![img](https://images0.cnblogs.com/i/569008/201405/281442118387025.jpg)

图13 StackReg运行中栈帧结构-4

​     通过以上运行时分析，可看到函数调用过程中堆栈扩展与恢复的动态过程。%esp和%ebp两个寄存器之间的赋值时机，正是主调函数和被调函数职责交替之时。也正是该时机的正确，才能保证堆栈的恢复。

 

## 6.3 栈帧的消亡

​     在把程序控制权返还给主调函数前，被调函数若有返回值，则先将其保存在相应寄存器(通常是%eax)中，然后按需恢复%ebx、%esi和%edi寄存器的值，最后从栈里弹出返回地址。

​     下面观察tail函数内进行函数返回时栈空间如何变化。<指令804847a>为leave指令，将%esp寄存器的值设置为%ebp寄存器值并做一次弹栈操作，将弹栈操作的内容放入%ebp寄存器中。该指令的功能等价于"mov %ebp, %esp"加"pop %ebp"，可将tail函数所建立的栈帧清除。该指令执行后的栈布局与图13完全相同。<指令804847b>用于将栈上的返回地址弹出到%eip寄存器中，执行该指令后程序返回到middle函数的0x804848d地址处。该指令执行后的栈结构与图12相同。

 

## 6.4 返回结构体

​     分析以下示例程序：



[![复制代码](http://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
 1 //StackStrt.c
 2 #include <stdio.h>
 3 
 4 typedef struct{
 5     int member1;
 6     int member2;
 7     int member3;
 8 }T_RET_STRT;
 9 
10 //FETCH_SREG/PRINT_SREG/PRINT_ADDR宏定义，略(详见StackReg.c)
11 T_RET_STRT func(int paraFunc){
12     T_RET_STRT locStrtFunc = {.member1=1, .member2=2, .member3=3};
13     int ebpReg, espReg;
14 
15     FETCH_SREG(ebpReg, espReg);
16     PRINT_SREG(ebpReg, espReg);
17     PRINT_ADDR(paraFunc);
18     printf("[%s]: (BelowPara) = 0x%08x\n", __FUNCTION__, *((int *)&paraFunc - 1));
19     PRINT_ADDR(locStrtFunc.member1);
20     PRINT_ADDR(locStrtFunc.member2);
21     PRINT_ADDR(locStrtFunc.member3);
22     return locStrtFunc;
23 }
24 int main(void){
25     int ebpReg, espReg;
26     T_RET_STRT locStrtMain = func(100);
27 
28     FETCH_SREG(ebpReg, espReg);
29     PRINT_SREG(ebpReg, espReg);
30     PRINT_ADDR(locStrtMain.member1);
31     PRINT_ADDR(locStrtMain.member2);
32     PRINT_ADDR(locStrtMain.member3);
33     return 0;
34 }
```

[![复制代码](http://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

​     该示例中，main和func函数内均定义类型为T_RET_STRT的局部变量，且func函数的返回值类型也是T_RET_STRT。变量locStrtMain和locStrtFunc的内存将分配在各自函数的栈帧中，那么func函数的locStrtFunc变量值如何通过函数返回值传递到main函数的locStrtMain变量中？编译该程序并运行以观察结果，如图14所示。图15示出func函数内所看到的栈布局。 

![img](https://images0.cnblogs.com/i/569008/201405/281446299949090.jpg) 

图14 StackStrt运行结果

 ![img](https://images0.cnblogs.com/i/569008/201405/281454393842313.jpg)

图15 StackStrt栈帧布局

​     从图中可看出，main函数调用func函数时除将后者所需的参数压入栈中外，还将局部变量locStrtMain地址也压入栈中；func函数返回时将locStrtFunc变量的值通过该地址直接拷贝到main函数的locStrtMain变量中，从而省去一次通过栈的中转拷贝。

​     删除打印等无关语句后，查看StackStrt.c源文件汇编代码如下图所示(略有删减)：

![img](https://images0.cnblogs.com/i/569008/201405/281456539631898.jpg)

图16 StackStrt汇编片段 

​     <指令804839a>将局部变量locStrtMain结构体在栈中的地址存入%eax寄存器。<指令804839d>将标量参数(100)入栈，因<指令8048397>已预留好存储空间，故此处等效于"pushl $0x64"。<指令8048397>将%eax中保存的结构体地址(&locStrtMain)入栈，此处等效于"pushl %eax"。

​     <指令804835a>将8(%ebp)处所存储的主调函数locStrtMain结构体地址存入%edx寄存器。<指令804835d>至<指令804836b>对被调函数栈内的局部变量locStrtFunc结构体赋值。<指令8048372>至<指令8048380>将locStrtFunc结构体的各个成员变量值依次存入%edx寄存器所指向的内存地址处(&locStrtMain)。<指令8048383>将暂存的%edx寄存器内容存入%eax寄存器，此时%eax内存放主调函数结构体locStrtMain的地址。

​     根据汇编结果，可知func函数被“改编”为以下实现：



[![复制代码](http://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
1 void func(T_RET_STRT *pStrtMain, int paraFunc){
2     T_RET_STRT locStrtFunc = {.member1=1, .member2=2, .member3=3};
3     pStrtMain->member1 = locStrtFunc.member1;
4     pStrtMain->member2 = locStrtFunc.member2;
5     pStrtMain->member3 = locStrtFunc.member3;
6     return; //此句可有可无
7 }
```

[![复制代码](http://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

​     若显式声明结构体指针参数，则可编写更高效的func函数代码：



```
1 void func(T_RET_STRT *pStrtMain, int paraFunc){
2     pStrtMain->member1 = 1;
3     pStrtMain->member2 = 2;
4     pStrtMain->member3 = 3;
5 }
```

​     注意，若T_RET_STRT locStrtMain = func(100)改为func(100)，主调函数栈上仍会预留一个结构体变量的空间，然后将该变量地址存入%eax寄存器。<指令8048397>和<指令804839a>分别变为sub $0x1c, %esp和lea 0xffffffe8(%ebp), %eax。 

​     从以上分析亦知，当函数以结构体或联合体作为返回值时，函数第一个参数存放在栈帧12(%ebp)位置处，而8(%ebp)位置处存放返回值的地址。