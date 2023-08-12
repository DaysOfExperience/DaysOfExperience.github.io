# 

# 线程概念

- 线程在进程内部执行，是OS调度的基本单位。
- 线程是进程的一个执行分支，是在进程内部运行的一个执行流。
- 在一个程序里的一个执行路线就叫做线程（thread）。更准确的定义是：线程是“一个进程内部的控制序列”。
- 线程（英语：thread）是操作系统能够进行运算调度的最小单位。它被包含在进程之中，是进程中的实际运作单位。一条线程指的是进程中一个单一顺序的控制流，一个进程中可以并发多个线程，每条线程并行执行不同的任务。

> 历史上，早期，操作系统中是没有线程的，也就是只有进程这个概念，一个进程内只有一个执行流。60年代，在OS中能拥有资源和独立运行的基本单位是进程，然而随着计算机技术的发展，进程出现了很多弊端，一是由于进程是资源拥有者，创建、撤消与切换存在较大的时空开销，因此需要引入轻型进程；二是由于对称多处理机（SMP）出现，可以满足多个运行单位，而多个进程并行开销过大。
>
> 因此在80年代，出现了能独立运行的基本单位——线程（Threads）。
>
> 线程有着自己的特点，比如执行粒度更细，更轻量化，调度切换的开销更小等等，而不同操作系统设计线程时有着不同的方案，Windows对于线程，设立了全新的数据结构，进程和线程划分的很清晰，这是比较复杂的。而因为线程创建，执行，切换，销毁等等很多行为都和进程有着很大的相似性，因此Linux采用了**用进程模拟线程的设计方案**（实现了进程内核代码的复用），这样的设计方案虽然没有为线程设计全新的数据结构，但是最终设计出的“轻量级进程”依旧符合线程的要求。

![img](https://img-blog.csdnimg.cn/img_convert/6acafb3455a159e9e849f1291225d432.png)

上图为学习线程之前，进程加载运行的示意图，每个进程都有一个task_struct（Linux），即进程PCB。

# Linux线程的原理

![img](https://img-blog.csdnimg.cn/img_convert/62afe8000059c59fb86057d05e11ba0b.png)**Linux线程原理：OS内，如果我们创建“进程”时，不创建新的地址空间，用户级页表，不进行IO将程序的代码和数据加载到内存**
**只创建task_struct，让这个新的PCB指向旧的PCB（创建此新线程的主线程)指向的地址空间mm_struct，再通过一定的技术手段，将当前进程的资源合理划分给不同的task_struct，此时，这里的每一个task_struct，就称为一个线程。**

- 线程在进程内部执行，指的是线程在进程的地址空间内运行。
- 每个进程内至少有一个执行线程（一个执行流）
- 在Linux系统中，在CPU眼中，看到的PCB都要比传统的进程的PCB更加轻量化，因此将Linux下的进程统一称之为轻量级进程。（多执行流时，PCB task_struct占用整个进程的一部分资源，当然轻量化）
- 之前一个进程，代码执行流程一定是按顺序执行的，多线程之后，就可以进行资源划分，所有线程共享一个地址空间，一个页表。此时的执行就由串型执行变为并发执行，效率更高。(???
- 我们之前所写的程序为内部只有一个执行流的进程。而多线程即内部具有多个执行流的进程。
  重新理解定义task_struct：进程内部的一个执行流（Linux下）
- CPU执行的基本单位是线程，OS调度的基本单位是线程。但并不是OS只能调度线程，实际上OS也是可以以进程整体为单位进行调度的。（见下方疑问）

# 重新理解进程的概念


![img](https://img-blog.csdnimg.cn/img_convert/288257a39f4195f320e1a3282b223a39.png)如上红色区域即进程全部。

**从用户视角来说**：进程=内核数据结构+进程对应的代码和数据（内核数据结构中PCB的数量>= 1）

**从内核视角来说**：进程：承担分配系统资源的基本实体。（因为在进程创建时，系统给这个进程分配资源。而线程是使用创建此线程的进程的部分资源，进程进行资源分配，分配给线程。故进程才是承担分配系统资源的基本实体）

> 疑问
>
> 对于线程这块我有个疑惑，既然CPU和OS调度的基本单位是线程，且一个进程内至少有一个执行线程（一个执行流），那能不能说CPU和OS只能调度线程呢，也就是理解为CPU调度一个单执行流进程时，本质上也是调度此进程内的一个执行线程？
>
> [一个CSDN链接，可供参考](https://blog.csdn.net/weixin_42387906/article/details/113052932)
>
> 调度是一方面，另一方面是调度的目的是什么。如果调度的目的是为了让执行流去执行，那肯定是让线程去跑。而如果目的是以进程整体为单位进行资源分配，则OS也是可以做到调度整个进程的。对于CPU来说，特别是Linux下，CPU并不关心线程还是进程，它只关心task_struct，因为在Linux下，只存在轻量级进程，不存在线程。只是用轻量级进程去模拟线程。

# 线程优缺点

## 线程的优点

创建一个新线程的代价要比创建一个新进程小得多（创建时的成本更小

与进程之间的切换相比，线程之间的切换需要操作系统做的工作要少很多（切换的成本更小

线程占用的资源要比进程少很多（占用的资源更少

能充分利用多处理器的可并行数量（并行

在等待慢速I/O操作结束的同时，程序可执行其他的计算任务

计算密集型应用，为了能在多处理器系统上运行，将计算分解到多个线程中实现

I/O密集型应用，为了提高性能，将I/O操作重叠。线程可以同时等待不同的I/O操作。

## 线程的缺点

性能损失
一个很少被外部事件阻塞的计算密集型线程往往无法与其它线程共享同一个处理器。如果计算密集型线程的数量比可用的处理器多，那么可能会有较大的性能损失，这里的性能损失指的是增加了额外的同步和调度开销，而可用的资源不变。

健壮性降低
编写多线程需要更全面更深入的考虑，在一个多线程程序里，因时间分配上的细微偏差或者因共享了不该共享的变量而造成不良影响的可能性是很大的，换句话说线程之间是缺乏保护的。

缺乏访问控制
进程是访问控制的基本粒度，在一个线程中调用某些OS函数会对整个进程造成影响。

编程难度提高
编写与调试一个多线程程序比单线程程序困难得多

# 线程异常

单个线程如果出现除零，野指针问题导致线程崩溃，进程也会随着崩溃

线程是进程的执行分支，线程出异常，就类似进程出异常，进而触发信号机制，终止进程，进程终止，该进程内的所有线程也就随即退出

# 线程用途

合理的使用多线程，能提高CPU密集型程序的执行效率

合理的使用多线程，能提高IO密集型程序的用户体验（如生活中我们一边写代码一边下载开发工具，就是 多线程运行的一种表现）

# 进程vs线程

进程是资源分配的基本单位

线程是调度的基本单位。

## 线程共享进程内的部分数据

**进程内的所有线程共享同一个进程地址空间，则其中的代码区，全局数据区，共享区，命令行参数和环境变量，内核区都是共享的。**而对于堆区和栈区，根本上来说是共享的，因为一个线程可以将栈帧内的局部数据或堆区开辟空间的地址通过全局数据的方式传递给其他线程，其他线程也可以访问。但是一般情况下我们不会这样做，**所以也可以认为栈区和堆区是线程私有的。**

**除此之外，各线程还共享进程内的文件描述符表（一个线程打开一个文件，其他线程也会自动打开），各种信号的处理方式（SIG_IGN，SIG_DFL或者自定义的信号处理函数（代码区）），当前工作目录，用户id和组id等...**

## 属于线程自己的一部分数据

线程id，`errno`，信号屏蔽字，调度优先级。

最重要的是：**每个线程私有 一组寄存器和栈。**

> 一组寄存器：**线程是CPU调度的基本单位，每个线程一定有自己的上下文。在线程被CPU调度时，上下文数据就会保存在CPU内的一组寄存器中。**
>
> 栈：**每个线程运行时要调用函数，一定有出栈入栈的行为，形成的临时变量会保存在栈内，故每个线程必须有自己的私有栈。**
>
> 一组寄存器和栈能体现出线程的动态属性。

验证每个线程有独立的信号屏蔽字

```C++
#include <iostream>
#include <string>
#include <thread>
#include <signal.h>
#include <unistd.h>
#include <string>
using namespace std;
 
void showSigblock(sigset_t* curSigset);
 
void *routine(void *arg)
{
    int cnt = 0;
    while (true)
    {
        cnt++;
        sleep(1);
        cout << "新线程的信号屏蔽字为 : ";
        sigset_t curSigset;
        sigprocmask(SIG_BLOCK, nullptr, &curSigset);
        showSigblock(&curSigset);
        if(cnt == 5) break;
    }
    string *p = new string("新线程执行完毕");
    pthread_exit(reinterpret_cast<void*>(p));
}
 
void showSigblock(sigset_t* curSigset)
{
    for (int i = 1; i <= 31; ++i)
    {
        if (sigismember(curSigset, i))
            cout << "1";
        else
            cout << "0";
    }
    cout << endl;
}
 
int main()
{
    // 主线程
    sigset_t sigset;
    sigemptyset(&sigset);
    sigaddset(&sigset, 8);
    sigaddset(&sigset, 1);
    sigaddset(&sigset, 11);
    sigprocmask(SIG_BLOCK, &sigset, nullptr);
 
    sigset_t curSigset;
    sigprocmask(SIG_BLOCK, nullptr, &curSigset);
    cout << "主线程此时的信号屏蔽字为 : ";
    showSigblock(&curSigset);
 
    pthread_t pid;
    pthread_create(&pid, nullptr, routine, nullptr);
 
    sleep(3);
    sigaddset(&sigset, 3);
    sigprocmask(SIG_BLOCK, &sigset, nullptr);
    cout << "主线程成功将3号信号进行屏蔽 : ";
    sigprocmask(SIG_BLOCK, nullptr, &curSigset);
    showSigblock(&curSigset);
 
    void* retThread = nullptr;
    pthread_join(pid, &retThread);
    cout << *(string*)retThread << endl;
    delete (string*)retThread;
    return 0;
}
```

```bash
.[yzl@VM-4-5-centos Thread]$ ./mythread 
主线程此时的信号屏蔽字为 : 1000000100100000000000000000000
新线程的信号屏蔽字为 : 1000000100100000000000000000000
新线程的信号屏蔽字为 : 1000000100100000000000000000000
主线程成功屏蔽3号信号: 1010000100100000000000000000000
新线程的信号屏蔽字为 : 1000000100100000000000000000000
新线程的信号屏蔽字为 : 1000000100100000000000000000000
新线程的信号屏蔽字为 : 1000000100100000000000000000000
新线程执行完毕
```

## 为什么线程切换的成本更低，而进程切换成本更高

进程内的线程之间切换时，**进程地址空间和页表不需要切换**（尽管它们在CPU内就是一个寄存器值），**CPU内的进程范畴的状态寄存器也不需要切换**。这些成本不是很高，故不是最重要的。

**重点是：CPU内有L1~L3的cache（高速缓存），对内存的代码和数据，根据局部性原理，预读到CPU内部。若进程切换，则cache立即失效，切换为新进程之后，cache需要进行预热和重新缓存。这个的影响更大。**

GPT：线程切换的成本比进程切换的成本低的原因是：线程是轻量级的，共享父进程的资源，**切换时需要保存的状态信息相对较少**；而进程是独立的，拥有自己的资源，**切换时需要保存的状态信息相对较多。**因此线程切换的速度比进程切换的速度快。

# 线程控制

## pthread库介绍

![img](https://img-blog.csdnimg.cn/img_convert/2173033f31d4a72abd8422527defe42e.png)

![img](https://img-blog.csdnimg.cn/img_convert/fcd41e32fb0b48ea906ab4c87e34f110.png)

Linux操作系统并没有设计真正的线程，而是用轻量级进程来模拟线程。故Linux无法直接提供针对线程的系统调用（如线程创建，终止...），最多只能提供轻量级进程的系统接口。但是Linux必须满足用户对于线程的使用需求。最终，在用户层提供了一个pthread库，这里面包含了用户对于线程的使用接口。严格来说，这个pthread库是第三方库，并不是系统库和语言库。但是所有版本的Linux操作系统都直接提供了这个线程库在系统默认路径中。（上图为动静态库和头文件）

因为这个库严格来说并不是系统库和语言库，而是第三方库。故用g++编译时要加上-lpthread指令说明要链接这个库。（g++默认使用动态库进行动态链接，头文件在源文件中已经声明了，而因为libpthread-2.17.so已经在系统默认搜索路径下了，故编译好之后可以直接运行可执行，不会出现链接错误）

## pthread线程库API

```C++
       pthread_create - create a new thread
       int pthread_create(pthread_t *thread, const pthread_attr_t *attr,
                          void *(*start_routine) (void *), void *arg);
 
       pthread_join - join with a terminated thread
       int pthread_join(pthread_t thread, void **retval);
 
       pthread_exit - terminate calling thread
       void pthread_exit(void *retval);
 
       pthread_cancel - send a cancellation request to a thread
       int pthread_cancel(pthread_t thread);
 
       pthread_detach - detach a thread
       int pthread_detach(pthread_t thread);
 
       pthread_self - obtain ID of the calling thread
       pthread_t pthread_self(void);
```

### 线程创建：pthread_create

线程id地址（本质是一个unsigned long int类型），输出型参数。线程属性设置，直接设为nullptr即可。新线程的执行函数，函数指针类型，参数和返回值必须为void*（一般是通过强转进行的）。线程执行方法的参数。

### 线程等待：pthread_join

1.线程id，2.用void\*\*接收线程执行的start_routine函数的返回值void\*。这里传void\*\*接收，是C语言典型的输出型参数指针传参问题...

### 线程等待的原因

1. 假设主线程中创建了一个新线程并且分配了一个任务，如果不等待新线程，那么主线程可能在新线程未完成任务时就已经结束了，这可能导致新线程的数据不一致或内存泄漏的问题。因此，调用pthread_join方法等待新线程可以保证主线程在新线程完成任务后再继续执行。
2. 如果主线程不需要确保在新线程将任务执行完之后再继续执行，也需要调用pthread_join等待回收一下新线程，这样可以避免产生类似于僵尸进程的问题。
3. 已经退出的线程，其空间没有被释放，仍然在进程的地址空间内。
4. 创建新的线程不会复用刚才退出线程的地址空间。

### 线程终止：pthread_exit

- 线程执行的start_routine函数执行return
- 调用pthread_exit(void* retval)进行线程退出，retval就是return的返回值。
- pthread_cancel(pthread_t thread)：如果主线程调用pthread_cancel取消新线程，则新线程会返回PTHREAD_CANCELED，宏定义，-1。（一般不建议这样做）

需要注意,pthread_exit或者return返回的指针所指向的内存单元必须是全局的或者是用malloc分配的,不能在线程函 数的栈上分配,因为当其它线程得到这个返回指针时线程函数已经退出了。

### 线程分离：pthread_detach

若不关心一个新线程的退出结果，则join是一个负担，（若不join会造成无法释放资源，造成系统泄漏），在新线程终止后也不想join回收新线程（如果不关心退出结果，那肯定不想join啊）。则可以调用pthread_detach分离新线程。作用类似于`signal(SIGCHLD, SIG_IGN)`

在Linux下，调用pthread_detach函数将线程分离，使得该线程独立运行，不再需要由任何其他线程等待其终止。分离的线程不需要被回收，内存资源在线程终止时自动回收。

```C++
#include <iostream>
#include <string>
#include <thread>
#include <signal.h>
#include <unistd.h>
#include <string>
using namespace std;
 
 
#define N 10
 
void* routine(void* arg)
{
    long long num = 0;
    for(int i = 0; i < N; ++i)
    {
        if(((int*)arg)[i] % 2 == 0) num++;
    }
    pthread_exit((void*)num);
    // return (void*)num;  // 偶数个数
}
 
int main()
{
    int* p = new int[N];
    srand(time(nullptr));
    for(int i = 0; i < N; ++i)
    {
        p[i] = rand()%100;
    }
    pthread_t pid = 0;
    pthread_create(&pid, nullptr, routine, (void*)p);
 
    void* retVal;
    pthread_join(pid, &retVal);
    cout << "共" << (long long)retVal << "个偶数" << endl;
    return 0;
}
```

### 补充

- 一般情况下，不建议调用pthread_cancel进行线程取消。
- 新线程cancel主线程，或者主线程调用pthread_exit退出，新线程都可以继续执行，但是不建议这样做，主线程会处于defunct状态。
  多进程和多线程中，都是让父进程和主线程最后退出，父进程和主线程承担起创建和回收资源的职责。通常多线程编程就是，pthread_create创建新线程，新线程pthread_exit或者return，主线程pthread_join，若不关心退出结果且不想join等待回收则pthread_detach即可。
- 进程中任何一个线程发生异常，收到信号。都会导致整个进程终止。进程中所有线程也会终止。同理，任何一个线程调用execl，整个进程就会发生进程替换。
- joinable和分离是冲突的，一个线程不能既是joinable又是分离的。

# 线程id，线程的独立栈结构，线程的局部存储

## 线程id的本质

前面说了，Linux操作系统只设计提供了轻量级进程，而用户需要的是线程。于是有了用户层的pthread线程库。

因此，对于线程的管理，OS承担一部分，pthread库承担一部分。OS承担的是对于轻量级进程的创建调度等和对于内核数据结构的管理。而库需要进一步包装描述内核的轻量级进程，需要提供一些线程相关的属性字段，比如：线程id。

OS完成的是对轻量级进程的调度，管理工作。而线程的用户层属性是pthread库管理的，管理就要先描述再组织。

<img src="https://img-blog.csdnimg.cn/2d718b7fa19c4d2686aab4febd5403d6.png" alt="img" style="zoom: 50%;" />

**线程id就是这个线程在pthread库中的属性集合的起始地址。pthread库，如动态库是会加载到进程地址空间中的共享区的，因此线程id本质就是共享区的一个地址。**

## 线程的独立栈结构

之前我们说过，线程有两个很重要的私有数据：一组寄存器和栈。

进程内的所有线程共享一个进程地址空间，进程地址空间中代码区，全局数据区，堆区，共享区，内核区都没有问题，可是如何保证每个线程有自己独立的栈结构？所有线程共享一个栈空间是不可以的。

**新线程的栈空间是用户层pthread库提供并维护的。**

**如上图所示，主线程使用的是进程地址空间中内核级的栈区，而新线程使用的是pthread库提供的用户级栈区。从而保证每个线程有自己独立的栈结构。**

<img src="https://img-blog.csdnimg.cn/b08edc3ae0684175bf1b82f5b613b1e9.png" alt="img" style="zoom: 80%;" />

上图为验证线程id的程序，发现线程id转化为地址之后确实在堆栈之间的共享区中，因为是动态链接的pthread动态库。

## clone系统调用

<img src="https://img-blog.csdnimg.cn/46cc55a991cb493fbe9902f03be67111.png" alt="img" style="zoom: 80%;" />


Linux提供了一个clone系统调用，作用是创建一个轻量级进程，和所属进程共享一个进程地址空间，而这个轻量级进程独立的栈结构就可以通过第二个参数void* child_stack传递。pthread_create内部可能就调用了这个clone系统调用。

ChatGPT：

在Linux中，进程对不同线程进行资源划分的方法是：通过Clone系统调用创建线程。Clone系统调用允许父进程在创建子进程时，指定如何共享资源。
每个线程都有一个进程控制块(PCB)，该PCB记录了该线程的信息，如线程ID、执行状态等。线程之间可以共享进程的一些资源，如文件描述符表、地址空间、进程ID等。同时，每个线程还有一些独有的资源，如线程栈、寄存器状态等。
因此，在Linux中，线程可以通过Clone系统调用进行资源划分，使得多个线程可以共享一些公共资源，同时又有独立的资源。这种资源划分方法有助于提高程序的效率，同时又保证了线程之间的隔离性。

## \_\_thread ：线程的局部存储

**全局变量在全局数据区，是所有线程共享的。如果希望每个线程都有一份独立的某全局变量，则可以用__thread修饰该全局变量。这称为线程的局部存储。**

<img src="C:\Users\yangzilong\AppData\Roaming\Typora\typora-user-images\image-20230810193514000.png" alt="image-20230810193514000" style="zoom: 50%;" />

其实本质上就是存储在了pthread库中每个线程属性集合内的线程局部存储空间中

GPT ：\__thread可以用于修饰全局变量和局部变量，它表示线程本地存储（Thread Local Storage），即每个线程都有自己独立的存储空间，不与其他线程共享。**使用__thread修饰的变量，在同一线程中可以直接访问，不需要加锁，效率更高。**

```C++
#include <iostream>
#include <string>
#include <thread>
#include <signal.h>
#include <unistd.h>
#include <string>
using namespace std;
 
// 验证线程id的本质
 
__thread int num = 6;
 
void* routine(void* arg) {}
 
int main()
{
    pthread_t tid;
    pthread_create(&tid, nullptr, routine, nullptr);
    int a = 10;
    int *p = new int(10);
    printf("栈区 : %p\n", &a);
    printf("tid  : %p\n", tid);
    printf("堆区 : %p\n", p);
    printf("num : %p\n", &num);
    return 0;
}
```

```bash
.[yzl@VM-4-5-centos Thread]$ ./mythread 
栈区 : 0x7ffdf25d43cc
tid  : 0x7f0b310f8700
堆区 : 0x1f3d260
num : 0x60105c
[yzl@VM-4-5-centos Thread]$ make
g++ -o mythread mythread.cc -std=c++11 -lpthread
.[yzl@VM-4-5-centos Thread]$ ./mythread 
栈区 : 0x7ffed17feccc
tid  : 0x7f56c3fb1700
堆区 : 0x67a270
num : 0x7f56c4fce77c
```

第一次运行时num没有被__thread修饰

> 浅谈C++线程库
>
> ```C++
> #include <iostream>
> #include <string>
> #include <thread>
> #include <signal.h>
> #include <unistd.h>
> // #include <pthread.h>
> #include <string>
> // using namespace std;
>  
>  
> // C++线程库
>  
> void routine()
> {
>     std::cout << "haha" << std::endl;
> }
>  
> int main()
> {
>     std::thread t1(routine);
>     t1.join();
>     return 0;
> }
> ```
>
> ```bash
> [yzl@VM-4-5-centos Thread]$ ldd mythread
>     linux-vdso.so.1 =>  (0x00007ffdf5fbd000)
>     libpthread.so.0 => /lib64/libpthread.so.0 (0x00007fe163154000)
>     libstdc++.so.6 => /lib64/libstdc++.so.6 (0x00007fe162e4c000)
>     libm.so.6 => /lib64/libm.so.6 (0x00007fe162b4a000)
>     libgcc_s.so.1 => /lib64/libgcc_s.so.1 (0x00007fe162934000)
>     libc.so.6 => /lib64/libc.so.6 (0x00007fe162566000)
>     /lib64/ld-linux-x86-64.so.2 (0x00007fe163370000)
> ```
>
> 如上所示，虽然这个程序没有包pthread.h头文件，但是这个可执行依旧链接了pthread动态库。因为使用了C++线程库。
>
> 故，C++线程库底层一定使用了原生线程库。（虽然pthread库不是系统调用，但对于Linux用户来说，地位上几乎已经和系统调用等价了），任何一个语言，要想在Linux下跑多线程，一定要使用原生线程库。
>
> 语言上的线程库，底层必须使用原生线程库，一定是对原生线程库进行封装，目的是让用户更好地使用。类似于C语言的FILE结构体内部一定包含文件描述符fd字段。

# 补充

OS是可以做到使进程进行资源的细粒度划分的。(完善Linux线程的原理)

![img](https://img-blog.csdnimg.cn/bdf9e47206c5410e8aa7cf01ea9be65a.png)


Linux通过内存管理机制和进程管理机制实现了进程对内存资源的细粒度划分。

vm_area_struct是内核内存管理的一个数据结构，它表示一个进程的虚拟内存地址空间中的一段连续区域（例如代码段、堆、栈等）。通过这个数据结构，系统可以对进程的内存资源进行管理和分配。（例如将代码段中的某连续区域或堆区的某块区域划分给不同的线程）

**因此，vm_area_struct与进程资源的细粒度划分具有密切关系，它是实现进程内存资源管理的关键数据结构。**（进程/线程在堆区不断开辟空间，进程地址空间中堆区只有一个start和end，是无法准确管理每次申请的堆区空间的，底层就是使用vm_area_struct对每一段空间进行管理）

> vm_area_struct结构包含了以下关键字段
>
> vm_start：虚拟内存区域的起始地址
> vm_end：虚拟内存区域的结束地址
> vm_flags：标识虚拟内存区域的特征，如是否可读、可写、可执行等
> vm_page_prot：虚拟内存区域的页保护标志
> vm_file：如果这个虚拟内存区域是从文件映射而来，则指向该文件的指针；否则为NULL
> vm_pgoff：文件映射的偏移量
> vm_ops：操作虚拟内存区域的函数指针
>
> 这些字段是vm_area_struct的核心内容，它们共同描述了进程的虚拟内存地址空间，以及如何管理和使用这些虚拟内存。

(和进程对不同线程进行资源划分联系起来

