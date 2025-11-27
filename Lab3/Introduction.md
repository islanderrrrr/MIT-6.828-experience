# 介绍
在本实验室中，你将实现运行受保护的用户模式环境(即“进程”)所需的基本内核设施。你将增强 JOS 内核，以设置数据结构来跟踪用户环境、创建单个用户环境、将程序映像加载到其中并启动它。你还将使 JOS 内核能够处理用户环境发出的任何系统调用，并处理它引起的任何其他异常。

注意：在本实验室中，术语“环境”和“进程”是可以互换的——它们都是指允许你运行程序的抽象。我们引入术语“环境”而不是传统的术语“进程”，是为了强调 JOS 环境和 UNIX 进程提供不同的接口，并且不提供相同的语义。

# 开始
Use Git to commit your changes after your Lab 2 submission (if any), fetch the latest version of the course repository, and then create a local branch called lab3 based on our lab3 branch, origin/lab3:

```
athena% cd ~/6.828/lab
athena% add git
athena% git commit -am 'changes to lab2 after handin'
Created commit 734fab7: changes to lab2 after handin
 4 files changed, 42 insertions(+), 9 deletions(-)
athena% git pull
Already up-to-date.
athena% git checkout -b lab3 origin/lab3
Branch lab3 set up to track remote branch refs/remotes/origin/lab3.
Switched to a new branch "lab3"
athena% git merge lab2
Merge made by recursive.
 kern/pmap.c |   42 +++++++++++++++++++
 1 files changed, 42 insertions(+), 0 deletions(-)
athena%
```

Lab 3 contains a number of new source files, which you should browse:

```
inc/	env.h	    Public definitions for user-mode environments
		用户态环境的公共定义
	trap.h	    Public definitions for trap handling
		中断（trap）处理的公共定义
	syscall.h	Public definitions for system calls from user environments to the kernel
		从用户态到内核的系统调用的公共定义
	lib.h	    Public definitions for the user-mode support library
		用户态支持库的公共定义

kern/	env.h	    Kernel-private definitions for user-mode environments
		内核私有的用户态环境定义
	env.c	    Kernel code implementing user-mode environments
		实现用户态环境的内核代码
	trap.h	    Kernel-private trap handling definitions
		内核私有的中断处理定义
	trap.c	    Trap handling code
		中断处理代码
	trapentry.S	Assembly-language trap handler entry-points
		中断处理器入口点的汇编实现
	syscall.h	Kernel-private definitions for system call handling
		内核私有的系统调用处理定义
	syscall.c	System call implementation code
		系统调用的实现代码

lib/	Makefrag	Makefile fragment to build user-mode library, obj/lib/libjos.a
		用于构建用户态库（obj/lib/libjos.a）的 Makefile 片段
	entry.S	    Assembly-language entry-point for user environments
		用户态环境的汇编入口点
	libmain.c	User-mode library setup code called from entry.S
		由 entry.S 调用的用户态库初始化代码
	syscall.c	User-mode system call stub functions
		用户态的系统调用桩函数
	console.c	User-mode implementations of putchar and getchar, providing console I/O
		用户态的 putchar 和 getchar 实现，提供控制台 I/O
	exit.c	    User-mode implementation of exit
		用户态的 exit 实现
	panic.c	    User-mode implementation of panic
		用户态的 panic 实现

user/	*	        Various test programs to check kernel lab 3 code
		各种测试程序，用于检验内核实验3的代码
```

In addition, a number of the source files we handed out for lab2 are modified in lab3. To see the differences, you can type:

```
$ git diff lab2
```

You may also want to take another look at the lab tools guide, as it includes information on debugging user code that becomes relevant in this lab.

