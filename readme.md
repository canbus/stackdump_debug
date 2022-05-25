在Windows上，通过Cygwin编译的c程序在运行时，若有内存错误也会产生类似Linux上的core文件，但是该文件一般是以stackdump为后缀的文本文件，且文件提供的信息有限，只包含了程序coredump时函数调用的栈信息，不能像Linux一样使用gdb调试。所以，在Windows平台调试Cygwin编译的c程序不太方便。本文介绍一种方法，通过反汇编c程序，结合程序coredump时生成的stackdump文件，可以快速定位出程序的coredump位置。

示例c程序如下：
```
#include <stdio.h>
#include <stdlib.h>

int f2() {
    printf("entering %s...\n", __func__);
    char *buff = "0123456789";
    free(buff);  // coredump
    printf("leaving %s...\n", __func__);
    return 0;
}

int f1() {
    printf("entering %s...\n", __func__);
    f2();
    printf("leaving %s...\n", __func__);
    return 0;
}

int main()
{
    printf("entering %s...\n", __func__);
    f1();
    printf("leaving %s...\n", __func__);
    return 0;
}
```
该程序在运行时，会产生stackdump文件。因为在f2函数中，调用free释放的内存不是由malloc分配，所以导致程序coredump。当然，这个示例程序比较简答，很容易就知道是由于free非法内存导致coredump。但如果在一个大项目中，定位coredump位置就没那么容易了。

使用Cygwin的gcc编译该程序：
```
gcc core_dump_demo.c -g -o core_dump_demo
```
这里需要使用-g选项，编译时添加调试信息，编译成功会生成一个可执行文件core_dump_demo.exe，然后使用反汇编工具objdump，将该可执行文件反汇编，运行下面命令反汇编该示例程序：
```
objdump -D -S core_dump_demo.exe > core_dump_demo.rasm
```
这里将反汇编的结果重定向到core_dump_demo.rasm文件，相关的反汇编结果，如下：
```
0000000100401080 <f2>:
   100401080:	55                   	push   %rbp
   100401081:	48 89 e5             	mov    %rsp,%rbp
   100401084:	48 83 ec 30          	sub    $0x30,%rsp
   100401088:	48 8d 05 9b 1f 00 00 	lea    0x1f9b(%rip),%rax        # 10040302a <__func__.2>
   10040108f:	48 89 c2             	mov    %rax,%rdx
   100401092:	48 8d 05 67 1f 00 00 	lea    0x1f67(%rip),%rax        # 100403000 <.rdata>
   100401099:	48 89 c1             	mov    %rax,%rcx
   10040109c:	e8 0f 01 00 00       	call   1004011b0 <printf>
   1004010a1:	48 8d 05 68 1f 00 00 	lea    0x1f68(%rip),%rax        # 100403010 <.rdata+0x10>
   1004010a8:	48 89 45 f8          	mov    %rax,-0x8(%rbp)
   1004010ac:	48 8b 45 f8          	mov    -0x8(%rbp),%rax
   1004010b0:	48 89 c1             	mov    %rax,%rcx
   1004010b3:	e8 e8 00 00 00       	call   1004011a0 <free>
   1004010b8:	48 8d 05 6b 1f 00 00 	lea    0x1f6b(%rip),%rax        # 10040302a <__func__.2>
   1004010bf:	48 89 c2             	mov    %rax,%rdx
   1004010c2:	48 8d 05 52 1f 00 00 	lea    0x1f52(%rip),%rax        # 10040301b <.rdata+0x1b>
   1004010c9:	48 89 c1             	mov    %rax,%rcx
   1004010cc:	e8 df 00 00 00       	call   1004011b0 <printf>
   1004010d1:	b8 00 00 00 00       	mov    $0x0,%eax
   1004010d6:	48 83 c4 30          	add    $0x30,%rsp
   1004010da:	5d                   	pop    %rbp
   1004010db:	c3                   	ret    

00000001004010dc <f1>:
   1004010dc:	55                   	push   %rbp
   1004010dd:	48 89 e5             	mov    %rsp,%rbp
   1004010e0:	48 83 ec 20          	sub    $0x20,%rsp
   1004010e4:	48 8d 05 42 1f 00 00 	lea    0x1f42(%rip),%rax        # 10040302d <__func__.1>
   1004010eb:	48 89 c2             	mov    %rax,%rdx
   1004010ee:	48 8d 05 0b 1f 00 00 	lea    0x1f0b(%rip),%rax        # 100403000 <.rdata>
   1004010f5:	48 89 c1             	mov    %rax,%rcx
   1004010f8:	e8 b3 00 00 00       	call   1004011b0 <printf>
   1004010fd:	e8 7e ff ff ff       	call   100401080 <f2>
   100401102:	48 8d 05 24 1f 00 00 	lea    0x1f24(%rip),%rax        # 10040302d <__func__.1>
   100401109:	48 89 c2             	mov    %rax,%rdx
   10040110c:	48 8d 05 08 1f 00 00 	lea    0x1f08(%rip),%rax        # 10040301b <.rdata+0x1b>
   100401113:	48 89 c1             	mov    %rax,%rcx
   100401116:	e8 95 00 00 00       	call   1004011b0 <printf>
   10040111b:	b8 00 00 00 00       	mov    $0x0,%eax
   100401120:	48 83 c4 20          	add    $0x20,%rsp
   100401124:	5d                   	pop    %rbp
   100401125:	c3                   	ret    

0000000100401126 <main>:
   100401126:	55                   	push   %rbp
   100401127:	48 89 e5             	mov    %rsp,%rbp
   10040112a:	48 83 ec 20          	sub    $0x20,%rsp
   10040112e:	e8 5d 00 00 00       	call   100401190 <__main>
   100401133:	48 8d 05 f6 1e 00 00 	lea    0x1ef6(%rip),%rax        # 100403030 <__func__.0>
   10040113a:	48 89 c2             	mov    %rax,%rdx
   10040113d:	48 8d 05 bc 1e 00 00 	lea    0x1ebc(%rip),%rax        # 100403000 <.rdata>
   100401144:	48 89 c1             	mov    %rax,%rcx
   100401147:	e8 64 00 00 00       	call   1004011b0 <printf>
   10040114c:	e8 8b ff ff ff       	call   1004010dc <f1>
   100401151:	48 8d 05 d8 1e 00 00 	lea    0x1ed8(%rip),%rax        # 100403030 <__func__.0>
   100401158:	48 89 c2             	mov    %rax,%rdx
   10040115b:	48 8d 05 b9 1e 00 00 	lea    0x1eb9(%rip),%rax        # 10040301b <.rdata+0x1b>
   100401162:	48 89 c1             	mov    %rax,%rcx
   100401165:	e8 46 00 00 00       	call   1004011b0 <printf>
   10040116a:	b8 00 00 00 00       	mov    $0x0,%eax
   10040116f:	48 83 c4 20          	add    $0x20,%rsp
   100401173:	5d                   	pop    %rbp
   100401174:	c3                   	ret    
 ```
在命令行中运行该示例程序，输出如下：
```
E:\share>core_dump_demo.exe
entering main...
entering f1...
entering f2...
      1 [main] core_dump_demo 5476 cygwin_exception::open_stackdumpfile: Dumping
 stack trace to core_dump_demo.exe.stackdump
 ```
并在当前目录生成一个core_dump_demo.exe.stackdump文件，内容如下：
```
Stack trace:
Frame        Function    Args
000FFFFC050  001800620B7 (000FFFFC258, 00000000002, 00000000002, 000FFFFDE50)
00000000000  001800640F5 (00000000064, 00000000000, 00000000150, 00000000000)
000FFFFC760  00180130FD8 (00000000000, 000FFFFC904, 00000000001, 001802BC140)
000000000C1  0018012C70B (00000000000, 00000000000, 00000000000, 00000000000)
000FFFFCBD0  0018012CB15 (000FFFFCA90, 00000000000, 00180159655, 00100403030)
000FFFFCBD0  001802137F8 (00000000000, 00000000000, 00180369E74, 00000000000)
000FFFFCBD0  00180213C55 (001801BD29F, 00000000000, 0018022EE30, 0010040300B)
000FFFFCBD0  001800D8738 (0010040302A, 000FFFFC944, 00000000000, 00000000000)
000FFFFCBD0  0018018FBDB (0010040302A, 000FFFFC944, 00000000000, 00000000000)
000FFFFCBD0  001004010B8 (0010040302D, 000FFFFC974, 00000000000, 000FFFFCC30)   call   1004011a0 <free>
000FFFFCC00  00100401102 (00100403030, 00000000000, 00000000000, 000FFFFCD30)   call   100401080 <f2>
000FFFFCC30  00100401151 (00180049B21, 00180048A70, 00000000002, 00180323FC0)   call   1004010dc <f1>
000FFFFCD30  00180049B8D (00000000000, 00000000000, 00000000000, 00000000000)
000FFFFFFF0  00180047746 (00000000000, 00000000000, 00000000000, 00000000000)
000FFFFFFF0  001800477F4 (00000000000, 00000000000, 00000000000, 00000000000)
End of stack trace
```
可以看到，该文件只提供了程序在coredump时函数调用的栈信息。如果只看这个stackdump文件，没法看出程序具体在哪个位置coredump。通过分析该文件，可以看见文件中的函数地址主要有2个段，分别是：

00180xxxxxx
00100xxxxxx
从反汇编文件中可以看到，00100xxxxxx地址段是示例程序中函数地址，而00180xxxxxx地址段应该是Cygwin库函数地址段。由于栈是先进后出，所以在stackdump文件中，从下往上才是函数的调用顺序。在反汇编文件中查找coredump时最后调用的地址00100401112，就可以定位出具体的coredump位置了。这里需要指出，反汇编文件中的函数地址段没有前2个0，所以在反汇编文件查找00100401112时要省去前面2个0，经过查找，可以看到该地址位于函数f2。如下所示：
```
0000000100401080 <f2>:
...
   1004010b3:	e8 e8 00 00 00       	call   1004011a0 <free>
   1004010b8:	48 8d 05 6b 1f 00 00 	lea    0x1f6b(%rip),%rax        # 10040302a <__func__.2>
   1004010bf:	48 89 c2             	mov    %rax,%rdx
...   

00000001004010dc <f1>:
...
   1004010f8:	e8 b3 00 00 00       	call   1004011b0 <printf>
   1004010fd:	e8 7e ff ff ff       	call   100401080 <f2>
   100401102:	48 8d 05 24 1f 00 00 	lea    0x1f24(%rip),%rax        # 10040302d <__func__.1>
   100401109:	48 89 c2             	mov    %rax,%rdx
...

0000000100401126 <main>:
...
   100401147:	e8 64 00 00 00       	call   1004011b0 <printf>
   10040114c:	e8 8b ff ff ff       	call   1004010dc <f1>
   100401151:	48 8d 05 d8 1e 00 00 	lea    0x1ed8(%rip),%rax        # 100403030 <__func__.0>
...
```
至此，就可以知道coredump位置位于地址00100401112的上一行代码，即调用free函数时coredump，如下：

```
10040110d:   e8 ce 00 00 00          callq  1004011e0 <free>
```

原出处:https://www.jianshu.com/p/4fbb0b863c6e
