# 

# 5_基础IO

## C标准库IO接口

```c++
int main()
{
    FILE* fp = fopen("log.txt", "w+");
    if(fp == NULL)
    {
        perror("fopen");
        return 1;
    }
    // C output
    fprintf(fp, "hello fprintf\n");
    fputs("hello fputs\n", fp);
    fwrite("hello fwrite\n", strlen("hello fwrite\n"), 1, fp);
    fclose(fp);

    fp = fopen("log.txt", "r");
    char arr1[64];
    char arr2[64];
    char arr3[64];
    fscanf(fp, "%s", arr1);
    fgets(arr2, sizeof(arr2), fp);
    fread(arr3, sizeof(arr3), 1, fp);
    printf("%s", arr1);
    printf("%s", arr2);
    printf("%s", arr3);
    fclose(fp);
    return 0;
}
```

1. 打开文件fopen，关闭文件fclose，output输出类函数 fwrite fputs fprintf，intput输入类函数fread fscanf fgets

2. 观察可以发现，这里的所有的文件IO的C库函数都和一个FILE\*的类型有关，比如fopen的返回值就是一个FILE\*类型，剩余所有文件IO类函数都和FILE\*有关。

**这里的FILE实际上是C标准库在语言层面用来描述一个文件的结构体。**

> fopen的打开模式：fopen的第二个参数为打开模式，有 r r+ w w+ a a+，不同参数对应不同的打开方式，比如r 为只读，r+为读写；w为写，文件存在则清空内容，不存在则创建，w+为读写，存在则清空，不存在则创建；a为追加模式写，也就是打开文件时会定位到文件末尾，不存在则创建，a+为追加模式读写，不存在则创建；
>
> 这里主要是为了对应后面要学习的文件IO的系统接口open，理解库函数和系统调用的关系

> C标准库默认打开的三个输入输出流
>
> 默认打开的三个输入输出流分别为：标准输入：stdin，标准输出：stdout，标准错误stderr
> 它们的类型都是FILE*，也就是文件结构体FILE的指针。
> 这三个文件，所对应的硬件外设分别为键盘，显示器，显示器（一切皆文件）
>
> ![img](https://img-blog.csdnimg.cn/4fd421b9bb4f4fb6ba444b64af0179a9.png)

> 理解当前路径
>
> `FILE *fopen(const char *path, const char *mode);`
>
> 上方为fopen的函数声明，第一个参数path为你要打开的文件的相对路径或绝对路径。
> 每一个进程运行起来的时候，都会记录自己当前所在的工作路径，称为cwd
>
> ![img](https://img-blog.csdnimg.cn/9d505dc747004bcfbd1734bc72556fff.png)
>
> 可以理解为，exe为这个进程对应可执行程序的绝对路径，可执行程序名为myfile。而工作路径cwd为这个进程执行起来的时候所处的路径。
> 如果我们写 fopen("log.txt", "w");   则这里的log.txt 自动转换为 cwd/log.txt   这里和可执行程序所处的位置有关。 

## IO接口 - 系统调用

```C++
int main()
{
    // blog，再次使用文件类系统接口
    int fd = open("log.txt", O_RDWR | O_CREAT | O_TRUNC, 0666);
    if(fd < 0)
    {
        perror("open");
        return 1;
    }
    const char* p = "hello write\n";
    write(fd, p, strlen(p));
    close(fd);

    fd = open("log.txt", O_RDWR | O_CREAT | O_APPEND, 0666);
    char arr[64];
    //memset(arr, '\0', sizeof arr);
    read(fd, arr, sizeof(arr));
    printf("%s", arr);
    close(fd);
    return 0;
}
```

有关文件IO的系统接口调用，简单学习一下open write read close，我们会发现它们和C标准库的文件IO函数相似度非常高，比如 fopen fwrite fread fclose

> open系统调用简介
>
> open函数int open(const char *pathname, int flags, mode_t mode);
> 第一个参数为要打开的文件的路径，相对路径或绝对路径
> 第二个参数为标记位，这里是通过宏的方式传递的，比如O_WRONLY 表示只写， O_RDONLY表示只读，O_RDWR表示读写，O_CREAT表示若文件不存在，创建该文件，O_TRUNC表示打开文件时，清空文件....
>
> 这里的open函数的第二个参数是以二进制标记位方式传递的，比如 O_RDWR为000001 O_WRONLY为000010，经过|运算符后，为0000011，组成一个int。函数内会根据每一个二进制标记位的01情况判断文件的打开方式
>
> 第三个参数为，创建文件时文件的权限设置，比如0666这里为8进制，转换为二进制表示110110110   --    rw-rw-rw-

## 文件描述符

通过文件IO的系统调用接口可以发现，**操作系统内核中标识一个进程打开的某一个文件使用的是一个整型int，称为文件描述符 - file descriptor - fd**  我们需要学习的是这个int是如何在操作系统中标识一个文件的。

### 文件描述符的本质

1. 进程打开一个文件，需要将文件加载到内存中 -> 文件分为进程打开的内存文件，和没有被任何进程打开的磁盘文件。
2. 每一个进程都可能打开一些文件，所以操作系统中就会有很多内存级文件。那么操作系统就必须管理这些文件。管理就代表着先描述，再组织（类似操作系统管理进程，用PCB描述进程）。而**Linux操作系统内核中描述一个被进程打开的文件使用的是 struct file结构体，它会存储文件相关的inode元信息**

<img src="https://img-blog.csdnimg.cn/306c769c9a394ca28727edb1a596bd56.png" alt="img"  />

操作系统中，有很多进程，有很多进程打开的文件。进程和进程打开的每一个文件需要匹配起来。
每一个进程的进程控制块PCB结构体 sturct task_struct中都有一个struct files_struct* flles的结构体指针数据成员
struct files_struct存储的就是这个进程打开的所有文件的各种信息。(Open file table structure)
struct files_struct结构体内有一个struct file* fd_array[] 指针数组。我们知道，操作系统内部描述一个打开的文件就是用的struct file。

所以，这个**数组就是存储着该进程打开的所有内存级文件在操作系统内部的file结构体的地址。而文件描述符fd就是这个数组的下标！**

操作系统内核使用int类型的文件描述符来标识该进程打开的每一个文件，实际上这个整型就是一个结构体（struct file）指针数组的下标。来实现进程和进程打开的每一个文件的配对。因此，那些系统调用接口也自然使用文件描述符来标识某一个文件。

> 而现在知道，文件描述符就是从0开始的小整数。当我们打开文件时，操作系统在内存中要创建相应的数据结构来描述目标文件。于是就有了file结构体。表示一个已经打开的文件对象。而进程执行open系统调用，所以必须让进程和文件关联起来。每个进程都有一个指针*files, 指向一张表files_struct,该表最重要的部分就是包涵一个指针数组，每个元素都是一个指向OS内描述某打开文件的结构体的指针！所以，本质上，文件描述符就是该数组的下标。所以，只要拿着文件描述符，就可以找到对应的文件

### 文件描述符的分配规则

```C++
int main()
{
    close(0);
    int f1 = open("log1.txt", O_WRONLY|O_CREAT|O_TRUNC, 0666);
    int f2 = open("log2.txt", O_WRONLY|O_CREAT|O_TRUNC, 0666);
    int f3 = open("log3.txt", O_WRONLY|O_CREAT|O_TRUNC, 0666);
    int f4 = open("log4.txt", O_WRONLY|O_CREAT|O_TRUNC, 0666);
    printf("%d\n", f1);
    printf("%d\n", f2);
    printf("%d\n", f3);           
    printf("%d\n", f4);
    close(f1);
    close(f2);
    close(f3);
    close(f4);
}
```

<img src="https://img-blog.csdnimg.cn/43928043ab7a4ddcaea7c900e6919a3b.png" alt="img"  />

fd 0 1 2 分别对应的是标准输入 标准输出 标准错误。关闭0号标准输入之后，打开四个文件，发现是0345，所以文件描述符的分配规则为：

**在files_struct结构体内部的文件结构体指针数组当中，找到当前没有被使用的最小的一个下标，作为新的文件描述符。**

## 三个默认打开的文件

Linux进程默认情况下会有3个缺省打开的文件描述符，分别是标准输入0， 标准输出1， 标准错误2. 0,1,2对应的物理设备一般是：键盘，显示器，显示器。（Linux一切皆文件，外设在OS内也是对应一个文件，通过这个文件来与外设进行交互）

其实这和C语言默认打开的三个输入输出流是相对应的。那么，是谁配合的谁呢？当然是语言层配合系统层喽<img src="https://img-blog.csdnimg.cn/23e61b06278244a28d747adbcc5f646c.png" alt="img" style="zoom: 80%;" />

## C库函数和系统调用接口的关系

文件存在磁盘中，磁盘为硬件，要想读写文件，就要向硬件写入，只有操作系统可以通过驱动直接访问并管理硬件
操作系统给出了一些系统调用接口，比如文件IO类系统调用接口，open close read write，可以供给打开，关闭，读写文件
C标准库作为上层语言，要想访问硬件，读写文件，就必须使用操作系统提供的系统调用接口

所以，C标准库函数实际上就是系统调用接口的封装，比如fopen 封装open fclose封装close... 
比如，fopen("log.txt", w); 封装 open("log.txt", O_WRONLY | O_CREAT | O_TRUNC, 0666);
再比如，fopen("log.txt", a); 封装 open("log.txt", O_WRONLY | O_CREAT | O_APPEND, 0666);

操作系统中，描述一个被打开的文件，用的是struct file，而将进程和打开的文件配对起来使用的是文件描述符fd，系统调用接口中也都是用的是文件描述符来标识文件

而C标准库在语言层面描述一个文件使用的是FILE结构体。综上，FILE结构体中一定封装了文件描述符fd

stdout 这个FILE*指向的FILE内就有一个数据成员\_fileno，这就是系统调用接口使用的文件描述符fd，在C标准库中定义的FILE结构体内，名为\_fileno。

```C++
fprintf(stdout, "%d\n", stdin->_fileno);      // 0
fprintf(stdout, "%d\n", stdout->_fileno);     // 1
fprintf(stdout, "%d\n", stderr->_fileno);     // 2
```

## 重定向

### 输出重定向

```C++
int main()
{
    close(1);  // 1就是标准输出,stdout->fileno = 1
    int fd = open("log.txt", O_CREAT|O_TRUNC|O_WRONLY, 0666);  // 写打开，且清空
    if(fd < 0)
    {
        perror("open");
        return 1;
    }
    // 现在其实fd就是1
    printf("fd:%d\n", fd);
    fprintf(stdout, "hello fprintf\n");
    fwrite("hahaha\n", strlen("hahaha\n"), 1, stdout);
    // 缓冲区相关
    fflush(stdout);
    close(fd);
    return 0;
}
```

运行结果：显示器上没有打印内容，数据全部输出到了文件log.txt内。

代码分析：先把1号文件描述符close了，再打开一个文件时，根据文件描述符的分配规则，选取的是文件描述符表中最小的且没有被占用的下标。所以给log.txt分配的文件描述符就是1。

stdout为标准输出，类型为FILE\*，而FILE是语言层面的，内部封装的fd为1，这是语言层面没有改变的。
此时，在OS内核中的该进程对应的文件描述符表的1号下标处存储的struct file\*已经不是显示器的内存文件结构体的地址了(struct file\*)，而变为了log.txt对应的内核struct file结构体的地址，所以，printf fprintf fwrite都会将数据output到log.txt内，因为1号文件描述符处的指针指向的是log.txt对应的struct file结构体。 **这就是输出重定向！**

同理，如果把O_TRUNC改为O_APPEND，**就变为了追加重定向。**

### 输入重定向

```C++
close(0);
int fd = open("log.txt", O_RDONLY);
char arr[64];
fgets(arr, sizeof(arr), stdin);
printf("%s", arr);
close(fd);
```

此时log.txt的fd = 0，文件描述符表的0下标处存储的是log.txt的内核struct file结构体的地址。

所有本应从标准输入读取的数据，变为了从其他文件中读取，**即输入重定向。**

### 重定向的本质

![img](https://img-blog.csdnimg.cn/43cfb2cc6b6c41ae87e5ec4938fa24a1.png)

凡是往原本1号文件描述符对应的标准输出中写的内容，都写到了其他文件中，不再写到标准输出！这就是输出重定向。 

凡是从原本0号文件描述符对应的标准输入读取的内容，变为了从其他文件中读取，即输入重定向。

**重定向的本质：在OS内部，更改fd对应的struct file*指针内容的指向。**

### dup2系统调用

上方利用文件描述符的分配规则，改变0号或1号文件描述符处内容，使其指向其他文件，完成重定向。
事实上，系统调用中有一个dup2系统调用可以更方便地完成这一操作。

```C++
#include <unistd.h>
int dup2(int oldfd, int newfd);

//dup2() makes newfd be the copy of oldfd, closing newfd first if necessary, but note the following:
//*  If oldfd is not a valid file descriptor, then the call fails, and newfd is not closed.
//*  If oldfd is a valid file descriptor, and newfd has the same value as oldfd, then dup2() does nothing, and returns newfd.
```

介绍：**dup2是一个系统调用接口，将newfd文件描述符下标处的内容改为oldfd文件描述符下标处内容的拷贝。**使newfd和oldfd文件描述符指向同一个文件。（如果newfd处的文件是打开状态，则拷贝之前会先关闭掉newfd处的文件）

### dup2示例代码

```C++
int main(int argc, char* argv[])
{
    int fd = open("log.txt", O_CREAT | O_TRUNC | O_WRONLY, 0666);
    dup2(fd, 1);
    // 此时1和fd处的内容指向同一个文件
    printf("fd:%d\n", fd);
    printf("test message\n");
    int n = 5;
    while(n--)
    {
        fprintf(stderr, "%d\n", n);
        sleep(1);
    }
    close(fd);
    fflush(stdout);
    sleep(5);
    printf("after close\n");   
    return 0;
}
```

![img](https://img-blog.csdnimg.cn/50d79d9d78ea40d6a8891648f9491070.png)

这段代码，一方面是使用dup2来演示输出重定向。另一方面是探究两个问题。
1、 1 和 fd都指向同一文件，那么close(fd)之后，文件描述符为1处的文件访问还有效吗？
2.、缓冲区问题<img src="https://img-blog.csdnimg.cn/6fee9648fb4742a4a770d651838694a8.png" alt="img" style="zoom:80%;" />

事实证明，**fd和1文件描述符指向同一文件，在close(fd)之后，fd = 1处的文件访问依然有效，所以后面的"after close"成功输出到log.txt内。**

缓冲区问题：**重定向导致刷新策略改变**，前两个printf中的\n失效，必须fflush。才能将语言层缓冲区内的数据刷新到fd=1对应的普通文件中

若将第一个fflush注释掉，则

<img src="https://img-blog.csdnimg.cn/ef0225a6228348779de1ffd72a30d901.png" alt="img" style="zoom: 80%;" />

前10s，数据都存储在stdout缓冲区内。最后进程退出强制刷新stdout语言级别缓冲区，使数据刷新到stdout -> fd = 1 -> log.txt文件内。（这里是缓冲区问题，已经不属于上方研究的dup2的范畴了，具体缓冲区问题见下方）

其实现在看来很简单，只是重定向导致fd=1处不再是标准输出（行刷新）而是普通文件（满刷新）

## 缓冲区

### 缓冲区是什么

缓冲区就是一段内存空间

### 为什么要有缓冲区？

比如用户使用fprintf往文件中输出数据时，实际上需要向磁盘中写入。如果每次fprintf都直接将数据通过操作系统写入到硬件外设内，则会非常慢，原因是内存 与 外设的处理数据速度差距非常大并且与外设预备IO的过程是非常耗时间的，可能是1000倍的差距，导致我们往硬件写入时，需要非常长的处理和响应时间。

因此，我们在内存中设计一段缓冲区，将需要多次与外设IO的数据先暂存在内存的缓冲区内，**等待刷新条件达成时**，再一次性全部刷新写入到外设中，这样减少了与外设的访问IO次数，**则可以提高整机的效率。更重要的是提高用户的响应速度**，因为只需要将数据拷贝到内存的另一区域即可！

总而言之：提高效率，提高用户的响应速度。

### 缓冲区的刷新策略

通常刷新策略分为1. 立即刷新 2. 行刷新（行缓冲）3. 满刷新（全缓冲） 很容易理解的是，全缓冲的效率是最高的，因为意味着最少的外设IO次数，所以，所有的设备文件都倾向于全缓冲。但是，具体的刷新策略需要考量具体的外设使用需求。

通常，显示器的内存文件采用行缓冲（遇到\n则刷新（指的是刷新到外设中，这里就是显示器中）），磁盘文件通常采用全缓冲（缓冲区满了则刷新）。
同时，我们也可以通过fflush强制刷新，或者进程结束时，缓冲区内的数据也会强制刷新。

### 探究缓冲区的示例代码

```C++
// 示例代码1
    printf("hello printf\n");
    fprintf(stdout, "hello fprintf\n");
    fputs("hello fputs\n", stdout);
    write(1, "hello write\n", strlen("hello write\n"));
    fork();

// 示例代码2 
    int fd = open("log.txt", O_WRONLY | O_TRUNC | O_CREAT, 0666);
    dup2(fd, 1);
    printf("hello printf\n");
    fprintf(stdout, "hello fprintf\n");
    fputs("hello fputs\n", stdout);
    write(1, "hello write\n", strlen("hello write\n"));
    //fflush(stdout);
    fork();
 
// 示例代码3
    int fd = open("log.txt", O_WRONLY | O_TRUNC | O_CREAT, 0666);
    dup2(fd, 1);
    printf("hello printf\n");
    fprintf(stdout, "hello fprintf\n");
    fputs("hello fputs\n", stdout);
    write(1, "hello write\n", strlen("hello write\n"));   
    fflush(stdout);
    fork();
```

#### 运行结果

示例代码1：显示器中显示全部内容
示例代码2：log.txt文件中C标准库的output函数的输出内容有两份，系统调用接口write函数的输出内容有一份
示例代码3：文件中显示全部内容

#### 结论

FILE结构体是语言层C标准库提供的，C标准库在每一个FILE结构体内都有一个语言层面的缓冲区（可以简单理解为一个数组，一段内存空间），缓冲区刷新策略由FILE结构体内封装的fd指向的文件类型决定，如果是显示器，则为行缓冲，如果是磁盘文件，则为全缓冲。

操作系统内核中在每一个struct file结构体内都有一个内核级缓冲区，事实上，C标准库IO函数fprintf，fputs，是将数据先存储到FILE结构体内的语言层面缓冲区，等待刷新条件达成或者发生强制刷新，再将数据刷新到OS内核中struct file内的内核级缓冲区中，最终由OS将数据从内核struct file内的缓冲区output到外设硬件中。（我们不关心内核缓冲区到硬件的刷新，只关心语言层面缓冲区到内核缓冲区的刷新，也就是C标准库的缓冲区何时刷新）

#### 示例代码的分析

> 示例代码1中，stdout的类型为FILE*，指向FILE为标准输出，内部封装了语言层面缓冲区，此时fd = 1对应的外设为显示器，故刷新策略为行缓冲。 printf fprintf fputs将数据刷新到stdout指向的FILE内的缓冲区内，因为此时对应的是显示器，所以为行刷新，于是，当fork时，数据已经刷新到OS内了。 而对于write，这是一个系统调用接口，是直接将数据写入到OS中的，对于write，并没有语言层面的缓冲区。因此，最终所有数据都只有一份，写入到了显示器中。
>
> 示例代码2中，由于dup2，使得stdout对应的fd实际指向的文件为磁盘文件，所以，刷新策略由行缓冲变为全缓冲。致使fork前，数据并没有由FILE内的语言层缓冲区刷新到OS内。fork创建子进程，之后，进程结束，会强制刷新缓冲区，此时缓冲区内的数据需要刷新到OS内，清空缓冲区，即一种写操作，故缓冲区内的数据发生写时拷贝，父子进程各一份。最后父子进程都将数据刷新到OS内，所以最终log.txt文件中，C标准库的IO函数的数据有两份。而对于write，是直接将数据写入到OS内的，所以只有一份。
>
> 示例代码3中，如果在fork之前，fflush(stdout)，则会强制将stdout指向的FILE内的缓冲区数据刷新到OS内（刷新策略外的强制刷新）。所以，在fork创建子进程时，父进程的C标准库的缓冲区内已经没有缓冲数据了（因为已经刷新到log.txt的内核struct file结构体内的缓冲区中了，属于OS，不属于进程），自然不会发生写时拷贝后有两份数据等情况了。

上方缓冲区相关知识，揭示了缓冲区的概念，缓冲区存在的意义，缓冲区的刷新策略，语言层缓冲区和内核级缓冲区。那个示例代码也不是很复杂，只是对不同的文件/设备有不同的刷新策略，当发生重定向时，刷新策略也随之改变，致使数据存留在了语言层，即C标准库提供的缓冲区内，属于进程的数据。再结合fork创建子进程，进程结束时强制刷新缓冲区，以及写时拷贝。就产生了最终现象~

## 一切皆文件 - Linux设计哲学

一切皆文件指的是，在操作系统层面，所有的外设硬件，以及磁盘内的文件，都可以看作文件，都可以统一以文件来处理。这是体现在操作系统的软件设计层面的。

如何实现一切皆文件呢？

Linux操作系统是用C语言写的，它要将所有外设都看作统一的文件，就需要以统一的方式来描述和管理。
最好的就是使用结构体，这里可以通过让结构体：struct file内存储一些属性数据，函数指针。
**从而使用C语言结构体达到面向对象机制和运行时多态。**

<img src="https://img-blog.csdnimg.cn/6e4c6d8b0ab0413eb2a6f1f413367914.png" alt="img" style="zoom:80%;" />

上方仅是一个简单示例。

我们可以将磁盘，显示器，键盘，网卡，显卡等硬件外设都用这个struct file结构体在操作系统内部描述管理起来。属性数据存储在结构体的数据成员中。而函数指针，可以指向不同外设的具体read write函数实现， 因为底层不同的外设，具体读写方法肯定是不同的，但是我们可以通过函数指针机制，达到在struct file的使用管理层面上，调用读写方法的方式都相同的效果。

这样一来，在操作系统层面来看，硬件之间已经没有任何差别了，都可以统一看作struct file，且可以使用struct file内的函数指针，调用对应外设的具体的读写方法。

这就是一切皆文件，这种机制称为VFS，Virtual File System，虚拟文件系统。
