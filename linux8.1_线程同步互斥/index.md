# 

# 线程互斥

## 线程互斥及相关概念

**线程互斥（Mutual Exclusion）**是指在多线程环境下，同一时刻只能有一个线程访问共享资源，以避免对该资源的不正确访问，造成数据不一致等问题。
例如，如果有多个线程都要同时对同一个全局变量进行修改，那么就需要使用线程互斥来保证对该变量的访问是互斥的，也就是说，在任意时刻只能有一个线程对该变量进行访问。

**临界资源（Critical Resource）**是指在多线程环境下需要被多个线程共享访问的资源，对该资源的访问需要进行同步（如使用互斥进行同步）以避免出现不正确的访问。

**临界区**：每个线程内部，访问临界资源的代码，就叫做临界区

**互斥**：任何时刻，互斥保证有且只有一个执行流进入临界区，访问临界资源，通常对临界资源起保护作用。是对临界资源保护的一种手段。

**原子性**：不会被任何调度机制影响的操作，该操作只有两态，要么完成，要么未完成。

**互斥量mutex**：
大部分情况，线程使用的数据都是局部变量，变量的地址空间在线程栈空间内，这种情况，变量归属单个线程，其他线程无法获得这种变量。 
但有时候，很多变量都需要在线程间共享，这样的变量称为共享变量，可以通过数据的共享，完成线程之间的交互。 
多个线程并发的操作共享变量，会带来一些问题。

## 多线程抢票

```C++
#include <iostream>
#include <pthread.h>
#include <string>
#include <unistd.h>
 
using namespace std;
 
// 共享资源，多线程同时访问(未来的临界资源)
int tickets = 10000;
 
void* getTickets(void* args)
{
    string* ps = (string*)args;
    while(true)
    {
        if(tickets > 0)  // 未来的临界区
        {
            usleep(1000);
            printf("%s : %d\n", ps->c_str(), tickets); // 未来的临界区
            // cout << *ps << " get ticket " << tickets << endl; 
            tickets--; // 未来的临界区
        }
        else
        {
            break;
        }
    }
    delete ps;
    return nullptr;
}
 
 
// 多线程抢票程序
int main()
{
    pthread_t tid[3];
    for(int i = 0; i < 3; ++i)
    {
        string* ps = new string("thread");
        ps->operator+=(to_string(i+1));
        pthread_create(tid + i, nullptr, getTickets, (void*)ps);
    }
    for(int i = 0; i < 3; ++i)
    {
        pthread_join(tid[i], nullptr);
        // printf("主线程等待回收新线程%d成功\n", i + 1);
        // cout << "主线程等待回收新线程" << i+1 << "成功" << endl;
    }
    return 0;
}
```

<img src="https://img-blog.csdnimg.cn/img_convert/f9149b93c594d7d09388ffbcdf5eab6d.png" alt="img" style="zoom:80%;" />

上方程序为多线程抢票程序，全局数据tickets为共享资源（未来的临界资源，此时还没有进行互斥保护)，多个线程对getTickets函数进行了重入，getTickets方法中对全局tickets变量访问和修改的代码都是未来的临界区，如if判断，printf打印及tickets--代码都是未来的临界区代码。

因为多线程并发执行，访问共享资源，因时序及线程切换等原因造成的数据不一致等问题。我们则需要对访问共享资源的代码进行加锁保护，进行线程互斥。

> 为什么多个线程并发访问共享资源时，因为线程切换就会造成数据不一致呢？下面举两点解释说明：
>
> if(tickets > 0)：tickets > 0判断的本质也是计算，则该代码执行时需要CPU进行逻辑运算，tickets全局数据存储在内存中，则需要将tickets数据load到CPU寄存器中，本质就是将数据load到当前线程的上下文中。执行if判断的后面代码块时，因为多线程并发执行，此时的执行线程随时可能被切换（此时寄存器中的线程上下文数据也会被线程保存起来），则就可能造成多个执行线程同时进入if判断内部。若此时tickets为1，则就会因为线程切换造成tickets减到0甚至-1。（上方程序中的usleep更加提高了这种情况发生的可能性）
>
> tickets--：这条C语句在不进行优化的情况下翻译为汇编时，最少会变为三条：1. load到CPU寄存器中 2. 对寄存器内容进行-- 3. 将寄存器数据load回内存中。因此这个--操作并不是原子的，执行到哪一步时都有可能进行线程切换。则存在以下场景:两个线程，线程1和线程2接下来要进行tickets--操作，此时 tickets为10，线程1执行完load到寄存器之后，被切换了，此时线程1会保存它的上下文数据，比如此时保存tickets的寄存器值，其他临时数据，程序计数器eip，栈顶栈底指针的值等。线程2执行tickets--的过程中没有被切换，此时内存中的tickets的值成功被--到了9。再切换为线程1，线程1执行第二步和第三步。此时内存中的tickets又被重复写入到了9。

造成上方多线程抢票程序问题的主要原因其实是第一点，第二点也会存在，只是概率相对更低。实际的执行的情况会比上方所述复杂的多，**总之这样的多线程并发访问共享资源的程序是有问题的。**

**因此我们需要进行线程互斥，常见的保护措施就是互斥锁。**使得加锁和解锁之间的代码区域同一时刻只能有一个线程执行，这样的代码区域称为临界区，tickets数据称为临界资源。

Linux上提供的这把锁叫互斥量。

## pthread线程库mutex互斥锁（互斥量）

```C++
// 初始化互斥锁
    // 1. 静态分配，适用于全局或静态的互斥锁
       pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
    // 2. 动态分配，适用于局部的互斥锁
       int pthread_mutex_init(pthread_mutex_t *restrict mutex,
              const pthread_mutexattr_t *restrict attr);   // 参数二设为nullptr即可
// 销毁互斥锁
    // 使用PTHREAD_MUTEX_INITIALIZER初始化的互斥锁不需要销毁
       int pthread_mutex_destroy(pthread_mutex_t *mutex);
// 加锁，解锁
       int pthread_mutex_lock(pthread_mutex_t *mutex);
       int pthread_mutex_unlock(pthread_mutex_t *mutex);
    // int pthread_mutex_trylock(pthread_mutex_t *mutex);
```

调用pthread_mutex_lock时，可能出现以下情况。

1. 互斥锁处于未锁状态，该函数会将互斥锁锁定，同时返回0（成功

2. 调用时，互斥锁已经被其他线程锁定，或者有其他线程同时申请互斥锁且此线程没有竞争到互斥锁。则pthread_mutex_lock会将调用线程进行阻塞等待，等待其他线程解锁该互斥锁。

<img src="C:\Users\yangzilong\AppData\Roaming\Typora\typora-user-images\image-20230811175623282.png" alt="image-20230811175623282" style="zoom:67%;" />

```C++
#include <iostream>
#include <pthread.h>
#include <string>
#include <unistd.h>
 
using namespace std;
 
// 共享资源，多线程同时访问(未来的临界资源)
int tickets = 3000;
// pthread_mutex_t mtx = PTHREAD_MUTEX_INITIALIZER;
 
struct ThreadData
{
public:
    ThreadData(const string& tname, pthread_mutex_t* pmtx)
    : tname_(tname), pmtx_(pmtx)
    {}
    string tname_;
    pthread_mutex_t* pmtx_;
};
 
void* getTickets(void* args)
{
    ThreadData* td = (ThreadData*)args;
    while(true)
    {
        pthread_mutex_lock(td->pmtx_);
        if(tickets > 0)  // 未来的临界区
        {
            usleep(1000);
            printf("%s : %d\n", td->tname_.c_str(), tickets); // 未来的临界区
            // cout << *ps << " get ticket " << tickets << endl;
            tickets--; // 未来的临界区
            pthread_mutex_unlock(td->pmtx_);
        }
        else
        {
            pthread_mutex_unlock(td->pmtx_);
            break;
        }
        // usleep(1000);
    }
    delete td;
    pthread_exit(nullptr);
    // return nullptr;
}
 
// 多线程抢票程序
int main()
{
    pthread_t tid[3];
    pthread_mutex_t mtx;
    pthread_mutex_init(&mtx, nullptr);
    for(int i = 0; i < 3; ++i)
    {
        string s("thread");
        s += to_string(i+1);
        ThreadData* td = new ThreadData(s, &mtx);
        pthread_create(tid + i, nullptr, getTickets, (void*)td);
    }
    for(int i = 0; i < 3; ++i)
    {
        pthread_join(tid[i], nullptr);
        // printf("主线程等待回收新线程%d成功\n", i + 1);
        // cout << "主线程等待回收新线程" << i+1 << "成功" << endl;
    }
    pthread_mutex_destroy(&mtx);
    return 0;
}
```

- 不加锁时，多线程并发执行，效率较高。加锁之后，同一时刻只会有一个线程执行临界区代码，其他线程都会在pthread_mutex_lock这里阻塞等待，等待这个锁被解锁。效率会降低。**因此加锁的粒度越小越好。**

- 加锁之后，线程在临界区内依旧会被切换，但是不会造成之前的数据不一致等问题。**因为执行线程切换时，是持有锁被切换的**（调用过pthread_mutex_lock），其他线程要想进入临界区执行临界区代码，也要先申请锁，此时它是申请不成功的，会阻塞等待持有锁线程解锁。因此同一时刻只会有一个线程进入临界区执行临界区代码访问临界资源。从而保证了临界区中数据的一致性。

- **加锁之后，临界区代码是串行执行的。而不是之前的多线程并发执行。**

要进入临界区访问临界资源，每个线程必须先调用pthread_mutex_lock申请锁，则每个线程必须看到同一把锁&&访问它（pthread_mutex_t）。则锁本身就是一个共享资源（类比上方的tickets），那么如何保证多线程加锁时，访问锁的安全呢？

## 互斥锁mutex的实现原理

> 共识：
>
> 在汇编的角度，我们认为一条汇编语句的执行是原子的，也就是要么执行完成，要么未执行。没有中间态。
>
> 在执行流视角，CPU内部的寄存器，本质叫做当前执行流的上下文。这些寄存器，空间上是被所有执行流共享的，但是寄存器的内容，是每一个执行流执行时私有的，叫做执行流的上下文。

<img src="https://img-blog.csdnimg.cn/img_convert/8df4d713652c4faa18cf2184e4c5afc0.png" alt="img" style="zoom:80%;" />

上图为pthread_mutex_lock和pthread_mutex_unlock的伪代码。

为了实现互斥锁操作,大多数体系结构都提供了swap或exchange指令,该指令的作用是把寄存器和内存单元的数据相交换,由于只有一条指令，因此该操作是原子的，保证了操作的原子性。

lock：mutex可以理解为pthread_mutex_t互斥锁，初始化后，在内存中它的值为1。假设现有两个线程，线程a执行movb，将CPU寄存器%al的值置为0，然后被切换了（注意此时%al寄存器的内容是线程a的上下文，切换时要保存起来），线程b执行movb，exchange，将内存中mutex内存块存储的1和%al的0相交换（注意此操作是原子的），至此，线程b申请锁成功，后面会执行return语句。此时内存中mutex的值为0。线程切换为线程a，根据程序计数器的值，继续执行exchange语句，将%al的0和内存中的0交换（表示线程b申请锁失败，此时互斥锁已经被其他线程锁定了，竞争锁失败）。之后就会执行挂起等待，等待申请锁的线程执行unlock：将内存中mutex的值置为1，下次线程a执行goto lock，如果是第一个执行exhcnage %al，mutex语句的线程，则线程a竞争锁成功。

**竞争锁的原理：其实根本上就是竞争锁时，利用exchange这样的指令的原子性，谁先执行exchange，将内存中1这样的公有数据变为线程上下文数据（私有数据），则表示竞争锁成功。**

重新理解，线程在临界区内也会被切换，但是它是持有锁被切换的（那个内存中的1）。其他线程要想进入临界区比如申请锁，此时会申请失败阻塞等待，等待持有锁线程解锁（执行unlock）

## 可重入VS线程安全

- 线程安全：多个线程并发同一段代码时，不会出现不同的结果。常见对全局变量或者静态变量进行操作， 并且没有锁保护的情况下，会出现该问题。
- 重入：**同一个函数被不同的执行流调用**，当前一个流程还没有执行完，就有其他的执行流再次进入，我们称之为重入。一个函数在重入的情况下，运行结果不会出现任何不同或者任何问题，则该函数被称为可重 入函数，否则，是不可重入函数。

常见的线程不安全的情况

- 不保护共享变量的函数
- 函数状态随着被调用，状态发生变化的函数
- 返回指向静态变量指针的函数
- 调用线程不安全函数的函数

常见的线程安全的情况

- 每个线程对全局变量或者静态变量只有读取的权限，而没有写入的权限，一般来说这些线程是安全的

- 类或者接口对于线程来说都是原子操作
- 多个线程之间的切换不会导致该接口的执行结果存在二义性

常见不可重入的情况

- 调用了malloc/free函数，因为malloc函数是用全局链表来管理堆的
- 调用了标准I/O库函数，标准I/O库的很多实现都以不可重入的方式使用全局数据结构
- 可重入函数体内使用了静态的数据结构（函数状态随着调用而变化）

常见可重入的情况

- 不使用全局变量或静态变量
- 不使用用malloc或者new开辟出的空间
- 不调用不可重入函数
- 不返回静态或全局数据，所有数据都有函数的调用者提供
- 使用本地数据，或者通过制作全局数据的本地拷贝来保护全局数据（数据的局部存储）

可重入与线程安全的联系

- 函数是可重入的，那就是线程安全的

- 函数是不可重入的，那就不能由多个线程使用，有可能引发线程安全问题

- 如果一个函数中有全局变量，那么这个函数既不是线程安全也不是可重入的。

可重入与线程安全区别

- 可重入函数是线程安全函数的一种

- 线程安全不一定是可重入的，而可重入函数则一定是线程安全的。

- 如果将对临界资源的访问加上锁，则这个函数是线程安全的，但如果这个重入函数若锁还未释放则会产生死锁，因此是不可重入的。（第二点的一种情况）

## 死锁

**死锁（Deadlock）是指两个或多个进程（或线程）在执行过程中，因互相等待对方释放资源而陷入无限等待的一种状态。**

死锁是指在一组进程中的各个进程均占有不会释放的资源，但因互相申请被其他进程所站用不会释放的资 源而处于的一种永久等待状态。

例如，如果线程A获取了锁1，正在申请锁2，需要等待线程B释放锁2才能继续执行，而线程B同时获取了锁2，正在申请锁1，但需要等待线程A释放锁1才能继续执行，那么两个线程就会陷入无限等待的状态，即死锁。

死锁是一种非常严重的问题，因为它会导致应用程序无法继续执行，并可能导致系统崩溃。因此，在编写多线程或多进程的应用程序时，必须小心处理锁的获取和释放顺序，以避免死锁的发生。

### 死锁的四个必要条件

- 互斥条件：一个资源每次只能被一个执行流使用（多线程互斥场景下使用了互斥锁）

- 请求与保持条件：一个执行流因请求资源而阻塞时，对已获得的资源保持不放

- 不剥夺条件:一个执行流已获得的资源，在末使用完之前，不能强行剥夺

- 循环等待条件:若干执行流之间形成一种头尾相接的循环等待资源的关系

### 避免死锁

- 破坏死锁的四个必要条件

- 加锁顺序一致

- 避免锁未释放的场景

- 资源一次性分配

# 线程同步

## 线程同步解决的问题

- 一、例如上方的多线程抢票程序，会发生一段时间甚至整个程序运行过程中都是一个线程在抢票，也就是某一个线程因为调度器调度的缘故一直抢到了锁，获取了临界资源。导致其他线程长时间访问不到临界资源，造成其他线程的饥饿问题。
  GPT：执行流饥饿（Starvation）问题是指某个线程或进程无法获得所需的系统资源，导致它无法继续执行的一种情况。在并发编程中，如果多个线程或进程同时竞争一些共享资源，可能会出现某些线程或进程一直得不到访问共享资源的机会，导致它们无法执行或执行效率非常低下，这就是执行流饥饿问题。

- 二、线程在访问临界资源前需要加锁，这是为了保护临界资源，避免多线程并发访问导致数据不一致的问题。同时，在某些情况下，例如一个线程访问队列时，发现队列为空，它只能等待，直到其它线程将一个节点添加到队列中。这种情况就需要用到条件变量进行线程同步。
  此情况下，队列为临界资源，线程在获取临界资源前需要先判断临界资源是否就绪，是否满足获取条件，而这种判断本质也是对临界资源的一种访问，故需要在加锁和解锁之间的互斥条件下进行。因此可能造成一种情况：线程加锁，判断临界资源（比如此情况下的队列）是否就绪，未就绪，解锁。因为它并不知道什么时候就绪，即队列中有新增节点（由另一个线程完成此工作），则该线程就需重复加锁，判断，解锁的工作。这个工作无疑是浪费锁资源，不合理的。

解决上方两种问题，我们就可以利用条件变量进行线程同步。

## 线程同步

同步：在保证数据安全的前提下，让线程能够按照某种特定的顺序访问临界资源，从而有效避免饥饿问题，叫做同步。

竞态条件：因为时序问题，而导致程序异常，我们称之为竞态条件。在线程场景下，这种问题也不难理解。

当一个线程互斥地访问某个变量时，它可能发现在其它线程改变状态之前，它什么也做不了。 例如：一个线程访问队列时，发现队列为空，它只能等待，直到其它线程将一个节点添加到队列中。这种情况就需要用到条件变量。

## 条件变量

```C++
// 条件变量的初始化与销毁
       int pthread_cond_destroy(pthread_cond_t *cond);
       int pthread_cond_init(pthread_cond_t *restrict cond,
              const pthread_condattr_t *restrict attr);
       pthread_cond_t cond = PTHREAD_COND_INITIALIZER;
// 使线程在cond条件变量下等待，mutex互斥锁
       int pthread_cond_wait(pthread_cond_t *restrict cond,
              pthread_mutex_t *restrict mutex);
// 唤醒在cond条件变量下等待的所有线程
       int pthread_cond_broadcast(pthread_cond_t *cond);
// 唤醒在cond条件变量下等待的一个线程
       int pthread_cond_signal(pthread_cond_t *cond);
```

pthread_cond_signal用于发送一个通知信号，唤醒等待在条件变量上的一个线程。如果有多个线程在等待，那么只有一个线程会被唤醒，并且系统并不保证哪个线程会被唤醒。因此pthread_cond_signal通常用于通知某个线程某资源已就绪，可以继续执行。

pthread_cond_broadcast用于发送广播通知信号，唤醒所有等待在条件变量上的线程。这意味着所有等待线程都会被唤醒，并且可以同时开始执行。因此，pthread_cond_broadcast通常用于通知多个线程可以同时开始执行。

```C++
#include <iostream>
#include <pthread.h>
#include <string>
#include <functional>
#include <unistd.h>
using namespace std;
 
struct ThreadData;
#define THREAD_NUM 5
typedef void (*func_t)(ThreadData*);
// #define std::function<void (ThreadData*)> func
 
bool quit = false;
 
struct ThreadData
{
public:
    ThreadData(const string& name, pthread_mutex_t* pmtx, pthread_cond_t* pcond, func_t func)
    :name_(name), pmtx_(pmtx), pcond_(pcond), func_(func)
    {}
    string name_;
    pthread_mutex_t* pmtx_;
    pthread_cond_t* pcond_;
    func_t func_;
};
 
void thread1(ThreadData* td)
{
    while(!quit)
    {
        pthread_mutex_lock(td->pmtx_);
        // 这里原本是要先判断临界资源是否就绪，若不就绪则wait等待
        cout << td->name_ << " wait" << endl;
        pthread_cond_wait(td->pcond_, td->pmtx_);
        cout << td->name_ << " wait done" << endl;
        pthread_mutex_unlock(td->pmtx_);
        sleep(1);
    }
}
 
void thread2(ThreadData* td)
{
    while(!quit)
    {
        pthread_mutex_lock(td->pmtx_);
        // 这里原本是要先判断临界资源是否就绪，若不就绪则wait等待
        cout << td->name_ << " wait" << endl;
        pthread_cond_wait(td->pcond_, td->pmtx_);
        cout << td->name_ << " wait done" << endl;
        pthread_mutex_unlock(td->pmtx_);
        sleep(1);
    }
}
 
void thread3(ThreadData* td)
{
    while(!quit)
    {
        pthread_mutex_lock(td->pmtx_);
        // 这里原本是要先判断临界资源是否就绪，若不就绪则wait等待
        cout << td->name_ << " wait" << endl;
        pthread_cond_wait(td->pcond_, td->pmtx_);
        cout << td->name_ << " wait done" << endl;
        pthread_mutex_unlock(td->pmtx_);
        sleep(1);
    }
}
 
void thread4(ThreadData* td)
{
    while(!quit)
    {
        pthread_mutex_lock(td->pmtx_);
        // 这里原本是要先判断临界资源是否就绪，若不就绪则wait等待
        cout << td->name_ << " wait" << endl;
        pthread_cond_wait(td->pcond_, td->pmtx_);
        cout << td->name_ << " wait done" << endl;
        pthread_mutex_unlock(td->pmtx_);
        sleep(1);
    }
}
 
void thread5(ThreadData* td)
{
    while(!quit)
    {
        pthread_mutex_lock(td->pmtx_);
        // 这里原本是要先判断临界资源是否就绪，若不就绪则wait等待
        cout << td->name_ << " wait" << endl;
        pthread_cond_wait(td->pcond_, td->pmtx_);
        cout << td->name_ << " wait done" << endl;
        pthread_mutex_unlock(td->pmtx_);
        sleep(1);
    }
}
 
void* entry(void* args)
{
    ThreadData* td = (ThreadData*)args;
    td->func_(td);
    delete td;
    pthread_exit(nullptr);
}
 
int main()
{
    pthread_mutex_t mtx;
    pthread_cond_t cond;
 
    pthread_mutex_init(&mtx, nullptr);
    pthread_cond_init(&cond, nullptr);
 
    pthread_t tid[THREAD_NUM];
    func_t funcs[THREAD_NUM] = {thread1, thread2, thread3, thread4, thread5};
    for(int i = 0; i < THREAD_NUM; ++i)
    {
        string name = "thread";
        name += to_string(i + 1);
        ThreadData* td = new ThreadData(name, &mtx, &cond, funcs[i]);
        pthread_create(tid + i, nullptr, entry, (void*)td);
    }
    int cnt = 10;
    while(cnt != 0)
    {
        cnt--;
        sleep(2);
        pthread_cond_signal(&cond);
        // pthread_cond_broadcast(&cond);
    }
    cout << "main thread control done" << endl;
    quit = true;
    pthread_cond_broadcast(&cond);
 
    for(int i = 0; i < THREAD_NUM; ++i)
    {
        pthread_join(tid[i], nullptr);
        printf("main thread waits thread%d success\n", i+1);
    }
 
    pthread_mutex_destroy(&mtx);
    pthread_cond_destroy(&cond);
    return 0;
}
```

现象结论

`pthread_cond_siganl`唤醒某条件变量下的一个线程时，并不是完全随机的，而是类似于在条件变量下组织了一个队列，按照等待顺序去唤醒。

`pthread_cond_broadcast`唤醒某条件变量下的全部线程时，确实是全部唤醒，但一次全部唤醒的内部也是有顺序的，也是按照等待顺序唤醒。

这就证实了利用条件变量进行线程同步时，可以让线程按照一定顺序访问临界资源，避免线程饥饿的问题。

> 上方这个条件变量的测试代码。仅仅是最简单的一种测试代码，实际上，这里的`pthread_cond_wait`和`pthread_cond_signal`的使用是完全生硬的使用测试。因为这里根本没有临界资源，更不要说判断临界资源是否就绪，若不就绪，则调用wait等待（正确的使用方式） 具体的条件变量的使用场景见生产者消费者模型，在恰当的场景下才能理解条件变量的正确使用方式。
