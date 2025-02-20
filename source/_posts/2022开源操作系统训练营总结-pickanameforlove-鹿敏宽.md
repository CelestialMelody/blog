---
title: 2022开源操作系统训练营总结-pickanameforlove-鹿敏宽
date: 2022-08-01 10:46:53
categories:
    - report
tags:
    - author:pickanameforlove
    - summerofcode2022
    - rcode-lab
---

<!-- more -->

### rust语言学习

* rust最具特色的就是他的内存所有权机制。一块内存只能有一个所有者。当遇到```=```和函数的参数传递的时候，所有权便会转移。
* rust所有的变量默认上都是不可变的，这和其他语言不同，如果想要修改变量的内容，需要加上```mut```关键字。
* rust中没有```Null```这个关键字，有的只是```Option```枚举类型。



### rcore学习体会

#### 实验一

* 初步理解了时间片轮转，预先设置好一个个时钟中断，每次到了时钟中断的时候，切换任务。

#### 实验二

* 解决了一直以来对页表的误解。之前以为页表中的每一项都是分为两个部分，虚拟页号和物理帧号。阅读文档之后才明白，页表是按一个数组组织的，下标即为虚拟页号，因此页表项中不需要存储虚拟页号。
* 理解了之前学的“段”，“页”是什么概念。一个进程包含多个逻辑段，逻辑段具有相同权限，以同样方式映射的一段连续的虚拟内存。一个地址空间中既有逻辑段数组，也有页表。这两项都存储着虚拟页号和物理帧号的映射关系，更新的时候需要注意两个都需要维护。

#### 实验三

* 学习了stride调度算法，了解了如何处理stride溢出。
* 知道了新建一个进程需要：获取地址空间，pid，内核栈；新建PCB；维护父子关系；初始化trap上下文。

#### 实验四容易混淆的地方

- 文件和目录，文档中将文件和目录写成了同一层次的东西。但通过看代码得知，获取文件内容的流程是，给定文件名，根据文件名去找DirEntry，根据DirEntry里面的inode\_number解析出文件inode所在的blockid和offset.通过文件的inode获取文件的内容。因此逻辑结构上，目录是在文件之上的。
- DirEntry的inode\_number和blockid混淆了。因为文档中指定一个物理block中存四个inode，因此上DirEntry给一个blockid是无法唯一指定inode的，并且结合代码得知inode\_number是由blockid和offset构成的。
- Super Block与ROOT\_INODE混淆了，Super Block是磁盘的第一个块，记录了磁盘的相关划分信息。而ROOT\_INODE是根目录的Inode，包含了根目录包含的DirEntry所在的物理块的blockid。我们在实现系统调用的时候，所有的操作都要基于ROOT\_INODE来进行操作，从他这里获取到文件的inode，进而获取文件的内容。
- DiskInode，Inode和OSInode混淆了，这三个是逐步升级的封装。从做实验的角度来看，DiskInode就是最基本的文件inode，包含了文件内容所在的数据块的索引，我们不会在这个层次上进行操作。inode是对diskInode的进一步封装，包含了inode\_number，type等额外属性，实验中涉及到的创建硬链接就是在这个类上操作的。而OSInode则将其封装的更像一个文件了，十分简洁，就三个属性：是否可读，是否可写，对应的inode。
- DiskInode和block混淆了，据文档所说，一个物理block（512B)上面有4个DiskInode(128B)。
- 和DiskInode相关的两个offset混淆，一个offset是用于指明当前DiskInode所在的block的位置的，毕竟一个block可以容纳4个DiskInode。另一个和diskInode相关的offset则是读取DiskInode关联的数据块中的数据时的offset，文件系统实现的时候将inode和数据节点的区别隐藏了，在inode节点上调用的read方法返回的就是该inode节点关联的数据节点上的数据。处理的时候将这些数据视为一个数组，即读取的时候需要不断地指定offset，直到读取完数据。
- 将逻辑上文件与目录的依赖关系和具体实践上文件和目录的独立混淆了。逻辑上的话，文件系统就是一棵树，树的叶子节点就是文件。非叶节点就是目录。因此从逻辑上来讲目录是依赖于文件而存在的，文件依赖着目录的索引功能。但是在实践上，这两种结构就是完全独立的，一个文件，具体实现起来的话，是inode+数据节点的结构。而一个目录，实现起来的话，也是inode+数据节点的结构。不同的是文件的数据节点中的数据是没有特定结构的二进制数据，而目录的数据节点中存储的是目录项，由名称和inode\_number组成。那么这就有问题了，在目录与文件独立的实现结构下，如何实现文件的索引呢？答案就是目录项的inode\_number，他是目录项名字对应的inode，无论是文件也好，还是下一级目录也好，获得这个inode\_number就可以进行迭代的索引了，直到索引到想要的文件为止。

#### 实验五

* 理解了“线程是执行的最小单位，进程是分配资源的最小单位”，这句话更通俗的解释就是：共享资源，地址空间是进程PCB的属性；线程是CPU调度的基本单位。

### rcore实验实现细节

#### 实验一

lab1要求实现一个获取当前任务的taskinfo的系统调用。为了实现该功能，为每个任务的TaskControlBlock添加了两个属性，一个属性描述该任务开始执行的时间，一个属性描述该任务使用的系统调用及其次数，用一个大小为500的列表实现。

任务开始执行的时间初始化为0，在任务从UnInit状态转为Running状态的时候，使用框架提供的get\_time()函数进行修改。而系统调用相关的信息在调用系统调用的时候进行更新，也就是在syscall目录下的mod.rs文件中进行更新。

#### 实验二

首先重写sys\_get\_time和sys\_task\_info这两个系统调用。改进的地方是将传入的虚拟地址翻译成物理地址，使用当前任务的地址空间翻译。

然后就是sys\_mmap和sys\_munmap两个系统调用，这其中最核心的是判断一个虚拟页号是否已经映射了。通过TaskManager找到当前任务，通过它的地址空间memory\_set判断是否包含虚拟页号。地址空间由多个逻辑段组成，遍历这些逻辑段一个个判断是否包含指定的逻辑页。逻辑段既可以通过起止页号来判断，也可以使用物理帧和逻辑页的map来进行判断。

进行判断之后，对Vec\<MapArea\>和PageTable进行更新（添加/删除）。

#### 实验三

实现了spawn系统调用，和fork+exec很相似，不过不用拷贝父进程的地址空间。大致的流程是从elf文件中获取相关信息；分配pid和内核栈；初始化实例；设置trap上下文信息。

然后是stride调度算法。包含两步，设置优先级，将Task Manager的fetch方法中默认的先入先出调度方法修改为stride调度方法。不过在实现中会有pass溢出的问题。而rust在通过强制截断的方式来处理溢出。那么需要解决的问题是，在pass累加溢出之后，我们还能找到真正的最小pass的进程。根据定理：

> **在不考虑溢出的情况下** , 在进程优先级全部 >= 2 的情况下，如果严格按照算法执行，那么 PASS\_MAX – PASS\_MIN <= BigStride / 2。

利用两个pass的差来判定大小，并且需要保证pass的差不会溢出。比如BigStride取值usize::MAX ，那么差可以表示为isize类型，因为isize的范围是[-BigStride/2,BigStride/2-1]，符合要求。

#### 实验四

实现了三个系统调用，sys\_linkat，sys\_unlinkat，sys\_stat。

sys\_linkat的思路就是在根目录上新建一个目录项，目录项的名字是硬链接的名字，inode\_number就是原链接的inode\_number。

sys\_unlinkat和sys\_stat涉及到统计文件链接数目的一个功能。实现的细节就是遍历根目录的目录项，检查其inode\_number是否与给定文件的inode\_number相同，如果相同就加一。

sys\_unlinkat中删除目录项采用了最简单的实现方式，直接在要删除的目录项上复写一个空的目录项。没有实现目录项的搬运，也没有更新目录的大小。

sys\_stat最关键的是通过文件描述符获得inode\_number。实现的细节是给file trait加一个方法get\_inode\_number，让OSInode实现即可(ps：文件描述符表存的对象就是OSInode类型的)。

#### 实验五

实现了死锁检测算法。算法涉及到的几个信息。Allocated，Needed全是存在了mutex/semaphore上，等到调用算法的时候再汇总为一个矩阵。其中Needed信息就是该mutex/semaphore的等待队列。而Allocated则是新建了一个Vec数组来记录分配到资源的线程情况。vec数组的下标表示的tid，值表示的是分配到的互斥资源的个数。

mutex/semaphore各自实现了get\_count函数，用来获取剩下可用的资源个数。mutex根据lock与否决定个数。semaphore可以将count直接返回，但是需要做一定的改进，当count小于0的时候，返回0.
