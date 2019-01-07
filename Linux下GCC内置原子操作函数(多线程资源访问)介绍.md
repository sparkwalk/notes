## Linux下GCC原子操作介绍

​    在多进程（线程）访问资源时，能够确保所有其他的进程（线程）都不在同一时间内访问相同的资源。原子操作（atomic operation）是不需要synchronized。所谓原子操作是指不会被线程调度机制打断的操作；这种操作一旦开始，就一直运行到结束，中间不会有任何 context switch （切换到另一个线程）。通常所说的原子操作包括对非long和double型的primitive进行赋值，以及返回这两者之外的primitive。

​    原子操作是不可分割的，在执行完毕之前不会被任何其它任务或事件中断。在单处理器系统（UniProcessor）中，能够在单条指令中完成的操作都可以认为是" 原子操作"，因为中断只能发生于指令之间。这也是某些CPU指令系统中引入了test_and_set、test_and_clear等指令用于临界资源互斥的原因。但是，在对称多处理器（Symmetric Multi-Processor）结构中就不同了，由于系统中有多个处理器在独立地运行，即使能在单条指令中完成的操作也有可能受到干扰。我们以decl （递减指令）为例，这是一个典型的"读－改－写"过程，涉及两次内存访问。设想在不同CPU运行的两个进程都在递减某个计数值，可能发生的情况是：

⒈ CPU A(CPU A上所运行的进程，以下同）从内存单元把当前计数值⑵装载进它的寄存器中；

⒉ CPU B从内存单元把当前计数值⑵装载进它的寄存器中。

⒊ CPU A在它的寄存器中将计数值递减为1；

⒋ CPU B在它的寄存器中将计数值递减为1；

⒌ CPU A把修改后的计数值⑴写回内存单元。

⒍ CPU B把修改后的计数值⑴写回内存单元。

​    我们看到，内存里的计数值应该是0，然而它却是1。如果该计数值是一个共享资源的引用计数，每个进程都在递减后把该值与0进行比较，从而确定是否需要释放该共享资源。这时，两个进程都去掉了对该共享资源的引用，但没有一个进程能够释放它--两个进程都推断出：计数值是1，共享资源仍然在被使用。

  原子性不可能由软件单独保证--必须需要硬件的支持，因此是和架构相关的。在x86 平台上，CPU提供了在指令执行期间对总线加锁的手段。CPU芯片上有一条引线#HLOCK pin，如果汇编语言的程序中在一条指令前面加上前缀"LOCK"，经过汇编以后的机器代码就使CPU在执行这条指令的时候把#HLOCK pin的电位拉低，持续到这条指令结束时放开，从而把总线锁住，这样同一总线上别的CPU就暂时不能通过总线访问内存了，保证了这条指令在多处理器环境中的原子性。

 在Linux下c/c++编程中，有两种方式实现多线程访问互斥资源:

1. 互斥锁,pthread_mutex系列函数，能实现多线程资源访问同步，适用范围广，性能较原子操作低。
2. 原子操作，针对单个变量的原子操作，性能高，适用范围窄。
##  Linux下GCC内置原子操作系列函数

  gcc从4.1.2提供了__sync_*系列的built-in函数，用于提供加减和逻辑运算的原子操作。

```
type __sync_fetch_and_add (type *ptr, type value, ...)
type __sync_fetch_and_sub (type *ptr, type value, ...)
type __sync_fetch_and_or (type *ptr, type value, ...)
type __sync_fetch_and_and (type *ptr, type value, ...)
type __sync_fetch_and_xor (type *ptr, type value, ...)
type __sync_fetch_and_nand (type *ptr, type value, ...)
type __sync_add_and_fetch (type *ptr, type value, ...)
type __sync_sub_and_fetch (type *ptr, type value, ...)
type __sync_or_and_fetch (type *ptr, type value, ...)
type __sync_and_and_fetch (type *ptr, type value, ...)
type __sync_xor_and_fetch (type *ptr, type value, ...)
type __sync_nand_and_fetch (type *ptr, type value, ...)
```

这两组函数的区别在于第一组返回更新前的值，第二组返回更新后的值。

type可以是1,2,4或8字节长度的int类型，即：

int8_t / uint8_t

int16_t / uint16_t

int32_t / uint32_t

int64_t / uint64_t

后面的可扩展参数(...)用来指出哪些变量需要memory barrier,因为目前gcc实现的是full barrier（类似于linux kernel 中的mb(),表示这个操作之前的所有内存操作不会被重排序到这个操作之后）,所以可以略掉这个参数。

bool __sync_bool_compare_and_swap (type *ptr, type oldval type newval, ...)

type __sync_val_compare_and_swap (type *ptr, type oldval type newval, ...)

这两个函数提供原子的比较和交换，如果*ptr == oldval,就将newval写入*ptr,

第一个函数在相等并写入的情况下返回true.

第二个函数在返回操作之前的值。

__sync_synchronize (...)

发出一个full barrier.

   关于memory barrier,cpu会对我们的指令进行排序，一般说来会提高程序的效率，但有时候可能造成我们不希望得到的结果，举一个例子，比如我们有一个硬件设备，它有4个寄存器，当你发出一个操作指令的时候，一个寄存器存的是你的操作指令（比如READ），两个寄存器存的是参数（比如是地址和size），最后一个寄存器是控制寄存器，在所有的参数都设置好之后向其发出指令，设备开始读取参数，执行命令，程序可能如下：

   write1(dev.register_size,size);

   write1(dev.register_addr,addr);

   write1(dev.register_cmd,READ);

   write1(dev.register_control,GO);

  如果最后一条write1被换到了前几条语句之前，那么肯定不是我们所期望的，这时候我们可以在最后一条语句之前加入一个memory barrier,强制cpu执行完前面的写入以后再执行最后一条：

   write1(dev.register_size,size);

   write1(dev.register_addr,addr);

   write1(dev.register_cmd,READ);

   __sync_synchronize();

   write1(dev.register_control,GO);

memory barrier有几种类型：

   acquire barrier : 不允许将barrier之后的内存读取指令移到barrier之前（linux kernel中的wmb()）。

   release barrier : 不允许将barrier之前的内存读取指令移到barrier之后 (linux kernel中的rmb())。

   full barrier    : 以上两种barrier的合集(linux kernel中的mb())。

还有两个函数：

type __sync_lock_test_and_set (type *ptr, type value, ...)

  将*ptr设为value并返回*ptr操作之前的值。

void __sync_lock_release (type *ptr, ...)

​    将*ptr置0

  从gcc 4.4起，标准库就提供了atomic C++类，尤其是4.6（支持C++0x标准），作了很好的封装，相关头文件是atomic_base.h, atomic_0.h,

atomic_2.h, atomic.h。

## 示例代码

```
#include <stdio.h>
#include <pthread.h>
#include <stdlib.h>
static int count = 0;
void *test_func(void *arg)
{
        int i=0;
        for(i=0;i<20000;++i){
                __sync_fetch_and_add(&count,1);
        }
        return NULL;
}
int main(int argc, const char *argv[])
{
        pthread_t id[20];
        int i = 0;
        for(i=0;i<20;++i){
                pthread_create(&id[i],NULL,test_func,NULL);
        }
        for(i=0;i<20;++i){
                pthread_join(id[i],NULL);
        }
        printf("%dn",count);
        return 0;
}
```