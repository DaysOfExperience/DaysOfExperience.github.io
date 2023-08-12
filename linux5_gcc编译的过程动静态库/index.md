# 

# C程序编译链接的过程

程序翻译的过程   源文件生成可执行程序的过程

## 1、预处理

预处理的作用/工作： **宏替换，头文件展开，条件编译，去注释。**

```bash
[yzl@VM-4-5-centos testdir]$ ll
total 4
-rw-rw-r-- 1 yzl yzl 314 Jul 30 17:14 test.c
[yzl@VM-4-5-centos testdir]$ gcc -E test.c -o test.i
[yzl@VM-4-5-centos testdir]$ ll
total 24
-rw-rw-r-- 1 yzl yzl   314 Jul 30 17:14 test.c
-rw-rw-r-- 1 yzl yzl 17030 Jul 30 17:15 test.i
```

gcc -E test.c -o test.i   选项“-E”，该选项的作用是让 gcc 在预处理结束后停止编译过程。选项“-o”是指定目标文件,“.i”文件为已经过预处理的C原始程序。

![img](https://img-blog.csdnimg.cn/87cac12be4ff46cc9097858fb2217873.png)

上图中左边为预处理之后的test.i文件，可以看出预处理做了哪些操作。 即宏替换，头文件展开，条件编译，去注释。

## 2、编译

在这个阶段中,gcc 首先要**检查代码的规范性、是否有语法错误等**，以确定代码的实际要做的工作,在检查无误后

gcc 把代码翻译成**汇编语言。**

gcc -S test.i -o test.s 进行程序的翻译，如果编译完成，就停下来，并生成test.s文件
-S 该选项只进行编译而不进行汇编,生成汇编代码。

```bash
[yzl@VM-4-5-centos testdir]$ gcc -S test.i -o test.s
[yzl@VM-4-5-centos testdir]$ ll
total 28
-rw-rw-r-- 1 yzl yzl   328 Jul 30 17:42 test.c
-rw-rw-r-- 1 yzl yzl 16991 Jul 30 17:50 test.i
-rw-rw-r-- 1 yzl yzl   745 Jul 30 17:51 test.s
```

![img](https://img-blog.csdnimg.cn/33dc57dd28f04d7886bd5b6aa53b0ffd.png)

## 3、汇编 

**汇编 ： 把汇编语言test.s 翻译为可重定向二进制目标文件 test.o，里面存储的是二进制目标代码。**

gcc -c test.s -o test.o  进行程序的翻译，如果汇编完成，就停下来，并生成test.o文件

```Bash
[yzl@VM-4-5-centos testdir]$ gcc -c test.s -o test.o
[yzl@VM-4-5-centos testdir]$ ll
total 32
-rw-rw-r-- 1 yzl yzl   328 Jul 30 17:42 test.c
-rw-rw-r-- 1 yzl yzl 16991 Jul 30 17:50 test.i
-rw-rw-r-- 1 yzl yzl  1576 Jul 30 17:53 test.o
-rw-rw-r-- 1 yzl yzl   745 Jul 30 17:52 test.s
```

利用hexdump工具，可以看到test.o里的二进制内容，此时test.o是一个**可重定向二进制目标文件**

<img src="https://img-blog.csdnimg.cn/7fb0d293ccf44289bef615d7e45726b5.png" alt="img" style="zoom: 50%;" />

<img src="https://img-blog.csdnimg.cn/438fc13ca29a47b480288d1e1a14d54e.png" alt="img" style="zoom:67%;" />

## 4、链接

成功编译之后，进入链接阶段。

**链接是将多个.o(linux).obj(win)可重定向二进制目标文件和静态库/动态库链接起来，生成可执行程序**

gcc test.o -o test      也可以  gcc test.c -o test

只是一个从test.o目标文件开始翻译，一个是从.c文件开始从头翻译。最终都生成可执行文件

## 总结

以.c程序为例，无论是Linux下，还是Windows的vs2019下，都要遵循这个程序翻译的过程：
预处理，编译，汇编，链接。我们的头文件在预处理之后，就已经在.c/.cpp文件中展开了。而gcc这个工具可以做到做完某个流程之后停下来。指令就是-E -S -c 分别对应完成预处理，完成编译，完成汇编。 生成的文件依次为.i .s .o    

## 动态库 / 静态库

任何一个C程序或者C++程序都离不开动态库或者静态库。在之前VS2019这样的IDE中，其实每次编译运行程序都用到了动态库，只是我们把关注点放在了编写代码上，对于程序的翻译过程没有研究，从而忽略了这个点。  

当我们在.c文件中使用printf这样的库函数时，需要包头文件，我们知道，头文件提供库函数的声明，而库函数的定义就是在动态库/静态库中。

静态库是指编译链接时,把库文件的代码全部加入到可执行文件中,因此生成的文件比较大,但在运行时可以独立运行，不再需要库文件。Linux下静态库后缀名一般为“.a” 动态库后缀名为“.so”

动态库与之相反,在编译链接时并没有把库文件的代码加入到可执行文件中,而是在程序执行时由运行时链接文件 加载库,这样可以节省系统的开销。动态库一般后缀名为“.so”, gcc 在编译时默认使用动态库。

![img](https://img-blog.csdnimg.cn/b226ee10a812453daf2fafb5e6ce9159.png)![img](https://img-blog.csdnimg.cn/6fe331517e6e4d7298f8a87a0f2a769f.png)

## gcc / g++

上面讲的是文件的编译链接过程，中间穿插了gcc的命令行使用方法。

gcc/g++ 默认形成的可执行程序是动态链接的！

```bash
[yzl@VM-4-5-centos testdir]$ ldd test
	linux-vdso.so.1 =>  (0x00007ffdd6fb0000)
	/$LIB/libonion.so => /lib64/libonion.so (0x00007f695bc3c000)
	libc.so.6 => /lib64/libc.so.6 (0x00007f695b755000)
	libdl.so.2 => /lib64/libdl.so.2 (0x00007f695b551000)
	/lib64/ld-linux-x86-64.so.2 (0x00007f695bb23000)
[yzl@VM-4-5-centos testdir]$ file test
test: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked (uses shared libs), for GNU/Linux 2.6.32, BuildID[sha1]=adae662af334b0bcb43e4bfacc2fe0559dd66250, not stripped
```

如上，.so就是Linux下动态库文件的后缀，下方的dynamically linked就是表明动态链接。

指定生成静态链接的可执行程序

加-static即可指定采用静态链接的方式编译链接源文件。可以明显看出静态链接的可执行程序比动态链接的可执行程序大很多。

```bash
[yzl@VM-4-5-centos testdir]$ gcc test.c -o test_static -static
[yzl@VM-4-5-centos testdir]$ ll
total 888
-rwxrwxr-x 1 yzl yzl   8384 Jul 30 21:04 test
-rw-rw-r-- 1 yzl yzl    328 Jul 30 17:42 test.c
-rw-rw-r-- 1 yzl yzl  16991 Jul 30 17:50 test.i
-rw-rw-r-- 1 yzl yzl   1576 Jul 30 17:53 test.o
-rw-rw-r-- 1 yzl yzl    745 Jul 30 17:52 test.s
-rwxrwxr-x 1 yzl yzl 861240 Jul 30 21:50 test_static
[yzl@VM-4-5-centos testdir]$ ldd test_static
	not a dynamic executable
[yzl@VM-4-5-centos testdir]$ file test_static
test_static: ELF 64-bit LSB executable, x86-64, version 1 (GNU/Linux), statically linked, for GNU/Linux 2.6.32, BuildID[sha1]=aab0a9e371c993a62c024b75d99c488c7bd1d3e0, not stripped
```



# 静态库与动态库的概念

1. 静态库（.a）：程序在编译链接的时候把静态库库的二进制代码拷贝到程序代码中。程序运行的时候将不再需要链接外部静态库。
   生成的可执行程序中自己的代码和静态库的代码是一体的，进程运行起来之后，全部都在地址空间的代码区。
2. 动态库（.so）：程序在运行的时候才去链接动态库的代码，不需要将动态库拷贝到程序的代码中，多个程序可以共享使用动态库的代码。

# 静态链接与动态链接

**静态链接**：

首先要明白的是：main函数所在的.o文件可以和其他.o文件一起编译（链接），生成可执行文件。**而静态库中有多个obj文件，将这些obj文件与main函数所在的obj文件一起编译，这个过程就叫做静态链接。**静态链接需要编译除自己之外的其他obj文件，生成的可执行文件会占用更多的空间

**动态链接**：

使用动态库的可执行文件含有一张**动态符号表**，表中包含了需要用到的函数符号。动态库也有一张**动态符号表**，包含了动态库的函数符号。
程序运行时，系统将动态库从磁盘加载到内存。同时，[动态链接器](https://www.zhihu.com/search?q=动态链接器&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra={"sourceType"%3A"article"%2C"sourceId"%3A"620900421"})根据符号表将程序与动态库之间建立绑定。一个动态库可以绑定多个程序的[动态符号表](https://www.zhihu.com/search?q=动态符号表&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra={"sourceType"%3A"article"%2C"sourceId"%3A"620900421"})，也就是可以被多个程序使用。使用动态库的程序，编译生成的可执行文件将占用更少的空间。

> 关于动态链接器的链接过程：符号解析 + 重定位
>
> 1. 动态链接器根据程序中的符号引用和动态库中的符号定义生成一张符号表，用来记录程序中每个符号在内存中的地址
>
> 2. 动态链接器遍历程序的重定位条目，根据符号表中的地址信息，修改程序中符号的引用，使它们指向正确的内存位置
>
> 3. 动态链接器调用动态库的初始化函数，完成必要的初始化工作
>
> 4. 动态链接器将控制权交给程序的入口点，使程序开始执行

# 结合虚拟地址空间 理解静态库和动态库的使用

> 静态库在你的源文件编译汇编，最后链接时，已经进入可执行程序内部了，生成可执行程序之后，后期执行此可执行程序，不再需要静态库文件。可以独立运行，所有代码都存在于进程地址空间的代码段中。
>
> 动态库和可执行程序可以分批加载，先运行可执行，若需要执行动态库中的函数代码时，从磁盘加载动态库代码到内存中，建立进程地址空间中共享区（栈堆之间的那部分区域），页表，内存中的动态库代码的映射关系。执行动态库代码时，从地址空间中的代码段跳转到共享区，然后通过页表映射执行内存中的动态库代码即可。
>
> 这也是为什么动态库的.o文件编译时，要加-fPIC选项，fPIC表示生成的.o文件中的地址，是一种与位置无关的地址，相对地址。编动态库代码的地址时，都是从0开始编址。
>
> 因为动态库的使用机制。动态库是一种共享库，动态库可以被多个进程共享，同时使用。所以动态链接使得可执行文件更小，节省了磁盘空间。操作系统采用虚拟内存机制允许物理内存中的一份动态库被要用到该库的所有进程共用，节省了内存和磁盘空间。    即，如果多个进程需要用到内存中的一个共享库，则利用虚拟地址空间机制，建立映射关系即可。    而静态库则不是，若有十个进程使用同一个静态库，同时运行，则内存中将会有10份一样的静态库代码。这无疑是一种内存资源浪费。
>
> 结合动态库是一种共享库，以及用gcc -c 汇编动态库源文件时要加-fPIC选项，这个选项表示动态库编址是一种相对地址。如何理解呢？
>
> 多个进程使用同一个动态库时，每一次动态库被加载到内存，映射到进程的地址空间的共享区后，映射的位置可能是不一样的。
> 但是，因为动态库内使用的是相对地址，所以，函数定位可以采用 库在地址空间中的起始虚拟地址 + 函数偏移量的方式，确定函数的虚拟地址。也就是动态库的编址为一种相对地址。
>
> 而静态库和自己的.o文件链接合并，使用的虚拟地址是一种绝对地址。
>
> 重在理解。

# gcc规则使用动静态库的规则：

1. gcc默认使用动态库，进行动态链接，若只有静态库，gcc只能对该库进行静态链接。

2. 若动静态库同时存在，则gcc默认使用动态库

3. 若动静态库同时存在，-static 即可指定使用静态库，进行静态链接。

# 制作静态库


如上，两个.c 两个.h    .h头文件包含库里面的函数声明 .c文件包含函数定义。

1. 将源文件，预处理，编译，汇编为.o可重定向二进制目标文件。
   `gcc -c mymath.c -o mymath.o`   `gcc -c myprint.c -o myprint.o`

2. 将所有.o文件归档为静态库（静态库文件）至此，静态库就生成好了
   `ar -rc *.o -o libplus.a`   （静态库文件的格式:  libxxx.a） 

3. 创建一个文件夹（目录），比如名为plus，将所有的头文件放在文件夹的include子目录下，将所有归档打包好的静态库放在plus的lib子目录下。   至此，这个plus文件就可以用来发布或者发给别人使用了。

# 制作动态库

还是基于 myprint.h myprint.c mymath.h mymath.c

1. `gcc -fPIC -c myprint.c -o myprint_d.o`
   `gcc -fPIC -c mymath.c -o mymath_d.o`
   编译.c文件生成.o可重定向二进制目标文件时，加-fPIC指令，表示与位置无关地生成.o文件（这块和动态库使用的本质有关）    

2. `gcc -shared mymath_d.o myprint_d.o -o libplus.so`
   不用ar，用gcc -shared即可。将.o文件合并归档打包生成libxxx.so动态库  （这里若不加-shared，就等于生成可执行程序了）

3. 可以将.h放在plus/include/下 将.so放在plus/lib/下，这样plus文件就可以用来发布或者传给使用库的人了。

# 使用静态/动态库

首先，无论是动态库还是静态库，源程序都需要包含头文件（*函数的声明*）。这样做可以使得程序编译成功，但是无法进行**链接**。

## 静态库

编译器在链接时，会在/usr/lib64这个默认路径下搜索源程序需要链接的库，在/usr/include这个路径下搜索需要使用的头文件。通常我们会用到[标准库](https://www.zhihu.com/search?q=标准库&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra={"sourceType"%3A"article"%2C"sourceId"%3A"620900421"})，所以编译器会默认在/usr/include和/usr/lib64这两个路径下查找头文件与标准库并且链接标准库。若源程序需要使用第三方库，就需要告知编译器：头文件所在路径、需要链接的库及其所在路径。

1. 若第三方库和第三方头文件已经添加到/usr/lib64和/usr/include目录下，那么编译源文件时，只要带上-l 选项，告知编译器要静态链接的库。比如`gcc test.c -l myfirst -o test`
   要注意的是：一般库文件名称都含有"lib"的前导和".so"或".a" 的后缀。比如[libmyfirst.so](https://www.zhihu.com/search?q=libmyfirst.so&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra={"sourceType"%3A"article"%2C"sourceId"%3A"620900421"})和libmyfirst.a，使用-l选项时，需要去除"lib"前导与".*"的后缀。比如libmyfirst.so写为myfirst
   （我们将库拷贝到系统默认路径下，就叫做库的安装。）

2. 若第三方库和第三方头文件没有添加到默认路径下，那么就需要使用-I (头文件所在路径)和-L (库文件所在路径)告知编译器库文件与头文件所在的位置。比如`gcc test.c -L ../lib-static/lib -I ../lib-static/include -l myfirst`依次为库文件所在路径，头文件所在路径，要链接的库。（不会污染系统默认路径下的环境）
3. Linux下，头文件gcc默认搜索路径：/usr/include  库文件默认搜索路径： /lib64 or  /usr/lib64

## 动态库

同样，使用动态库有两种方法，一种是简单的向默认路径添加库文件与头文件，编译时只使用-l选项。另一种使用-L、-I、-l三个选项进行编译。注意：对于后者，源程序依然可以编译成功，但是程序无法执行，一旦执行就会崩溃。

```bash
./haha_dong: error while loading shared libraries: libplus.so: cannot open shared object file: No such file or directory
```

原因是：虽然编译器知道源程序依赖的库文件所在路径，编译器根据库文件生成符号表。**但是程序加载时**，操作系统需要进行**动态链接** ，这个过程需要加载动态库到内存并修改符号表，使其指向正确的内存位置。**然而进行动态链接的动态链接器不知道动态库所在的位置（*磁盘中的位置*），动态链接的过程无法进行。**

运行时动态链接出错的解决方法：

1. 采用第一种方法，将动态库文件添加到默认路径下。因为程序运行过程中，需要动态库时，会自动去系统默认路径下查找动态库。这里不需要头文件，因为生成可执行的时候（预处理），头文件已经在你的源文件内部展开的。
2. 向环境变量LD_LIBRARY_PATH导入动态库路径（*用export设置*）。动态链接时，链接器除了会在默认路径下查找库文件，还会解析LD_LIBRARY_PATH中的路径，查找变量中的路径。不过系统一旦重新，环境变量重置，需要重新配置才能使程序正确运行。因为环境变量的初值是通过系统的配置文件获取的。
   `export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/home/yzl/aandso/plus2/lib`
3. 修改系统配置文件。/etc/ld.conf.so.d/目录下存储了系统的配置文件，每个配置文件存储了一个路径，将库文件所在的绝对路径为内容创建我们的配置文件，并添加到该路径下。添加完成使用sudo ldconfig指令，加载配置文件即可
4. 与第一种方式类似，向默认路径下以绝对路径的方式添加软连接。使用指令`sudo ln -s /home/xx/xxx/lib-dynamic/libmyfirst.so /usr/lib64/libmyfirst.so`   这两个路径以此为：库文件的绝对路径  需要在系统中创建的软链接文件

综上，其实所有方法的目的都是为了在程序运行过程中能够找到动态库，而使用静态库的程序只要链接好之后，就不用再找静态库了，因为静态库已经在可执行程序内部了，目标文件生成后，静态库删掉，程序照样可以运行。

# 程序加载时，链接器查找动态库文件的顺序？

动态链接器会根据以下顺序查找动态库文件，若都无法找到，动态加载失败，程序将崩溃

1. 在环境变量LD_LIBRARY_PATH中存储的指定路径中查找，使用export设置
2. 在/etc/ld.conf.so.d/目录下根据配置文件存储的路径查找，使用ldconfig使配置生效
3. 默认的/usr/lib64路径

其实一开始，动态链接器会根据程序硬编码的RPATH或RUNPATH选项指定的路径查找。不过我们一般都不会设置这两个选项，因为这会降低程序的可移植性和灵活性，所以我没有在上面提及。

# 怎么从进程地址空间的角度理解动态库？

加载程序时，系统就要为其分配进程地址空间，堆区往上栈区往下之间有一个区域：共享区，共享区有很多作用，其中一个是：**存储动态库函数的映射地址**。

映射地址只是虚拟地址，由于动态库函数的地址是在程序运行时才确定的，而共享区在程序加载时就需要进行创建。所以操作系统会在动态库函数地址确定之后修改页表，使共享区的映射地址指向正确的动态库函数地址。

## 运行使用动态库的程序的主要过程：

1. 系统将程序对应的可执行文件和需要使用的动态库加载到内存中（可分批加载）

2. 系统创建为可执行文件创建进程，建立进程地址空间，页表。同时调用动态链接器，进行符号解析和重定位

3. **当系统为动态库函数分配好了空间，系统将修改进程的页表，使共享区的动态库函数映射地址指向正确的地址**（建立进程地址空间中的共享区与内存中的动态库在页表中的映射）

4. 当进程调用动态库函数时，会跳转到共享区，获取函数的映射地址，经过页表的转换找到函数并且执行。函数执行完，程序会从共享区跳转回原来的区域，继续往下执行代码

# 对比动态库与静态库的优缺点

利用静态函数库编译成的文件比较大（因为整个函数库的所有数据都会被整合进目标代码中）

他的优点就显而易见了，即编译后的执行程序不需要外部的函数库支持，因为所有使用的函数都已经被编译进去了。

当然这也会成为他的缺点，因为如果静态函数库改变了，那么你的程序必须重新编译。

动态函数库所产生的可执行文件比较小。

由于函数库没有被整合进你的程序，而是程序运行时动态的申请并调用，所以程序的运行环境中必须提供相应的库。

动态函数库的改变并不影响你的程序，所以动态函数库的升级比较方便。

操作系统采用虚 拟内存机制允许物理内存中的一份动态库被要用到该库的所有进程共用，节省了内存和磁盘空间。

# 库依赖的查看

使用ldd命令来查看执行文件依赖于哪些库。








