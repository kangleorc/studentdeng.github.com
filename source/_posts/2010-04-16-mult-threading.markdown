---
layout: post
title: "多线程程序设计笔记一"
date: 2010-04-16 18:11
comments: true
categories: [windows, c++]
---
多线程编程，学习了一个星期，总结一下。以下内容全部基于windows操作系统。  由于实力有限，对操作系统没有深入了解（或是根本不了解吧），一下内容主要来自《windows核心编程》、《win32多线程程序设计》、一些网上高人的见解和自己的理解而自己的认识难免有些偏差，希望大家挑挑毛病。谢谢。

# 进程和线程的区别。

进程通常被定义为一个正在运行的程序实例，它由两个部分组成

1.操作系统用来管理进程的内核对象。内核对象也是系统用来存放关于进程的统计信息的地方。
2.地址空间，它饱含所有可执行模块或DLL模块的代码和数据。它还包含动态内存分配的空间。如线程堆栈和堆分配空间。

线程是CPU调度的基本单位。由两个部分组成的

1. 一个是线程的内核对象，操作系统用它来对线程实施管理。内核对象是系统用来存放线程统计信息的地方。
2. 线程堆栈，它用于维护线程在执行代码时需要的所有函数参数和局部变量。

单个进程可能包含多个线程，而线程都“同时”执行进程地址空间中的代码，为此每个线程都有它自己的一组CPU寄存器（称为线程的上下文）和它自己的堆栈。进程是不活泼的，从不执行任何东西，它只是线程的容器。操作系统为每个线程安排一定的CPU时间片，仿佛所有线程都是同时运行一样。在同一个进程下的多个线程，能够执行相同的代码，对相同的数据进行操作，共享内核对象句柄。

# Context Switch

在一个抢占式多任务操作系统中，操作系统确保每个线程都有机会执行，它依赖硬件的协助以及其他的工作。当硬件计时器认为某个线程已经执行够久了，就会发出一个中断，与之CPU保存线程的当前状态，把所有寄存器内容复制到堆栈中，再把它从堆栈中复制到CONTEXT结构中。操作系统通过恢复CONTEXT结构中的寄存器值来切换不同的线程。（当然得先切换到该线程隶属的进程内存）。这个过程为Context Switch 。当然如果有非常多的CPU，也就不需要进行Context Switch。

# 使用多线程的原因。

1. 和进程相比，它是一种非常"节俭"的操作方式。启动一个线程所花费的空间远远小于启动一个进程所花费的空间，而且，线程间彼此切换所需的时间也远远小于进程间切换所需要的时间。
2. 线程间方便的通信机制。同一进程下的线程之间共享数据空间，所以一个线程的数据可以直接为其它线程所用。
3. 提高应用程序响应。这对图形界面的程序尤其有意义，当一个操作耗时很长时，整个系统都会等待这个操作，此时程序不会响应键盘、鼠标、菜单的操作，而使用多线程技术，将耗时长的操作和UI线程工作分开。
4. 线程彼此分享了大部分核心对象的拥有权。
5. 在多CPU下，多个线程可以真正的同时运行，提高了CPU的利用率。

# 多线程所带来的问题

1. 多线程在单CPU上实际上是一个假象，CPU的时间总是有限的，如果用的不好，反而增加了CPU的负担，降低了系统性能。
2. 多线程发生共享数据空间，在这个抢占式环境下线程运行次序无法预期，容易产生竞争，导致程序效率降低，出错，甚至死锁。
3. 过多的线程导致过多的Context Switch，也会降低运行效率。

讲了这么多，可见多线程程序设计给程序员带来了更高的要求。再查MSDN的时候，有的API会附上线程安全而有的却没有。下面我们开始进入多线程编程世界。

在进入多线程世界之前，先想一个问题。

设想一下，假如有一个简单的增添链表的操作

	AddHead(struct List *list,struct Node *node)
	{
	   node -> next = List->head;
	   List->head = node;
	}

这个程序若是在多线程下进行的话，会出问题。 比如当thread1 正在执行 AddHead，而Context Switch发生在node->next = List->head 和 List->head = node语句之间，而thread2也要执行AddHead，而且正确执行之后，并把一个节点添加到List中，那么当Context Switch发生，而thread1执行List->head = node后，那么thread2添加的节点就不在List中了，造成了内存泄露，而且的确，你可以想到很多出错的可能性。你可能说这个问题出现的概率很低，但是多想一下，若是这个程序是在一个搜索引擎中，每天有多少人去运行这个程序？而且CPU是以每秒千万级（我的YY，反正是很快）运行，那么程序的问题就不可避免。而之所以产生了问题，就是因为thread1和thread2产生了竞争（race condition）。

当然可能你一下子就有了思路，产生这个问题的关键是在于在发生race 的地方设置变量，来“保护”AddHead的操作正确。

	AddHead(struct List *list,struct Node *node)
	{
	   while(flag !=0)
	   flag = 1;
	   node -> next = List->head;
	   List->head = node;
	   flag = 0;
	}

这个代码似乎可以正确运行，但是很遗憾，还是不行。好吧让我们先来明确一个知识。**Atomic Operations(原子操作)**。
一下是AddHead的汇编代码

	AddHead(struct List *list,struct Node *node)
	{
	  xor eax,eax 
	  s: 
	  ;while(flag !=0) 
	  cmp DWORD PTR _flag,eax 7   
	  jne short s 
	  ;flag = 1; 
	  mov eax,DWORD PTR _list$[esp-4]
	  mov ecx,DWORD PTR _node$[esp-4]
	  mov DWORD PTR _flag,1
	  ;node -> next = List->head;
	  mov edx,DWORD PTR [eax]
	  mov DWORD PTR [ecx],edxx
	  ;List->head = node;
	  mov DWORD PTR [eax],ecx
	  ;flag = 0;
	  mov DWORD PTR _flag,0
	  ret 0
	   ;小弟还没学习完汇编，有一些操作码还是不知道，只能是理解意思，不能保证
	  ;肯定能执行，例子来自《Win32多线程程序设计》
	;}

	# Atomic Operations(原子操作)

	简单的几行代码在执行时会转换成这么多代码，不能指望C，C++这种高级语言的语句可以一次执行完毕而不发生Context Switch。

而且还存在编译器优化（比如一些循环语句等等）真实在机器上执行的代码可能和你所希望的是不同的。

原子操作（Atomic Operation ）指一个操作如果能够不受中断的完成。

所以上例的检查标记和设立标记的动作必须是一个Atomic Operation，如果中断了，就不能避免竞争。

好吧让我们真正的开始多线程编程旅行吧，我们和学习其他任何知识一样，来一个HelloWorld。

	#include <stdio.h>
	#include <stdlib.h>
	#include <windows.h>

	DWORD WINAPI ThreadFunc(LPVOID);

	int main()
	{
	    HANDLE hThrd;
	    DWORD threadId;
	    hThrd = CreateThread(NULL,
	,
	            ThreadFunc,
	,//(LPVOID)i,
	,
	            &threadId );
	    if (hThrd)
	    {
	      printf("Thread launched\n");
	      CloseHandle(hThrd);
	    }
	    //Sleep(2000);
	    return 0;
	}

	DWORD WINAPI ThreadFunc(LPVOID n)
	{
	  printf("HelloWorld\n");
	  return 0;
	}

运行后发现悲剧了,Thread launched有了,但是没有我们的熟悉HelloWorld(对大部分人来说是没机会看到HelloWorld的),那么我们又遇到什么问题了?

加上Sleep后,HelloWorld出现了,好吧,我们又发现了,创建一个线程,并写一个调用函数,和我们在main函数中写一个子程序不是一回事。

一个函数调用操作，程序的控制权转移到被调函数中，执行完毕后在返回原调用处。

产生一个线程，情况也十分类似，但是有些曲折，线程函数中我们是通过CreateThread，并传给ThreadFunc的地址。CreateThread开启一个新的线程，该线程调用ThreadFunc而原来的线程继续前进。好吧。ThreadFunc相对main来说异步执行了。那么ThreadFunc不需要在main结束之前返回，所以，main函数返回了，ThreadFunc还没来得及返回，所以没有显示HelloWorld，而Sleep之后，那么ThreadFunc返回了，那么也就显示HelloWorld。

当main函数返回后，操作系统中止整个进程，收回大部分或全部资源，而你的thread就还没工作完就被“干”掉了。

**当然你也可能会产生另一个疑问，main函数返回了，进程结束了，thread就直接悲剧了，能不能让main函数返回，进程不结束，thread不悲剧呢？**

	
	 static void Main(string[] args)
	 {
	   Thread th = new Thread(ShowWindow);
	   th.Start();
	   Console.WriteLine("窗体已创建,敲任意键退出...");
	   Console.ReadKey();
	   Console.WriteLine("Main thread End");
	 }
	  static void ShowWindow()
	 {
	   MessageBox.Show("test");
	 }

这是一段C#代码，主要来自[ref](http://blog.csdn.net/bitfan/archive/2010/01/14/5191299.aspx)我稍微改动了一点点，这段代码可以保证即使主线程退出了，只要窗体没有关闭，操作系统会认为“进程”仍在执行，因此，控制台窗口会保持显示，直到窗体关闭，整个进程才结束。在这种情况下，本示例程序中有两个UI线程，一个是控制台窗口，另一个创建应用程序窗体的那个线程。

如下是我模仿上边代码，写的c++的例子

	 int main()
	 {
	   HANDLE hThrd;
	   DWORD threadId;
	   HANDLE handle;
	   hThrd = CreateThread(NULL,
	           0,
	           ThreadFunc,
	           0,//(LPVOID)i,
	            0,
	           &threadId );
	   if(hThrd)
	   {
	     printf("Thread launched\n");
	     CloseHandle(hThrd);
	   }
	   Sleep(5000);
	   printf("Main Thread End\n");
	   return 0;
	 }
	 DWORD WINAPI ThreadFunc(LPVOID n)
	 {
	   MessageBox(NULL,"test","test",MB_OK);
	   printf("thread1 End\n");
	   return 0;
	 }

令人遗憾的是，我写的这个，在main函数返回后，整个进程也随之结束了。。。。。。

好吧。我承认我错了，到底哪里出了问题。我是搞不定了。这个花了我大概2天的时间，曾经我一度以为UI线程和Message queue之间有某种必然的联系，但是上例完全证明了我之前的想法是可笑的或是我整个思考出发点就是错误的。在thread1中不仅调用了GDI函数，而且能够响应消息（说明已经有了Message queue和Message loop）。

看来我得继续学习才能搞定这个了。