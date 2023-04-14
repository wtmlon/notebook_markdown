# malloc函数

malloc实际上是分配虚拟地址空间（需要访管），最终调用的函数为sys_brk()和sys_mmap()。

    其中当申请的空间大于某个阈值时（一般是128kb），会使用mmap函数进行虚拟地址 空间申请，申请的位置处于堆区和栈区中间部分。当申请空间小于128kb时，则使用brk函数，他的作用是将堆区指针往后移动（扩大）。mmap生成的空间用vm_area_struct结构体表示，使用mumap函数释放，而整个进程的虚拟地址空间用mm_struct表示，mm->start_bk为堆区的起始地址，他不会大于mm->mmap_base（使用mmap申请的空间的最底地址）。

    申请出来的虚拟地址空间并没有实际的物理空间与之对应，只有等到产生第一次访问缺页时才会分配。

![](/Users/liuting/Library/Application%20Support/marktext/images/2023-04-10-19-29-48-image.png)

# c虚拟内存管理

![](/Users/liuting/Library/Application%20Support/marktext/images/2023-04-10-19-31-47-image.png)

 由上图可以看出，进程的虚拟地址空间，由多个虚拟内存区域构成。虚拟内存区域是进程的虚拟地址空间中的一个同质区间，即具有同样特性的连续地址范围。上图中所示的text数据段、初始数据段、Bss数据段、堆、栈、内存映射，都是一个独立的虚拟内存区域。而为内存映射服务的地址空间处在堆栈之间的空余部分。  

linux 内核使用的vm_area_struct 结构来表示一个独立的虚拟内存区域，由于每个不同质的虚拟内存区域功能和内部机制不同；因此同一个进程使用多个vm_area_struct 结构来分别表示不同类型的虚拟内存区域。各个vm_area_struct 结构使用链表或者树形结构链接，方便进程快速访问。

![](/Users/liuting/Library/Application%20Support/marktext/images/2023-04-10-19-32-35-image.png)

 vm_area_struct 结构中包含区域起始和终止地址以及其他相关信息，同时也包含一个vm_ops 指针，其内部可引出所有针对这个区域可以使用的系统调用函数。这样，进程对某一虚拟内存区域的任何操作都需要的信息，都可以从vm_area_struct 中获得。mmap函数就是要创建一个新的vm_area_struct结构 ，并将其与文件的物理磁盘地址相连。

# 进程间通信mmap和shm的区别

在说mmap之前我们先说一下普通的读写文件的原理，进程调用read或是write后会陷入内核，因为这两个函数都是系统调用，进入系统调用后，内核开始读写文件，假设内核在读取文件，内核首先把文件读入自己的内核空间（Buffer Cache），读完之后进程在内核回归用户态，内核把读入内核内存的数据再copy进入进程的用户态内存空间。所以，一般调用read和write时，程序员需要先malloc一块空间（buff[BUF_SIZE]）用来存放数据，实际上我们同一份文件内容相当于读了两次，先读入内核空间，再从内核空间读入用户空间。

mmap是在用户文件系统中创建一个文件，并映射到进程的虚拟空间，所以用户可以直接以访问内存的形式访问，不需要额外的buff，也不需要从内核拷贝到用户空间。

```c
  void *mmap(void *addr, size_t length, int prot, int flags,
                  int fd, off_t offset); //注意，参数中有个fd，说明基于文件
```

而shm则是在swap文件分区（安装ubuntu的时候你有配置过）中的shm文件系统新建一个文件，将这个文件的地址映射到进程虚拟空间。

![](/Users/liuting/Library/Application%20Support/marktext/images/2023-04-10-19-46-39-image.png)

```c
int shmget(key_t key, size_t size, int shmflg); //在物理内存创建一个共享内存，返回共享内存的编号。
void *shmat(int shmid, constvoid shmaddr,int shmflg); //连接成功后把共享内存区对象映射到调用进程的地址空间
void *shmdt(constvoid* shmaddr); //断开用户级页表到共享内存的那根箭头。
int shmctl(int shmid, int cmd, struct shmid_ds* buf); //释放物理内存中的那块共享内存。
```

总结mmap和shm:
1、mmap是在磁盘上建立一个文件，每个进程地址空间中开辟出一块空间进行映射。
而对于shm而言，shm每个进程最终会映射到同一块物理``内存``。shm保存在物理内存，这样读写的速度要比磁盘要快，但是存储量不是特别大。(存疑)
2、相对于shm来说，mmap更加简单，调用更加方便，所以这也是大家都喜欢用的原因。
3、另外mmap有一个好处是当机器重启，因为mmap把文件保存在磁盘上，这个文件还保存了操作系统同步的映像，所以mmap不会丢失，但是shmget就会丢失。

mmap贡献内存操作需要上文件锁，在Linux中，实现文件上锁的函数有lockf()和fcntl()

# 系统文件，文件描述符和Page/Buffer Cache的关系

先从一张图开始

![](/Users/liuting/Library/Application%20Support/marktext/images/2023-04-10-20-03-30-image.png)

``内核``维护的3个数据结构。进程级的文件描述符表(进程内核区)、 系统级的打开文件描述符表、文件系统的i-node表

（1）每个进程都有一个进程表，表中记录了文件描述符和对应的文件表指针。表项中包含的内容如下
                a.文件描述符 fd。
                b.指向一个进程文件表项的指针。
（2）内核为进程打开文件维持一张进程级的文件描述符表。每个文件表项包含：
                a.文件描述符及打开方式
                b.系统打开文件表索引
（3）内核为所有进程打开文件维持一张系统级的文件描述符表。每个文件表项包含：
                a.文件状态标志（读、写、添写、同步和非阻塞等）
                b.引用计数
                c.指向该文件V节点表项的指针
                d.当前文件偏移量
（4）每个打开文件（或设备）都有一个v节点结构，每个v节点结构包含：
                a.当前文件长度
                b.具体i节点信息

分析：

在进程A中，文件描述符1和30都指向了同一个打开的文件句柄（标号23）。这可能是通过调用dup()、dup2()、fcntl()或者对同一个文件多次调用了open()函数而形成的。

进程A的文件描述符2和进程B的文件描述符2都指向了同一个打开的文件句柄（标号73）。这种情形可能是在调用fork()后出现的（即，进程A、B是父子进程关系），或者当某进程通过UNIX域套接字将一个打开的文件描述符传递给另一个进程时，也会发生。再者是不同的进程独自去调用open函数打开了同一个文件，此时进程内部的描述符正好分配到与其他进程打开该文件的描述符一样。

此外，进程A的描述符0和进程B的描述符3分别指向不同的打开文件句柄，但这些句柄均指向i-node表的相同条目（1976），换言之，指向同一个文件。发生这种情况是因为每个进程各自对同一个文件发起了open()调用。同一个进程两次打开同一个文件，也会发生类似情况。

总结： 因为一个文件可以被多个进程打开，所以多个文件表对象可以指向同一个文件节点，但多个文件表对象其对应的索引节点和目录项对象肯定是惟一的。<mark>一个inode（i节点）对应一个page cache对象，一个page cache对象包含多个物理page。</mark>

## Page 和 Buffer cache的区别

### Page cache

CPU如果要访问外部磁盘上的文件，需要首先将这些文件的内容拷贝到内存中，由于硬件的限制，从磁盘到内存的数据传输速度是很慢的，如果现在物理内存有空余，可以使用空闲内存来缓存一些磁盘的文件内容，这部分用作缓存磁盘文件的内存就叫做page cache。多数文件系统的默认IO操作都是缓存IO、在Linux的缓存IO机制中、操作系统会将IO的数据缓存在文件系统的页缓存（page cache）中、也就是说、数据会先被拷贝到操作系统内核的缓冲区中、然后才会从操作系统内核的缓冲区拷贝到应用程序的地址空间。

Page Cache是内核与存储介质的重要缓存结构，当我们使用write()或者read()读写文件时，假如不使用O_DIRECT标志位打开文件，我们均需要经过Page Cache来帮助我们提高文件读写速度。而在MySQL的设计实现中，读写数据文件使用了O_DIRECT标志，其目的是使用自身Buffer Pool的缓存算法。在Linux内核内存的基本单元是Page，而Page Cache也驻存于物理内存，所以Page Cache的缓存基本单位也是Page，而Page Cache缓存的内容属于文件系统，所以Page Cache属于文件系统与物理内存管理的枢纽。

page cache: 页缓冲/文件缓冲，通常4K，由若干个磁盘块组成（物理上不一定连续），也即由若干个bufferCache组成。 Page cache 也叫页缓冲或文件缓冲，是由好几个磁盘块构成，大小通常为4k，在64位系统上为8k，构成的几个磁盘块在物理磁盘上不一定连续，文件的组织单位为一页， 也就是一个page cache大小，文件读取是由外存上不连续的几个磁盘块，到buffer cache，然后组成page cache，然后供给应用程序。 Page cache在linux读写文件时，它用于缓存文件的逻辑内容，从而加快对磁盘上映像和数据的访问。具体说是加速对文件内容的访问，buffer cache缓存文件的具体内容——物理磁盘上的磁盘块，这是加速对磁盘的访问。

buffer cache： Buffer cache 也叫块缓冲，是对物理磁盘上的一个磁盘块进行的缓冲，其大小为通常为1k，磁盘块也是磁盘的组织单位。设立buffer cache的目的是为在程序多次访问同一磁盘块时，减少访问时间。Buffer cache 是由物理内存分配，linux系统为提高内存使用率，会将空闲内存全分给buffer cache ，当其他程序需要更多内存时，系统会减少cahce大小。

区别： 磁盘的操作有逻辑级（文件系统）和物理级（磁盘块），这两种Cache就是分别缓存逻辑和物理级数据的。<mark>假设我们通过文件系统操作文件（既通过文件描述符->打开文件表...那一套操作），那么文件将被缓存到Page Cache</mark>，如果需要刷新文件的时候，Page Cache将交给Buffer Cache去完成，因为Buffer Cache就是缓存磁盘块的。也就是说，直接去操作文件，那就是Page Cache区缓存，用dd等命令直接操作磁盘块，就是Buffer Cache缓存的东西。在Linux2.4中，buffer cache和 page cache之间是独立的，前者使用老版本的buffer_head进行存储，这导致了一个磁盘block可能在两个cache中同时存在，造成了内存的浪费。2.6内核中将两者合并到了一起，使buffer_head只存储buffer-block的映射信息，不再存储block的内容。这样保证一个磁盘block在内存中只会有一个副本，减少了内存浪费。在linux kernel 2.4之前，这两个cache是不同的：file在page cache中，disk block在buffer cache中。考虑到大多数的文件是由disk上的文件系统表示的，数据会在系统中有两份缓存，一份在page cache中，一份在buffer cache中。许多类unix系统都是这种模式。这种实现是很简单，但是明显的低效和冗余。在linux kernel 2.4之后，这两种caches统一了。虚拟内存子系统通过page cache驱动IO。如果数据同时在page cache和buffer cache中，buffer cache会简单的指向page cache，数据只会在内存中保有一份。Page cache就是你所知道的disk在内存中的缓存：它会加快文件访问速度。

Page cache实际上是针对文件系统的，是文件的缓存，在文件层面上的数据会缓存到page cache。文件的逻辑层需要映射到实际的物理磁盘，这种映射关系由文件系统来完成。当page cache的数据需要刷新时，page cache中的数据交给buffer cache，但是这种处理在2.6版本的内核之后就变的很简单了，没有真正意义上的cache操作。Buffer cache是针对磁盘块的缓存，也就是在没有文件系统的情况下，直接对磁盘进行操作的数据会缓存到buffer cache中，例如，文件系统的元数据都会缓存到buffer cache中。简单说来，page cache用来缓存文件数据，buffer cache用来缓存磁盘数据。在有文件系统的情况下，对文件操作，那么数据会缓存到page cache，如果直接采用dd等工具对磁盘进行读写，那么数据会缓存到buffer cache。 一个inode（i节点）对应一个page cache对象，一个page cache对象包含多个物理page。
![](/Users/liuting/Library/Application%20Support/marktext/images/2023-04-10-20-23-22-image.png)

### Page cache的查找

![](/Users/liuting/Library/Application%20Support/marktext/images/2023-04-10-20-24-07-image.png)

![](/Users/liuting/Library/Application%20Support/marktext/images/2023-04-11-10-33-22-image.png)

Trie前缀树, 广泛应用于字符串搜索. 它是多叉树结构, 每个节点代表一个字符, 从根出发到叶子, 所访问过的字符连接起来就是一个字符串。Radix tree 是Trie的一种优化方式, 对空间进一步压缩。假如树中的一个节点是父节点的唯一子节点(the only child)的话，那么该子节点将会与父节点进行合并，这样就使得radix tree中的每一个内部节点最多拥有r个孩子， r为正整数且等于2^n(n>=1)。

查找： 如何查找需要的数据是不是已经缓存在page cache中？当查找所需要的页时，内核把页索引转换为基树的路径，并快速找到页描述符所在位置。如果找到，内核可以从基树获得页描述符，并且确定所找到的页是否是脏页，以及其数据是否正在进行。基树能快速根据文件索引节点和文件内的逻辑偏移快速确定指定页面是否缓存，并且能返回当前的页面状态。内核中具体的查找函数是find_get_page(mapping, offset)，如果在page cache中没有找到，就会触发缺页异常page fault，调用__page_cache_alloc()在内存中分配若干物理页面，然后将数据从磁盘对应位置copy过来。内存查找数据使用了基树的数据结构。Linux radix树最广泛的用途是用于内存管理，结构address_space通过radix树跟踪绑定到地址映射上的核心页，该radix树允许内存管理代码快速查找标识为dirty或writeback的页。

### 直接I/O

![](/Users/liuting/Library/Application%20Support/marktext/images/2023-04-10-20-25-16-image.png)O_DIRECT：如果进程在open()一个文件的时候指定flags为O_DIRECT，那进程和这个文件的数据交互就直接在用户提供的buffer和磁盘之间进行，page cache就被bypass了，这种文件访问方式被称为direct I/O，适用于用户使用自己设备提供的缓存机制的场景，比如某些数据库应用。Linux允许应用程序在执行磁盘IO时绕过缓冲区高速缓存，从用户空间直接将数据传递到文件或磁盘设备，称为直接IO（direct IO）或者裸IO（raw IO）。但是对于某些特殊的应用程序来说、避开操作系统内核缓冲区而直接在应用程序地址空间和磁盘之间传输数据会比使用操作系统内核缓冲区获取更好的性能、自缓存应用程序就是其中的一种、自缓存应用程序会有它自己的数据缓存机制、比如它会将数据缓存在应用程序地址空间、这类应用程序完全不需要使用操作系统内核中的高速缓冲存储器、**数据库管理系统是这类应用程序的一个代表。**直接IO中数据均直接在用户地址空间（用户空间）和磁盘之间直接进行传输、完全不需要页缓存的支持。以下情况下我们可能需要考虑DirectIO：对数据写的可靠性要求很高，必须确保数据落到磁盘上，业务逻辑才可以继续执行。特定场景下，系统自带缓存算法效率不高，应用层自己实现出更高的算法。

直接IO优点：减少一次从内核缓冲区到用户程序缓存的数据复制、这种访问文件的方式通常是在对数据的缓存管理由应用程序实现的数据库管理系统中、应用程序明确地知道应该缓存哪些数据、应该失效哪些数据、操作系统只是简单地缓存最近一次从磁盘读取的数据、

直接IO缺点: 如果访问的数据不在应用程序缓存中、那么每次访问数据都会直接从磁盘加载、这种直接加载会非常缓慢。

### page cache的回收

回收page释放内存空间，此时会选择合适的page进行释放，如果是脏页会先同步到磁盘然后释放。触发脏页回写到磁盘时机如下：

- 用户进程调用sync() 和 fsync()系统调用；
- 空闲内存低于特定的阈值（threshold）；
- Dirty数据在内存中驻留的时间超过一个特定的阈值。

在系统中除了内存将被耗尽的时候可以清缓存以外，我们还可以使用下面这个文件来人工触发缓存清除的操作：

方法是：

```bash
echo 1 > /proc/sys/vm/drop_caches
当然，这个文件可以设置的值分别为1、2、3。它们所表示的含义为：
```

```bash
echo 1 > /proc/sys/vm/drop_caches:表示清除pagecache。
echo 2 > /proc/sys/vm/drop_caches:表示清除回收slab分配器中的对象（包括目录项缓存和inode缓存）。slab分配器是内核中管理内存的一种机制，其中很多缓存数据实现都是用的pagecache。
echo 3 > /proc/sys/vm/drop_caches:表示清除pagecache和slab分配器中的缓存对象。
```
