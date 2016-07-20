#c++多线程编程基础知识杂记
##cache一致性
cache一致性保证了多个CPU之间看到的内存是一致的，即使每个CPU都有自己的cache。
它的实现是通过MESI协议实现的：

由下面四种状态组成，表明的是单个cacheline的状态。这是一个十分复杂的状态机，这里并不展开介绍。

 * M modified 修改的 只有一个CPU可以达到这个状态 dirty
 * E exclusive 独占的 只有一个CPU可以达到这个状态 clean
 * S shared 分享的 多个CPU可以达到这个状态 clean
 * I invalid 无效的 多个CPU可以达到这个状态

上面的dirty和clean指的是内存和cacheline的内容是否一致。

##volatile
他提供的功能仅仅是在编译期的，这个关键字可以保证：

 * 在编译期变量不会被放到寄存器中，一定会使用内存。
 * 多次重复访问内存的操作也不会被优化为仅仅读取一次。
 * 对两次volatile变量的访问之间提供了full complier memory barrier

对于有时希望volatile，有时不希望volatile，访问同一个变量时可以利用如下方式：

```
#define ACCESS_ONCE(x) (* (volatile typeof(x) *) &(x))
```

##complier memory barrier
在现代的编译器中，对于认为单线程没有因果关系的语句会做乱序处理。

对于volatile只能保证几个volatile变量的几次访问之间的顺序，其他的顺序并不能保证，为了弥补这个不足则出现了下面这个用法。

```
__asm__ __volatile__("": : :"memory"); 
```

利用上面这条语句可以实现complier memory barrier，它告诉编译器，内存发生了变化，它会生成一些额外的代码将所有寄存器的值去内存中重新读取一遍。因而存在因果关系防止编译器乱序。他是一种full complier memory barrier。

为了防止编译器在多线程中出现在不应该乱序的地方乱序我们可以插入编译器级别的memory barrier。

##CPU memory barrier
###1.防止指令乱序编译
在CPU memory barrier中隐含了complier memory barrier的语义，也就是说保证不会在编译期出现乱序。

###2.防止指令乱序执行
在现代CPU中，对于认为单线程没有因果关系的语句会乱序执行，但这在多线程中有可能出现问题。

例如：在存在指令乱序执行的情况下，下面这段代码可能被乱序，而使a, b在执行之后的值不一定相等。

```
//编制之后的指令序列
CPU0:
lock.lock()
a=1
b=1
lock.unlock()
CPU1:
lock.lock()
a=0
b=0
lock.unlock()
```
```
//实际执行的指令序列
CPU0:
a=1
b=1
lock.lock()
lock.unlock()
CPU1:
lock.lock()
lock.unlock()
a=0
b=0
```

因此需要memory barrier的防止指令乱序的指令，acquire barrier，release barrier，full barrier。

 * acquire barrier可以防止barrier下面的语句被拉上去。
 * release barrier可以防止barrier上面的语句被拉下来。
 * full barrier可以让barrier上面和下面的语句顺序隔离，上面的不会被拉下来，下面的也不会被拉上去。因而可以说full barrier=acquire barrier+release barrier。

对于上面的lock()和unlock()，如果带有acquire barrier和release barrier则可以防止CPU指令的乱序执行。

###3.保证cache一致性
对于acquire barrier和release barrier构成了一对happen before关系。有其中一个另外一个就需要配对出现。而他们的配对出现也保证了cache一致性的实现。

在CPU中，有两个重要结构：

* 一个是store buffer，写入内存的指令在指令完成的时候值只是写入到store buffer的。
* 一个是invalid queue，它用于存放当前失效的cacheline编号。

对于两个CPU一个写（CPU0）一个读（CPU1）的情形：

CPU0写的过程中会首先会写到store buffer，在遇到release barrier之后，才会将store buffer中的数据写到cacheline里面他的状态变成M（modified）并且稍后会写到内存中，然后发一条消息给其他的CPU，告诉他们那些cacheline要失效并且放到他们的invalid queue中。

CPU1在遇到acquire barrier之后，先会将invalid queue清空，并将自己的cache中指定的cacheline置为I，然后就开始读取变量对应的cacheline，发现是I（invalid），所以去别的CPU的cache中查找，并且在CPU0的那里找到状态为M的cacheline中有需要的变量，读取之并且返回，完成读取过程。

如果没有上面barrier的保证的话，就会出现CPU0写入后一直放在自己的store buffer中，而CPU1读取一直读自己cacheline上面的旧数据的情况。
##cas
对于cas操作来说，本身是带有full barrier语义的。但是它是写者将自己的cacheline先置为E让自己独占并发消息让其他CPU变为I而自己写完之后变为M。
##lock free
利用原子操作去避免锁的使用，可以保证不会出现死锁，让系统持续前进。一般都是乐观锁的实现，先做操作，做完之后在check，如果check失败就重做。
##wait free
所有线程都可以在有限的时间内完成任务。
##futex
在它出现之前可用的有spinlock和信号量，spinlock就是在用户态持续检查，而信号量则是每一次都需要进入内核。
而futex是linux的新同步原语，先会在用户态用原子操作进行检查，如果发现不能获得锁，则进入内核并wait，在用户态检查和进内核wait的空隙会有变化，在真正wait之前还会继续检查一次是否真正需要wait，如果不需要则可以直接拿锁，否则则需要wait。
##致谢
感谢孝尼大神 http://weibo.com/yebangyu
