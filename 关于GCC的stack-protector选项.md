---
title: 关于GCC的stack-protector选项
date: 2019-06-02 08:22:06
tags: CSDN迁移
---
 版权声明：本文为博主原创，无版权，未经博主允许可以随意转载，无需注明出处，随意修改或保持可作为原创！ https://blog.csdn.net/dog250/article/details/90735908   
  初看起来，觉得GCC的-fstack-protector选项并不是很好用，它对栈帧的保护程度非常有限！

 该选项只能发现 _**guard被冲刷掉的情况**_ 。面对精准的返回地址替换，guard变量是无能为力的，比如下面：

 
```
#include <stdio.h>
#include <stdlib.h>

unsigned long orig;

void stub_func()
{
	unsigned long *p;
	printf("stub\n");
	exit(0);
}


void func(int n)
{
	printf("call function\n");
	asm (
		"movq %0, %%r12 \n\t"
		"mov %%r12, 8(%%rbp)\n\t"
		:
		: "m" (orig));
}

int main()
{
	orig = stub_func;
	func(2);
	printf("function return\n");

	return 0;
}

```
 执行一下：

 
```
[root@localhost make_1]#
[root@localhost make_1]# gcc -fstack-protector-all new.c
new.c: 在函数‘main’中:
new.c:26:7: 警告：赋值时将指针赋给整数，未作类型转换 [默认启用]
  orig = stub_func;
       ^
[root@localhost make_1]# ./a.out
call function
stub

```
 这其实很容易理解，看一下汇编便知：

 
```
00000000004005f8 <func>:
  4005f8:   55                      push   %rbp
  4005f9:   48 89 e5                mov    %rsp,%rbp
  4005fc:   48 83 ec 20             sub    $0x20,%rsp
  400600:   89 7d ec                mov    %edi,-0x14(%rbp)
  400603:   64 48 8b 04 25 28 00    mov    %fs:0x28,%rax
  40060a:   00 00
  40060c:   48 89 45 f8             mov    %rax,-0x8(%rbp)
	...
  400627:   48 8b 45 f8             mov    -0x8(%rbp),%rax
  40062b:   64 48 33 04 25 28 00    xor    %fs:0x28,%rax
  400632:   00 00
  400634:   74 05                   je     40063b <func+0x43>
  400636:   e8 65 fe ff ff          callq  4004a0 <__stack_chk_fail@plt>
  40063b:   c9                      leaveq
  40063c:   c3                      retq

```
 只是把一个固定的随机数放在了栈帧上而已。

 而我们知道，有意而为的栈溢出的最终目的是替换RBP下面的返回地址，由于攻击者无法直接修改程序代码，比如像我这样嵌入内联汇编这样，唯一的方法就是找到字符串拷贝的漏洞，发送刻意构造的字符串，利用拷贝漏洞将栈溢出，从而使得程序执行ret指令时，跳转到自己的逻辑。

 因此，字符串缓冲区拷贝并不是目的，而只是一个攻击途径，几乎是唯一的途径。

 为什么不直接针对目标进行保护，让函数返回地址参与到guard计算中呢？

 其实根本没有必要。如果攻击者具有一次性精确打击返回地址的能力，那么他便同时具备了绕过guard校验的能力，比如直接ret。然而攻击者不是代码的作者，他们只能操作数据，而不能操作代码。

 栈上(以及堆上)的数据缓冲区是沟通数据和代码的纽带，不管哪种溢出攻击都利用了这一点：

  
  * 数据和逻辑元数据统一保存在连续的内存中。 
  * 数据拷贝以某种刻意的方式冲刷掉逻辑元数据。 
  * 代码使用了错误的元数据导致了错误的结果。  stack-protector选项更多的就是对付这种 _**利用数据而不是修改代码的攻击者**_ 的，而不是为了对付我这样的本地代码编写者，比如说，我就是故意写这样的程序来达到一些有益的目的的。我不是攻击者，我是代码的作者。

 换句话说， _**攻击者没有精准打击函数返回地址的能力，他们只能实施大摆拳式的栈缓冲区溢出！**_

 所以，一个随机数作为guard就够了！

 如果不理解这些，可能大部分会觉得让函数返回地址直接参与guard值计算会更 _**安全**_ 些。确实更安全，但却没必要。

 
--------
 那么为什么%FS:0x28里会保存着一个随机数呢？如果你strace程序，就会发现：

 
```
[root@localhost make_1]# strace -e trace=arch_prctl ./a.out
arch_prctl(ARCH_SET_FS, 0x7fef43a5c740) = 0
call function
stub
+++ exited with 0 +++

```
 FS和GS一样，都是段寄存器，但是却不像CS，DS，SS那样有明确的用途，在内存模式早就平坦化的现在，这些寄存器的原始意义已经不大，换句话说，FS和GS寄存器，操作系统可以随意用它们。

 比如Windows的结构化异常处理的数据结构就是保存在FS中的，而Linux的FS则保存一些TLS相关的数据，栈帧的guard就在其中。

 如strace所示，每一个程序一开始arch_prctl会设置FS的地址，然后程序就可以在这个地址处存值了。

 让我们探究一番，直接看代码：

 
```
#include <stdio.h>
#include <stdlib.h>

unsigned long addr, guard = 0;

void func()
{
	// 用-fstack-protector-all选项编译，获取rbp上面的guard值
	asm (
		"movq -8(%%rbp), %%r12 \n\t"
		"movq %%r12, %0 \n\t"
		: "=m"(guard)
		:);
	printf("guard value at [%fs:0x28] is:%llx\n", guard);
}

#define GET_FS	0x1003

int main()
{
	unsigned long *off;
	arch_prctl(GET_FS, &addr);

	off = (unsigned long *)addr;;
	off += 0x28/sizeof(unsigned long);

	// 从FS的0x28偏移获取值
	printf(" %llx\n", *off);
	func();
	
	return 0;
}

```
 两个值应该相等的：

 
```
[root@localhost make_1]# ./a.out
 3326e4ab95fc9100
guard value at [0.000000s:0x28] is:3326e4ab95fc9100

```
 OK，我们从当前进程本身来把TLS数据给篡改了试试看：

 
```
#include <stdio.h>
#include <stdlib.h>

#define GET_FS	0x1003
unsigned long addr, guard = 0;
unsigned long *off;

void func()
{
	int p;
	asm (
		"movq -8(%%rbp), %%r12 \n\t"
		"movq %%r12, %0 \n\t"
		: "=m"(guard)
		:);
	*off = 0; // 这个相当于修改了fs在0x28偏移的值
	printf("guard value at [%fs:0x28] is:%llx\n", guard);
}


int main()
{
	arch_prctl(GET_FS, &addr);

	off = (unsigned long *)addr;;

	off += 0x28/sizeof(unsigned long);
	printf(" %llx\n", *off);
	func();

	return 0;
}

```
 结果是：

 
```
[root@localhost make_1]# ./a.out
 dea850fc2de9ef00
guard value at [0.000000s:0x28] is:dea850fc2de9ef00
*** stack smashing detected ***: ./a.out terminated
======= Backtrace: =========
/lib64/libc.so.6(__fortify_fail+0x37)[0x7f11d3e959e7]
/lib64/libc.so.6(+0x1179a2)[0x7f11d3e959a2]
./a.out[0x40063a]
./a.out[0x4006ad]
/lib64/libc.so.6(__libc_start_main+0xf5)[0x7f11d3da03d5]
./a.out[0x400519]
======= Memory map: ========
00400000-00401000 r-xp 00000000 fd:00 117053769                          /root/AC/tcp_ac/src/7u/test/make_1/a.out
00600000-00601000 r--p 00000000 fd:00 117053769                          /root/AC/tcp_ac/src/7u/test/make_1/a.out
00601000-00602000 rw-p 00001000 fd:00 117053769                          /root/AC/tcp_ac/src/7u/test/make_1/a.out
020bd000-020de000 rw-p 00000000 00:00 0                                  [heap]
7f11d3b68000-7f11d3b7d000 r-xp 00000000 fd:00 94335                      /usr/lib64/libgcc_s-4.8.5-20150702.so.1
7f11d3b7d000-7f11d3d7c000 ---p 00015000 fd:00 94335                      /usr/lib64/libgcc_s-4.8.5-20150702.so.1
7f11d3d7c000-7f11d3d7d000 r--p 00014000 fd:00 94335                      /usr/lib64/libgcc_s-4.8.5-20150702.so.1
7f11d3d7d000-7f11d3d7e000 rw-p 00015000 fd:00 94335                      /usr/lib64/libgcc_s-4.8.5-20150702.so.1
7f11d3d7e000-7f11d3f40000 r-xp 00000000 fd:00 76401                      /usr/lib64/libc-2.17.so
7f11d3f40000-7f11d4140000 ---p 001c2000 fd:00 76401                      /usr/lib64/libc-2.17.so
7f11d4140000-7f11d4144000 r--p 001c2000 fd:00 76401                      /usr/lib64/libc-2.17.so
7f11d4144000-7f11d4146000 rw-p 001c6000 fd:00 76401                      /usr/lib64/libc-2.17.so
7f11d4146000-7f11d414b000 rw-p 00000000 00:00 0
7f11d414b000-7f11d416d000 r-xp 00000000 fd:00 76111                      /usr/lib64/ld-2.17.so
7f11d435a000-7f11d435d000 rw-p 00000000 00:00 0
7f11d4369000-7f11d436c000 rw-p 00000000 00:00 0
7f11d436c000-7f11d436d000 r--p 00021000 fd:00 76111                      /usr/lib64/ld-2.17.so
7f11d436d000-7f11d436e000 rw-p 00022000 fd:00 76111                      /usr/lib64/ld-2.17.so
7f11d436e000-7f11d436f000 rw-p 00000000 00:00 0
7ffe5f754000-7ffe5f775000 rw-p 00000000 00:00 0                          [stack]
7ffe5f7ce000-7ffe5f7d0000 r-xp 00000000 00:00 0                          [vdso]
ffffffffff600000-ffffffffff601000 r-xp 00000000 00:00 0                  [vsyscall]
已放弃

```
 我想说的是，我们也可以在这些寄存器里藏匿一些值的。

 
--------
 浙江温州皮鞋湿，下雨进水不会胖。

   
  