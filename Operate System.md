# 用户态与核心态
![](https://pic3.zhimg.com/80/v2-c9263b0dfebc2a68731cd198081fee46_720w.png)

因为不同指令之间存在权限差异，CPU分为了用户态和内核态，内核态用来运行特权指令和内核程序。所以对应的操作系统分成非内核功能和内核功能，其中内核是操作系统最主要最核心的部分

![](https://guide-blog-images.oss-cn-shenzhen.aliyuncs.com/2020-8/L181kk2Eou-compress.jpg)

# 三种上下文切换
进程上下文切换

一次系统调用的过程，其实是发生了两次 CPU 上下文切换。（用户态-内核态-用户态）

线程上下文切换

如果前后两个线程属于不同进程。此时，因为资源不共享，所以切换过程就跟进程上下文切换是一样。
如果前后两个线程属于同一个进程。只需要切换线程的私有数据、寄存器等不共享的数据

中断上下文切换

中断处理会打断进程的正常调度和执行，转而调用中断处理程序，响应设备事件。收到信号处理时触发

# BIO\NIO\同步IO\异步IO
IO操作分2个步骤，请求IO和实际IO
**请求 IO :** 进程向内核发出 IO 请求, 内核返回数据报文是否已准备好
**实际 IO :** 从内核向进程复制数据

**举例 :** 对于一个套接字上的输入操作，第一步通常涉及等待数据从网络中到达。当所有等待分组到达时，它被复制到内核中的某个缓冲区。第二步就是把数据从内核缓冲区复制到应用程序缓冲区
> 用户空间和内核空间
    1. 操作系统为了支持多个应用同时运行，需要保证不同进程之间相对独立（一个进程的崩溃不会影响其他的进程 ， 恶意进程不能直接读取和修改其他进程运行时的代码和数据）。 因此操作系统内核需要拥有高于普通进程的权限， 以此来调度和管理用户的应用程序。
    2. 于是内存空间被划分为两部分，一部分为内核空间，一部分为用户空间，内核空间存储的代码和数据具有更高级别的权限。内存访问的相关硬件在程序执行期间会进行访问控制（ Access Control），使得用户空间的程序不能直接读写内核空间的内存

==阻塞, 非阻塞是针对第一阶段而言
同步, 非同步是针对第二阶段而言==

![](https://static.oschina.net/uploads/img/201604/21095604_vhHX.png)

**1. 阻塞IO**
![](https://pic2.zhimg.com/80/e83d68da03da2e8c1568b4b4b630edfd_1440w.jpg?source=1940ef5c)

**2. 非阻塞IO**
![](https://pic1.zhimg.com/80/4bc31cab27a9a732ab7d1ba9e674ed64_1440w.jpg?source=1940ef5c)
进程把一个套接字设置成非阻塞是在通知内核，当所请求的I/O操作非得把本进程投入睡眠才能完成时，不要把进程投入睡眠，而是返回一个错误。==recvfrom总是立即返回==

**3. IO 多路复用**
![](https://pic2.zhimg.com/80/b1ec6a4f16844a27c175d5a6a94cd7f8_1440w.jpg?source=1940ef5c)
I/O多路复用的函数也是阻塞的，但是其与以上两种还是有不同的，I/O多路复用是阻塞在select，epoll这样的系统调用之上，而没有阻塞在真正的I/O系统调用如recvfrom之上。

**4. 异步 IO**
![](https://pic1.zhimg.com/80/5819fd0fdff2bd4fdc9652291aca1831_1440w.jpg?source=1940ef5c)
工作机制是告知内核启动某个操作，并让内核在整个操作（包括将数据从内核拷贝到用户空间）完成后通知我们

## 阻塞原理
阻塞是指进程在发起了一个系统调用（System Call） 后， 由于该系统调用的操作不能立即完成，需要等待一段时间，于是内核将进程挂起为等待 （waiting）状态， 以确保它不会被调度执行， 占用 CPU 资源

> 系统调用（system call）
    system call 是操作系统提供给应用程序的接口。 用户通过调用 systemcall 来完成那些需要操作系统内核进行的操作， 例如硬盘， 网络接口设备的读写等。

**工作队列:**
![](https://cdn.nlark.com/yuque/0/2020/jpeg/181910/1608369221075-ac2ec9f0-c364-40a9-a8d6-ec7abae56599.jpeg)
操作系统为了支持多任务, 把进程分为 "运行" 和 "等待" 等几种状态
如上图的进程 A, 在执行到 recv 整个 IO 阻塞方法时, 操作系统会将其转换为 "等待" 状态, 等到接收到数据, 再转换为 "运行" 的状态

**等待队列**
当进程 A 执行创建 socket 的语句时, 操作系统会创建一个由文件系统(图中的文件列表)管理的 socket 对象
![](https://cdn.nlark.com/yuque/0/2020/jpeg/181910/1608369221524-91520e15-e756-4a79-988c-cf8be8068f90.jpeg)
一个 socket 对象包括收发缓冲区, 和等待列表, 当进程 A 执行到 recv 方法的时候, 会把该进程放入对应的 socket 对象的等待队列中(==对于阻塞 IO 来说==)
![](https://cdn.nlark.com/yuque/0/2020/jpeg/181910/1608369221078-f8a161c0-5ca2-47dc-ba2b-c044e41c7de1.jpeg)


## IO 多路复用
### select
select 的伪代码
```c
int s = socket(AF_INET, SOCK_STREAM, 0); 
bind(s, ...) 
listen(s, ...) 
int fds[] = 存放需要监听的socket 
while (1) { 
    int n = select(..., fds, ...) 
    for(int i=0; i < fds.count; i++){ 
        if(FD_ISSET(fds[i], ...)){ 
            //fds[i]的数据处理 
        } 
    } 
}
```
#### select 的流程
1. 当进程 A 需要监听多个 socket 的时候, 便把该进程加入到需要监听的 socket 的等待队列里
![](https://cdn.nlark.com/yuque/0/2020/jpeg/181910/1608369221394-15a15def-7d6a-40b9-9389-64dff4a0c68a.jpeg)
2. 当其中一个 socket 接收到数据, 操作系统将唤醒在该 socket 的等待队列中的线程 A
![](https://cdn.nlark.com/yuque/0/2020/jpeg/181910/1608369221471-7df6eef9-f66e-4729-8f1a-773e465492c9.jpeg?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_10%2Ctext_eXVsb25nc3Vu%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)
3. 线程A 被唤醒后, 由于不知道是哪个 socket 接收到数据, 因此需要遍历一遍 socket 列表, 即伪代码里的 fds

**缺点:**
1. 每次唤醒后, 需要遍历一遍 socket 列表来确认哪个 socket 对象接收到数据(因此, 规定最大只能监控 1024 个 socket 对象)
2. 每次执行完, 需要重新将进程A 加入到各个 socket 对象的等待队列中

#### epoll
```c
int s = socket(AF_INET, SOCK_STREAM, 0); 
bind(s, ...) 
listen(s, ...) 
int epfd = epoll_create(...); 
epoll_ctl(epfd, ...); //将所有需要监听的socket添加到epfd中 
while(1) { 
    int n = epoll_wait(...) 
    for(接收到数据的socket){ 
        //处理 
    } 
}
```
#### epoll 流程
1. 创建 epoll 对象, 当进程A 执行 epoll_creat 方法时, 内核会创建一个 eventpoll 对象(伪代码里的 epfd)
![](https://cdn.nlark.com/yuque/0/2020/jpeg/181910/1608369221115-cdca633e-0069-4b49-aac7-884c5e279f25.jpeg)
eventpoll 对象也是文件系统的一员
2. 维护监视列表
![](https://cdn.nlark.com/yuque/0/2020/jpeg/181910/1608369221111-214398cf-4adc-479b-aeeb-2351cfc315a6.jpeg)
==内核会将 eventpoll 添加到进程 A 需要监听的 Socket 的等待队列中==
当 Socket 收到数据后，中断程序会操作 eventpoll 对象，而不是直接操作进程
3. 接收数据
当 socket 接收到数据之后, 中断程序会给 eventpoll 对象中的就序列表(rdlist) 添加 socket 引用
![](https://cdn.nlark.com/yuque/0/2020/jpeg/181910/1608369221169-416a5e91-600c-47f2-b004-0f6a2c192e71.jpeg)
4. 阻塞和唤醒进程
当进程 A 运行到 epoll_wait() 方法时, 操作系统会把进程A 放在 eventpoll 对象的等待队列里, 然后阻塞进程 A
![](https://cdn.nlark.com/yuque/0/2020/jpeg/181910/1608369221241-a82b67e8-8b22-4b1a-aaea-da9f5b333596.jpeg)
当 socket 接收到数据, 中断程序一方面将 socket 放入 rdlist, 另一方面将 eventpoll 中的进程 A 唤醒进入运行状态
![](https://cdn.nlark.com/yuque/0/2020/jpeg/181910/1608369221108-acb69d23-01d1-4a50-a177-84193b28effd.jpeg?)

##### epoll水平和边缘触发的区别?
在进程执行 epoll_wait() 操作时, 水平触发和边缘触发对进程唤醒从策略不同
1. 边缘触发 (edge trigger)
    **读操作**
    1. 在 buffer 由不可读变为可读的时候  
    ![](http://blog.chinaunix.net/attachment/201406/3/28541347_14018059421VJj.png)
    2. 在 buffer 有新数据到达时
    ![](http://blog.chinaunix.net/attachment/201406/3/28541347_14018059521U1k.png)

    **写操作**
    1. 在 buffer 由不可写变为可写的时候
    2. 当有旧数据被发送走时，即buffer中待写的内容变少得时候
    ![](http://blog.chinaunix.net/attachment/201406/3/28541347_1401805980jh7p.png)

2. 水平触发 (level trigger)
    LT模式下进程被唤醒（描述符就绪）的条件就简单多了，它包含==ET模式的所有条件==。此外，还有更普通的情况LT可以被唤醒，而ET则不理会，这也是我们需要注意的情况。

    **读操作**
    当buffer中有数据，且数据被读出一部分后buffer还不空的时候，即buffer中的内容减少的时候，LT模式返回读就绪。==只要还可读, 就返回读就绪==
    ![](http://blog.chinaunix.net/attachment/201406/3/28541347_14018060158h95.png)
    **写操作**
    当buffer不满，又写了一部分数据后扔然不满的的时候，即由于写操作的速度大于发送速度造成buffer中的内容增多的时候，LT模式会返回就绪
    ![](http://blog.chinaunix.net/attachment/201406/3/28541347_1401806029se68.png)

边缘触发通知之后进程必须立即处理事件, 而水平触发不是. ET下次再调用 epoll_wait() 时不会再得到事件到达的通知。很大程度上减少了 epoll 事件被重复触发的次数, 但也因此只能使用非阻塞 IO, 以避免一个阻塞读写操作, 把其他 socket 操作饿死

# 文件描述符
一个 Linux 进程可以打开成百上千个文件，为了表示和区分已经打开的文件，Linux 会给每个文件分配一个编号（一个 ID），这个编号就是一个整数，被称为**文件描述符**（File Descriptor）
一个 Linux 进程启动后，会在内核空间中创建一个 PCB 控制块，PCB 内部有一个文件描述符表（File descriptor table），记录着==当前进程所有可用的文件描述符==，也即当前进程所有打开的文件。

除了文件描述符表，系统还需要维护另外两张表：
- 打开文件表（Open file table）
- i-node 表（i-node table）

==文件描述符表每个进程都有一个，打开文件表和 i-node 表整个系统只有一个==

![](http://c.biancheng.net/uploads/allimg/190410/1-1Z4101H45S13.gif)

通过文件描述符，可以找到文件指针，从而进入打开文件表。该表存储了以下信息：
- 文件偏移量，也就是文件内部指针偏移量。调用 read() 或者 write() 函数时，文件偏移量会自动更新，当然也可以使用 lseek() 直接修改。
- 状态标志，比如只读模式、读写模式、追加模式、覆盖模式等。
- i-node 表指针。

然而，要想真正读写文件，还得通过打开文件表的 i-node 指针进入 i-node 表，该表包含了诸如以下的信息：
- 文件类型，例如常规文件、套接字或 FIFO。
- 文件大小。
- 时间戳，比如创建时间、更新时间。
- 文件锁

说明
- 在进程 A 中，文件描述符 1 和 20 都指向了同一个打开文件表项，标号为 23（指向了打开文件表中下标为 23 的数组元素），这可能是通过调用 dup()、dup2()、fcntl() 或者对同一个文件多次调用了 open() 函数形成的。
- 进程 A 的文件描述符 2 和进程 B 的文件描述符 2 都指向了同一个文件(73)，这可能是在调用 fork() 后出现的（即进程 A、B 是父子进程关系），或者是不同的进程独自去调用 open() 函数打开了同一个文件，此时进程内部的描述符正好分配到与其他进程打开该文件的描述符一样。
- 进程 A 的描述符 0 和进程 B 的描述符 3 分别指向不同的打开文件表项，但这些表项均指向 i-node 表的同一个条目（标号为 1976）；换言之，它们指向了同一个文件。发生这种情况是因为每个进程各自对同一个文件发起了 open() 调用。同一个进程两次打开同一个文件，也会发生类似情况

## inode
硬盘的最小存储单位是扇区(Sector)，块(block)由多个扇区组成。文件数据存储在块中。块的最常见的大小是 4KB，约为 8 个连续的扇区组成（每个扇区存储 512 字节）。一个文件可能会占用多个 block，但是一个块只能存放一个文件。

虽然，我们将文件存储在了块(block)中，但是我们还需要一个空间来存储文件的 元信息 metadata ：==如某个文件被分成几块、每一块在的地址、文件拥有者，创建时间，权限，大小等==。这种 存储文件元信息的区域就叫 inode，译为索引节点：i（index）+node。 每个文件都有一个 inode，存储文件的元信息。

可以使用 `stat` 命令可以查看文件的 inode 信息。每个 inode 都有一个号码，Linux/Unix 操作系统不使用文件名来区分文件，而是使用 inode 号码区分不同的文件。

简单来说：inode 就是用来维护某个文件被分成几块、每一块在的地址、文件拥有者，创建时间，权限，大小等信息。

简单总结一下：
- inode ：记录文件的属性信息，可以使用 stat 命令查看 inode 信息。
- block ：实际文件的内容，如果一个文件大于一个块时候，那么将占用多个 block，但是一个块只能存放一个文件。（因为数据是由 inode 指向的，如果有两个文件的数据存放在同一个块中，就会乱套了）

![](https://snailclimb.gitee.io/javaguide/docs/operating-system/images/%E6%96%87%E4%BB%B6inode%E4%BF%A1%E6%81%AF.png)

### 扇区，磁盘块，页
- 扇区(Sector): 硬盘的读写以扇区为基本单位, 一般是 512 字节 
- 块(Block): 文件系统的最小读写单位, 一般一个块由8个扇区组成, 即 4K 字节
- 页 (page): 内存的最小操作单位, 页的大小通常为磁盘块大小的 2^n 倍, 通常是 4K 字节, 与块一致

### linux 文件类型
- 普通文件（-） ： 用于存储信息和数据， Linux 用户可以根据访问权限对普通文件进行查看、更改和删除。比如：图片、声音、PDF、text、视频、源代码等等。
- 目录文件（d，directory file） ：目录也是文件的一种，用于表示和管理系统中的文件，目录文件中包含一些文件名和子目录名。打开目录事实上就是打开目录文件。
- 符号链接文件（l，symbolic link） ：保留了指向文件的地址而不是文件本身。
- 字符设备（c，char） ：用来访问字符设备比如硬盘。
- 设备文件（b，block） ： 用来访问块设备比如硬盘、软盘。
- 管道文件(p,pipe) : 一种特殊类型的文件，用于进程之间的通信。
- 套接字(s,socket) ：用于进程间的网络通信，也可以用于本机之间的非网络通信


# 软硬链接
- 硬链接: 指针, 与源文件有着一样的 inode 值, 同样指向物理硬盘的一个区块. 操作(增删改查)硬连接和操作源文件没有任何区别. 文件系统会维护一个引用计数, 只要引用大于 0, 就不会真正物理删除文件
- 软链接: 指向源文件的指针, 与源文件指向不一样的 inode, 实际上保存了一个绝对路径, 当操作软链接时, 会自动替换成操作源文件

```shell
$ ln -s myfile soft
$ ls -li

25869085 -rw-r--r--  2 unixzii  staff  36  7  8 17:45 hard
25869085 -rw-r--r--  2 unixzii  staff  36  7  8 17:45 myfile
25869216 lrwxr-xr-x  1 unixzii  staff   6  7  8 17:47 soft -> myfile
```

尝试删除源文件
```shell
$ rm myfile

$ cat hard

This is a plain text file.
New line

$ cat soft

cat: soft: No such file or directory
```

当我们向软连接新增东西时
```shell
$ echo "Something" >> soft
$ ls

hard   myfile soft
```

此时, 源文件将恢复(==但此时是新文件, 与硬链接指向不同的 inode==)


# 进程/线程/协程
**进程是资源分配的最小单位，线程是CPU调度的最小单位**

没有进程的操作系统只能运行单个程序，对于多个程序在内存分配的位置和大小都需要管理，所以进程是系统资源的分配单位。尤其引入线程之后，进程作为除了CPU之外系统资源的分配单元（如打印机，内存空间）。线程只占了CPU，剩下还是以进程为单位划分

引入进程可以使各个程序之间并发执行，但是对于一个程序内部来说所有的功能如果都限制为顺序执行显然对用户体验不好。为此我们引入线程的概念，来增加进程内部的并发度，比如我们用QQ发送文件的时候还可以聊天

![](https://pic4.zhimg.com/80/v2-c9b5a1f7585d8873b640b7391c51b443_720w.jpg)

## 进程给线程划分的资源
### 线程共享的资源
1. 进程代码段
2. 进程的文件描述符
3. 进程的共有数据(可以用于线程通信)

### 线程私有的资源
1. 线程 id
2. cpu 寄存器
3. 堆栈
4. 错误返回码
5. 线程优先级
6. 信号屏蔽码

### 系统调用与进程切换
#### 系统调用
当进程需要调用硬件(比如硬盘)时, 需要由系统内核处理

这个过程就会发生 cpu 上下文切换
1. 保存 cpu 寄存器原来用户态的指令位置 (pc 寄存器)
2. 为了执行内核态代码, cpu 寄存器需要更新为内核态指令的位置
3. 跳转到内核态执行内核任务
4. 恢复进程的 pcb, 再切换为用户态

因此一次系统调用, 发生两次 cpu 上下文切换 (用户态-内核态-用户态)

不过，需要注意的是，系统调用过程中，并不会涉及到虚拟内存等进程用户态的资源，也不会切换进程。这跟我们通常所说的进程上下文切换是不一样的：进程上下文切换，是指从一个进程切换到另一个进程运行；而系统调用过程中一直是同一个进程在运行

#### 进程切换
进程是由内核管理和调度, 进程切换只能发生在内核态

比系统调用多一步, 我们需要保存当前进程的用户态资源(虚拟内存等)

![](https://pic3.zhimg.com/80/v2-440bb1699b2fa0f0340b38eabcbd7452_1440w.jpg)

## 进程切换比线程切换慢
在 linux 中, 进程和线程都是用同样的结构体表示, 并且使用一个指针指向内存资源
1. 不同进程指向不同的内存资源
2. 同一进程里的不同线程指向同一内存资源

**任务调度的开销**
1. 进程上下文的切换
2. CPU Cache/TLB不命中，导致缺页置换的开销

而同一进程的不同线程由于共享内存资源, 因此 2 带来的影响较小


**协程**
在用户态实现, 一种轻量级的线程
线程的弊端, 在执行比如 IO 操作时, 会阻塞等待返回. 此时协程可以主动让出, 而协程的切换不需要像线程一样依赖于操作系统, 从用户态到内核态的转换, ==协程切换完全在用户空间进行==

## 进程间通信
[参考: ipc](https://www.jianshu.com/p/c1015f5ffa74)
每个进程各自有不同的用户地址空间，任何一个进程的全局变量在另一个进程中都看不到，所以进程之间要交换数据必须通过内核，在内核中开辟一块缓冲区，进程1把数据从用户空间拷到内核缓冲区，进程2再从内核缓冲区把数据读走，内核提供的这种机制称为**进程间通信**（IPC，InterProcess Communication）

![](https://upload-images.jianshu.io/upload_images/1281379-76c95f147203c797.png)

1. **匿名管道**
    比如 `ls | grep hello`
    - 管道是半双工的，数据只能向一个方向流动；需要双方通信时，需要建立起两个管道。
    - 只能用于父子进程或者兄弟进程之间(具有亲缘关系的进程). ==因为父子进程共享内存空间(准确说是共享文件描述符), 因此可以访问==
    - 单独构成一种独立的文件系统：管道对于管道两端的进程而言，就是一个文件，但它不是普通的文件，它==不属于某种文件系统==，而是自立门户，单独构成一种文件系统，并且==只存在与内存==中。
    - 数据的读出和写入：一个进程向管道中写的内容被管道另一端的进程读出。写入的内容每次都添加在管道缓冲区的末尾，并且每次都是从缓冲区的头部读出数据

    **管道的实质**
    - 底层实现：==由内核管理的一个文件共享符或者说是一个缓冲区，管道的一端连接一个进程的输出, 另一端连接另一个进程的输入==
    - 管道的实质是一个内核缓冲区，进程以先进先出的方式从缓冲区存取数据，管道一端的进程顺序的将数据写入缓冲区，另一端的进程则顺序的读出数据。
    - 该缓冲区可以看做是一个循环队列，读和写的位置都是自动增长的，不能随意改变，一个数据只能被读一次，读出来以后在缓冲区就不复存在了。
    - 当缓冲区读空或者写满时，有一定的规则控制相应的读进程或者写进程进入等待队列，当空的缓冲区有新数据写入或者满的缓冲区有数据读出来时，就唤醒等待队列中的进程继续读写

    **管道的缺点**
    - 只支持单向数据流；
    - 只能用于具有亲缘关系的进程之间；
    - 没有名字(文件描述符, 即非亲属线程不能访问数组下标)；
    - 管道的缓冲区是有限的（管道制存在于内存中，在管道创建时，为缓冲区分配一个页面大小）；
    - 管道所传送的是无格式字节流，这就要求管道的读出方和写入方必须事先约定好数据的格式，比如多少字节算作一个消息（或命令、或记录）等等

2. **有名管道**
    有名管道不同于匿名管道之处在于它提供了一个路径名与之关联，==以有名管道的文件形式存在于文件系统中，这样，即使与有名管道的创建进程不存在亲缘关系的进程，只要可以访问该路径，就能够彼此通过有名管道相互通信==，因此，通过有名管道不相关的进程也能交换数据. ==有名管道的名字存在于文件系统中，内容存放在内存中==。

>（1）管道是特殊类型的文件，在满足先入先出的原则条件下可以进行读写，但不能进行定位读写。
（2）匿名管道是单向的，只能在有亲缘关系的进程间通信；有名管道以磁盘文件的方式存在，可以实现本机任意两个进程通信。
（3）无名管道阻塞问题：无名管道无需显示打开，创建时直接返回文件描述符，在读写时需要确定对方的存在，否则将退出。如果当前进程向无名管道的一端写数据，必须确定另一端有某一进程。如果写入无名管道的数据超过其最大值，写操作将阻塞，如果管道中没有数据，读操作将阻塞，如果管道发现另一端断开，将自动退出。
（4）有名管道阻塞问题：有名管道在打开时需要确实对方的存在，否则将阻塞。即以读方式打开某管道，在此之前必须一个进程以写方式打开管道，否则阻塞。此外，可以以读写（O_RDWR）模式打开有名管道，即当前进程读，当前进程写，不会阻塞。

3. **信号**
    - 信号是Linux系统中用于进程间互相通信或者操作的一种机制，信号可以在任何时候发给某一进程，而无需知道该进程的状态。
    - 如果该进程当前并未处于执行状态，则该信号就有内核保存起来，知道该进程回复执行并传递给它为止。
    - 如果一个信号被进程设置为阻塞，则该信号的传递被延迟，直到其阻塞被取消是才被传递给进程

    > Linux系统中常用信号：
    （1）SIGHUP：用户从终端注销，所有已启动进程都将收到该进程。系统缺省状态下对该信号的处理是终止进程。
    （2）SIGINT：程序终止信号。程序运行过程中，按Ctrl+C键将产生该信号。
    （3）SIGQUIT：程序退出信号。程序运行过程中，按Ctrl+\\键将产生该信号。
    （4）SIGBUS和SIGSEGV：进程访问非法地址。
    （5）SIGFPE：运算中出现致命错误，如除零操作、数据溢出等。
    （6）SIGKILL：用户终止进程执行信号。==shell下执行kill -9发送该信号==, 进程如果正在 IO 等待, 该信号可能被阻塞。
    （7）SIGTERM：结束进程信号。==shell下执行kill 进程pid发送该信号, 本质是 kill -15==。
    （8）SIGALRM：定时器信号。
    （9）SIGCLD：子进程退出信号。如果其父进程没有忽略该信号也没有处理该信号，则子进程退出后将形成僵尸进程。

    **信号来源:**
    信号是==软件层次上对中断机制的一种模拟，是一种异步通信方式==，信号可以在用户空间进程和内核之间直接交互，内核可以利用信号来通知用户空间的进程发生了哪些系统事件，信号事件主要有两个来源
    - 硬件来源：用户按键输入Ctrl+C退出、硬件异常如无效的存储访问等。
    - 软件终止：终止进程信号、其他进程调用kill函数、软件异常产生信号

    **信号生命周期和流程**
    （1）信号被某个进程产生，并设置此信号传递的对象（一般为对应进程的pid），然后传递给操作系统；
    （2）操作系统根据接收进程的设置（是否阻塞）而选择性的发送给接收者，如果接收者阻塞该信号（且该信号是可以阻塞的），操作系统将暂时保留该信号，而不传递，直到该进程解除了对此信号的阻塞（如果对应进程已经退出，则丢弃此信号），如果对应进程没有阻塞，操作系统将传递此信号。
    （3）目的进程接收到此信号后，将根据当前进程对此信号设置的预处理方式，暂时终止当前代码的执行，保护上下文（主要包括临时寄存器数据，当前程序位置以及当前CPU的状态）、转而执行中断服务程序，执行完成后在回复到中断的位置。当然，对于抢占式内核，在中断返回时还将引发新的调度。

    ![](https://upload-images.jianshu.io/upload_images/1281379-3eed8cca67aa9f55.png)

4. 消息队列

5. **共享内存**
- 使得多个进程可以可以直接读写同一块内存空间，是最快的可用IPC形式。是针对其他通信机制运行效率较低而设计的。
- 为了在多个进程间交换信息，内核专门留出了一块内存区，==可以由需要访问的进程将其映射到自己的私有地址空间(快的原因: 只需要进程在用户态进行读写操作, 而不需要转到内存态)==。进程就可以直接读写这一块内存而不需要进行数据的拷贝，从而大大提高效率。
- 由于多个进程共享一段内存，因此需要依靠某种==同步机制, 如信号量==来达到进程间的同步及互斥。

![](https://upload-images.jianshu.io/upload_images/1281379-adfde0d80334c1f8.png)

6. **信号量**
    信号量是一个计数器，用于多进程对共享数据的访问，信号量的意图在于进程间同步。
    为了获得共享资源，进程需要执行下列操作：
    1. 创建一个信号量：这要求调用者指定初始值，对于二值信号量来说，它通常是1，也可是0。
    2. 等待一个信号量：该操作会测试这个信号量的值，如果小于0，就阻塞。也称为P操作。
    3. 挂出一个信号量：该操作将信号量的值加1，也称为V操作。

    为了正确地实现信号量，信号量值的测试及减1操作应当是原子操作。为此，信号量通常是在==内核中==实现的。Linux环境中，有三种类型

    ![](https://upload-images.jianshu.io/upload_images/1281379-a72c8fbe22340031.png)

    > **信号量与互斥量**
    （1）互斥量用于线程的互斥，信号量用于线程的同步。这是互斥量和信号量的根本区别，也就是互斥和同步之间的区别
    互斥：是指某一资源同时只允许一个访问者对其进行访问，具有唯一性和排它性。但互斥无法限制访问者对资源的访问顺序，即访问是无序的。
    同步：是指在互斥的基础上（大多数情况），通过其它机制实现访问者对资源的有序访问。
    在大多数情况下，同步已经实现了互斥，特别是所有写入资源的情况必定是互斥的。少数情况是指可以允许多个访问者同时访问资源
    （2）互斥量值只能为0/1，信号量值可以为非负整数。
    也就是说，一个互斥量只能用于一个资源的互斥访问，它不能实现多个资源的多线程互斥问题。信号量可以实现多个同类资源的多线程互斥和同步。当信号量为单值信号量是，也可以完成一个资源的互斥访问。
    （3）互斥量的加锁和解锁必须由同一线程分别对应使用，信号量可以由一个线程释放，另一个线程得到。


## 线程通信
1. 加锁(synchronized, ReentrantLock, 乐观锁)
2. wait, notify 机制
3. 信号量 (Semaphore)


## 孤儿进程 与 僵死进程

- 孤儿进程：父进程在子进程终止之前终止，那么此时子进程被称为孤儿进程，他们的父进程会被更改init进程。init进程收养的子进程永远不会变成僵死进程，因为只要有一个子进程终止，init就会调用一个wait函数取得其终止状态。
- 僵死进程：一个已经终止，但父进程并没有调用wait或waitpid进行善后处理（获取终止进程信息，释放它所占用的资源）的进程称之为僵死进程。

避免僵死进程的方法是调用fork()两次，将第一次fork子进程终止，这样第二次fork的子进程(本进程的孙子进程)就会被init进程收养，交给init进程来获取子进程的终止状态。

## fork 会占用两倍内存吗
内存子进程经过 fork 产生，占用内存大小等同于父进程，理论上须要两倍的内存。

但是可以采用写时复制（copy-on-wirte）技术，即父子进程会共享相同的物理内存区域，当父子进程试图修改这些区域时，会建立对应的副本(父进程享有原有的空间, 子进程开辟新的内存)。

## 进程调度算法
1. 先到先服务
**缺点:** 短作业排在长作业之后, 需等待很长时间
2. 短作业优先
**缺点:** 可能导致长作业饥饿
3. 高响应比
考虑上述两种情况, 设计出响应比 = (等待时间 + 作业时间) / 作业时间, 每次选择响应比最高的作业进行服务
**缺点:** 每次切换进程, 需要对每个作业计算响应比，增加系统开销
4. 优先级
**缺点:** 优先级识别与分配是一个问题, 还可能导致饥饿
5. 时间片轮询 (Linux 采用)
缺点: 时间片的选择是一个重要问题

# 死锁
死锁产生的条件
1. 互斥条件：该资源任意一个时刻只由一个线程占用。
2. 请求与保持条件：一个进程因请求资源而阻塞时，对已获得的资源保持不放。
3. 不剥夺条件: 线程已获得的资源在未使用完之前不能被其他线程强行剥夺，只有自己使用完毕后才释放资源。
4. 循环等待条件: 若干进程之间形成一种头尾相接的循环等待资源关系。

解决方法
1. 破坏互斥条件 ：这个条件我们没有办法破坏，因为我们用锁本来就是想让他们互斥的（临界资源需要互斥访问）。
2. 破坏请求与保持条件 ：一次性申请所有的资源。
3. 破坏不剥夺条件 ：占用部分资源的线程进一步申请其他资源时，如果申请不到，可以主动释放它占有的资源。
4. 破坏循环等待条件 ：靠按序申请资源来预防。按某一顺序申请资源，释放资源则反序释放(``非必须 -linus``)。破坏循环等待条件

> 释放锁的顺序
如果有一个单链表, 并且我们需要锁定他的全部节点, 唯一合理的行为是遍历一遍上锁, 再遍历一遍解锁(即顺序上锁, 顺序解锁)
--linus

# 虚拟内存
[参考](https://juejin.cn/post/6844904202720788493#heading-4)
[参考](https://www.zhihu.com/question/323415592)
虚拟内存为每个进程提供了一个一致的、私有的地址空间，它让每个进程产生了一种自己在独享主存的错觉（每个进程拥有一片连续完整的内存空间）

虚拟内存提供
- 它把主存看作为一个存储在硬盘上的虚拟地址空间的高速缓存，并且只在主存中缓存活动区域（按需缓存）。
- 它为每个进程提供了一个一致的地址空间，从而降低了程序员对内存管理的复杂性。
- 它还保护了每个进程的地址空间不会被其他进程破坏

## 虚拟内存与物理内存的映射
### 分段式
程序由若干逻辑分段组成, 如代码段, 栈段, 堆段.
不同段有不同的属性

分段机制下, 虚拟内存由两部分组成, 段选择子和段内偏移量

![](https://user-gold-cdn.xitu.io/2020/6/30/17303f81d3b6a20b)

![](https://user-gold-cdn.xitu.io/2020/6/30/17303f81ffd38048)

#### 分段的缺点:
存疑???

1. 内存碎片

    ![](https://user-gold-cdn.xitu.io/2020/6/30/17303f820070d348?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

    - 外部碎片: 产生多个不连续的物理内存, 导致新的程序无法装载
    - 内部碎片: 

==解决外部碎片的一个方法就是使用内存交换, 也就是 linux 内存里看到的 swap 区域==

2. 内存交换效率低
分段的方式, 很容易产生外部碎片(5k 的段换 4k 的段, 会有 1k 的碎片), 因此需要频繁的进行内存交换

### 分页式
将整个内存空间划分为一个个连续且固定大小的页, linux 中每页长度为 4k

### 寻址
CPU中含有一个被称为内存管理单元（Memory Management Unit, MMU）的硬件，它的功能是将虚拟地址转换为物理地址。MMU需要借助存放==在内存中的页表==来动态翻译虚拟地址，该==页表由操作系统管理==

### 页表
内存最小操作单元为一页, 一般是 4K 字节, 与一个块大小相同, 这个块被称为虚拟页（Virtual Page, VP），每个虚拟页的大小为P=2^p字节。物理内存也会按照这种方法分割为物理页（Physical Page, PP），大小也为P字节

CPU在获得虚拟地址之后，需要通过MMU将虚拟地址翻译为物理地址。而在翻译的过程中还需要借助页表，所谓==页表就是一个存放在物理内存中的数据结构，它记录了虚拟页与物理页的映射关系==

页表是一个元素为页表条目（Page Table Entry, PTE）的集合，每个虚拟页在页表中一个固定偏移量的位置上都有一个PTE。下面是PTE仅含有一个有效位标记的页表结构，该有效位代表这个虚拟页是否被缓存在物理内存中。

![](https://user-gold-cdn.xitu.io/2017/10/31/f37cc0b690138449ffd9fed42b41c44e)

虚拟页VP 0、VP 4、VP 6、VP 7被缓存在物理内存中，虚拟页VP 2和VP 5被分配在页表中，但并没有缓存在物理内存，虚拟页VP 1和VP 3还没有被分配

由于CPU每次进行地址翻译的时候都需要经过PTE，所以如果想控制内存系统的访问，可以在PTE上添加一些额外的许可位（例如读写权限、内核权限等），这样只要有指令违反了这些许可条件，CPU就会触发一个一般保护故障，将控制权传递给内核中的异常处理程序。一般这种异常被称为“段错误（Segmentation Fault）”

### 缺页
![](https://user-gold-cdn.xitu.io/2017/10/31/ae4f92f5f10d92a8332864ae0604a636)

如上图所示，MMU根据虚拟地址在页表中寻址到了PTE 2，该PTE的有效位为0，代表该虚拟页并没有被缓存在物理内存中。==虚拟页没有被缓存在物理内存中（缓存未命中）被称为缺页。==

当CPU遇见缺页时会触发一个缺页异常，缺页异常将控制权转向操作系统内核，然后==调用内核中的缺页异常处理程序==，该程序会选择一个牺牲页，如果牺牲页已被修改过，内核会先将它复制回硬盘（采用写回机制而不是直写也是为了尽量减少对硬盘的访问次数），然后再==把该虚拟页覆盖到牺牲页==的位置，并且更新PTE

当缺页异常处理程序返回时，它会重新启动导致缺页的指令，该指令会把导致缺页的虚拟地址重新发送给MMU。由于现在已经成功处理了缺页异常，所以最终结果是页命中，并得到物理地址。

这种在硬盘和内存之间传送页的行为称为页面调度（paging）：==页从硬盘换入内存和从内存换出到硬盘==。当缺页异常发生时，才将页面换入到内存的策略称为按需页面调度（demand paging），所有现代操作系统基本都使用的是按需页面调度的策略。

==虚拟内存跟CPU高速缓存（或其他使用缓存的技术）一样依赖于局部性原则==, 如果程序没有遵循局部性原理, 则会频繁地发生页调度, 产生抖动(thrashing)

#### 页面置换算法
- FIFO（First In First Out） 页面置换算法（先进先出页面置换算法） : 总是淘汰最先进入内存的页面，即选择在内存中驻留时间最久的页面进行淘汰。
- LRU （Least Currently Used）页面置换算法（最近最久未使用页面置换算法） ：LRU算法赋予每个页面一个访问字段，用来记录一个页面自上次被访问以来所经历的时间 T，当须淘汰一个页面时，选择现有页面中其 T 值最大的，即最近最久未使用的页面予以淘汰。
- LFU （Least Frequently Used）页面置换算法（最少使用页面置换算法） : 该置换算法选择在之前时期使用最少的页面作为淘汰页。

### 多级页表
真实的虚拟空间最大可以是 cpu 的寻址空间 (32 位 cpu 为 2^32, 每个地址一个字节, 则可寻址 4GB), 因此我们需要多级页表
![](https://user-gold-cdn.xitu.io/2017/10/31/9eb1c115f4d96c533c61c01ea4c5ef04)
- 如果一个一级页表不存在, 则对应的二级页表也不存在, 可以节约空间
- 只有一级页表才常驻内存, 二级页表可以在需要时创建, 页表调度, 减少内存压力

### 地址翻译过程
![](https://user-gold-cdn.xitu.io/2017/10/31/c7bf4fc683ff989b37bad182e4fda0f9)
一个 n 位的虚拟地址包括, 一个 p 位的虚拟页面偏移量 （Virtual Page Offset, VPO）和一个（n - p）位的虚拟页号（Virtual Page Number, VPN）
mmu 根据 vpn 来寻找 pte, 从而找到对应的物理页, 然后拼接上 VPO 得到物理地址

多级页表也是如此, 只是需要将 VPN 分成多段
![](https://user-gold-cdn.xitu.io/2017/10/31/04c2514a46f77fb489eb028d3ca57ad9)

### TLB
页表是缓存在内存中, 为了==防止每次翻译操作都去访问内存==, cpu 使用高速缓存和 TLB 来缓存 PTE

TLB（Translation Lookaside Buffer, TLB）被称为翻译后备缓冲器或翻译旁路缓冲器，它是MMU中的一个缓冲区，其中每一行都保存着一个由单个PTE组成的块。用于组选择和行匹配的索引与标记字段是从VPN中提取出来的，如果TLB中有T = 2^t个组，那么TLB索引（TLBI）是由VPN的t个最低位组成的，而TLB标记（TLBT）是由VPN中剩余的位组成的

当 TLB 命中时
![](https://user-gold-cdn.xitu.io/2017/10/31/bce825a3d3d87894a65e550fdba92f36)

如果 TLB 未命中, 则需要从高速缓存/内存中取得 PTE, 并将其放入 TLB
![](https://user-gold-cdn.xitu.io/2017/10/31/92dbaef67abacddbeff7cfd76039e57a)

## linux 的内存管理
主要使用分页式内存管理, 但由于 intel 使用段页式, 因此 linux 将每个段的基地址都设为 0

Linux 系统中的每个段都是从 0 地址开始的整个 4GB 虚拟空间（32 位环境下），也就是所有的段的起始地址都是一样的。这意味着，Linux 系统中的代码，包括操作系统本身的代码和应用程序代码，所面对的地址空间都是线性地址空间（虚拟地址），==这种做法相当于屏蔽了处理器中的逻辑地址概念==，段只被用于访问控制和内存保护

![](https://user-gold-cdn.xitu.io/2020/6/30/17303f8274ca33c9?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

内核态与用户态:
1. 只有进程处于内核态, 才能访问内核空间
2. 在用户态时, 只能访问用户空间

虽然每个进程都有独立的虚拟内存(==即每个进程有自己的一级页表==), 但是==每个虚拟内存的虚拟内核地址, 都指向同一个物理内存地址==

![](https://user-gold-cdn.xitu.io/2020/6/30/17303f828c5e812e?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

![](https://user-gold-cdn.xitu.io/2020/6/30/17303f82925ede17?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)
