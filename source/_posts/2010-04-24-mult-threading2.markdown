---
layout: post
title: "多线程程序设计笔记二"
date: 2010-04-24 18:46
comments: true
categories: [windows, c++]
---

当我们正式开始之前，我想再多说一点，上一篇最后的那个程序可能会给像我一样的菜鸟一个误解，这里解释下。

程序启动后就执行的那个线程称为主线程（也就是那个程序中的执行main函数的线程），而其他线程则成为子线程。主线程和其他线程最大的区别是当主线程返回或是调用一些函数强制退出后，使得程序中的其他子线程强制结束。 在一篇中native code和Manager code 在遇到同样的问题时，.net给我们做了一个好的榜样，其实不管是否是UI子线程，.net 控制台在终止进程结束之前，都会保证所有子线程已经退出。虽然我们无法知道到底它采用的是什么机制（我的理解是在调用结束进程之前调用了WaitForMultipleObjects或是其他wait函数等待所有子线程返回），但是这体现了一个很重要的原则，对产生的子线程负责，无论什么时候，都不应该直接结束程序而不等待子线程结束。因为子线程没有机会做清理工作。这个是一个非常可怕的问题，想这样的一个情况，子线程比如正在申请一个堆空间，而正好锁住了那个区域，然后被强制结束了。那么就很可能产生了内存泄露（产生不了也会增大系统负担）。所以再次提醒自己主线程在退出的时候，必须保证所有的子线程已经退出。

从上一篇《多线程程序设计笔记一》中，我们知道了多线程程序设计的最基础的知识。下面总结下线程之间的通讯和同步问题。不过这个问题实在是太大了，对我来说。这里先只涉及最简单的在同一进程下的用户方式下的同步问题。

以下内容将包括：

互锁函数
临界区
互锁函数

互锁函数运行在用户模式。它能保证当一个线程访问一个变量时，其它线程无法访问此变量，以确保变量值的唯一性。

下面是一个简单的例子。

	DWORD WINAPI ThreadFunc1(PVOID n)
	{
	  while(InterlockedExchange(&g_bResourceInUse,TRUE) == TRUE)
	    SwitchToThread();
	  printf("thread1 used\n");
	  InterlockedExchange(&g_bResourceInUse,FALSE);
	  return 0;
	}
	DWORD WINAPI ThreadFunc2(PVOID n)
	{
	  while(InterlockedExchange(&g_bResourceInUse,TRUE) == TRUE)
	    SwitchToThread();
	  printf("thread2 used\n");
	  InterlockedExchange(&g_bResourceInUse,FALSE);    
	  return 0;
	}
 

这个例子通过不断地判断bResourceInUse中的信息来确定线程是否能够使用资源。但是使用这个方法必须小心。大量的循环运算会浪费宝贵的CPU时间。而且如果是在单CPU下，线程不可能真正的异步执行，在thread1判断while的时候，thread2并不能做什么（不能修改该值）。所以我们应该避免在单个CPU计算机上使用循环锁。

这里面还有一个需要知道的是必须使用关键字volatile声明g_bResourceInUse。我们需要把循环锁变量和循环锁保护的数据维护在不同的高速缓存行中。通过高速缓存行CPU可以不必访问内存总线而获得数据，但是在多处理器环境中，高速缓存行使得内存更新更加困难。如下：

CPU1读取一个字节，将该字节和相邻字节读入CPU1的高速缓存行。
CPU2读取同一个字节。从而和第一步相同的内容读入了CPU2的高速缓存行。
CPU1修改该字节，因为已经在高速缓存行中，所以修改后的内容写入CPU1的高速缓存行，这个信息还没有写入内存。
CPU2再次入去同一个字节。因为已经放入了CPU2的高速缓存行，所以CPU2不会访问内存。那么问题出现了。这个字节并不是该字节的新值。
这个问题的确很严重，不过硬件工作者已经给我们解决了这个问题，当一个CPU修改高速缓存行字节时，其他CPU会被告知这个情况，他们的高速缓存行将无效。所以第四步中，CPU1必须将高速缓存行转入内存，而CPU2必须再次访问内存。

原因想说清楚这些我现在还不行，这又涉及到了多核编程（哎，愧对老师啊，《多核程序设计》那课是白学了）。不过这里还是必须解释下volatile。

被定义为volatile的变量，每次从内存中读取，而不能把他放在cache或寄存器中重复使用。
告知编译器不要对这个变量做优化。
告知编译器，变量可以被应用程序本身以外的某个东西进行修改，这些东西包括操作系统，硬件或同时执行的线程等。
 

当必须以原子操作方式修改32为，64位值时，我们可以使用互锁函数。他们很有效率。但是实际工作，我们需要面对更复杂的数据结构。而且他们的效率是不进入内核态而节省下的。如果等待资源时间过长，就变成对CPU极大的浪费了，我们需要一种机制，使线程在等待访问共享资源时不浪费CPU时间。

临界区（Critical Section）又叫做关键代码段

临界区的描述

win32提供的一种轻量级的同步机制，它存在于进程的内存空间中。一次只有一个线程获准进入临界区执行代码段，（其实就是让若干行代码能够以原子操作方式来使用资源）。
它并不总是执行向内核模式的控制转换，要是获得一个未被占用的临界区时，只需要在用户态内的很少运算就能完成，只有在尝试获得已占用临界区时，它才会跳至内核模式。
只能在属于同一个进程的线程间同步。
补充一点，比如当线程A试图进入线程B拥有的临界区时，线程A将被置于等待状态。线程B离开临界区，线程A将处于可调度状态。让线程A立即等待，并不一定立即切换到内核方式。MS为了提高关键代码段的运行性能，将循环锁加入了这些代码段。当调用 EnterCriticalSection 它使用循环锁进行循环，只有当每次尝试获取都失败时才转入内核方式，从而线程A进入等待状态。

这里可能就又糊涂了。之前说的循环锁不是效率很低么？的确，但是如果和转入内核方式比所消耗的资源少的话，就是可行的。比如我们仅仅是想操作一个指针。当然，如果是在单CPU下，循环锁是没有意义的，会直接转入内核方式。

临界区的使用方法

        通过 InitializeCriticalSection 或 InitializeCriticalSectionAndSpinCount 函数初始化一个 CRITICAL_SECTION 结构，使用 SetCriticalSectionSpinCount 函数设置临界区的Spin计数器（可选）。然后使用 EnterCriticalSection 或 TryEnterCriticalSection 获取临界区的所有权；完成需要同步的操作后，使用 LeaveCriticalSection 函数释放临界区。最后使用 DeleteCriticalSection 函数析构临界区结构（只是删除RTL_CRITICAL_SECTION_ DEBUG）。

讲了这么多理论，实践一下。

下面是对上一篇List做的多线程改进。

	typedef struct _Node
	{
	  struct _Node *next;
	  int data;
	}Node;
	typedef struct _List
	{
	  Node *head;
	  CRITICAL_SECTION sec;
	}List;
	List *CreateList()
	{
	  List *pList = (List *)malloc(sizeof(pList));
	  pList->head = NULL;
	  InitializeCriticalSection(&pList->sec);
	  return pList;
	}
	void DeleteList(List *pList)
	{
	  DeleteCriticalSection(&pList->sec);
	  free(pList);
	  pList = NULL;
	}
	void AddHead(List *pList,Node *node)
	{
	  EnterCriticalSection(&pList->sec);
	  node->next = pList->head;
	  pList->head = node;
	  LeaveCriticalSection(&pList->sec);
	}
 

当然事实上没有这么简单。比如当交换两个链表内容的函数

	void SwapLists(List *list1,List *List2)
	{
	  List *temp_List;
	  EnterCriticalSection(list1->sec);
	  EnterCriticalSection(list2->sec);
	  tmp_List->list = list1->head;
	  list1->head = list2->head;
	  list2->head = temp->list;
	  LeaveCriticalSection(list1->sec);
	  LeaveCriticalSection(list2->sec);
	}
 

当threadA: SwapLists(list1,list2);threadB:SwapLists(list2,list1)。两个线程会落入“我等你，你等我”的轮回，这种情况称为死锁。

任何时候当一段代码需要1个以上的资源时，都可能发生死锁。而我们防止死锁通常的做法是保证“all or nothing”，也就是要不全部拥有，要不什么也没有。

其实上面的代码还隐藏了一个问题，SwapList函数在使用的时候无形的需要确保一个资源使用的顺序。也就是说这个函数的运程依赖于代码的执行顺序，这种设计本身就是很脆弱的。

临界区需要注意的

每个共享资源使用一个CRITICAL_SECTION。只有被临界区Enter和Leave“围起来”的资源才能获得保护，临界区维护的只是一段代码（代码中通常有一些资源）。
当同时访问多个资源的时候，使用临界区非常容易造成死锁， EnterCriticalSection 的顺序是需要认真考虑的但是并不一定十分可靠， LeaveCriticalSection 顺序则没有关系。
不要长时间运行临界区，也就是不要长时间锁住一个资源。但是时间到底多长很难确定在windows OS中，所以不要在一个CRITICAL_SECTION中调用Sleep或Wait….API函数，SendMessage。当你以一个同步机制保护一份资源时，必须牢记“这项资源被使用频率如何？线程必须多块释放资源，才能确保整个程序运作平顺”。
无法获知进入临界区的线程状态。由于临界区不是核心对象，如果一个线程进入临界区后，没有Leave，系统没有办法清除临界区。而且如果一个线程在Enter时被等待，那么等待的最长时间也是不能设定的。（也减少了一个处理错误的方式）
想要了解更多的关于临界区的，参考http://msdn.microsoft.com/zh-cn/magazine/cc164040(en-us).aspx

中文http://www.microsoft.com/china/MSDN/library/enterprisedevelopment/softwaredev/ousCriticalSections.mspx?mfr=true