今天在编写nginx模块的时候遇到一个诡异的core dump，看了core位置相关的变量值后简直被惊呆了、下面使用一个可复现的列子来复现。。。


相关复现环境
====

gcc版本： 

```
gcc version 4.4.6 20110731 (Red Hat 4.4.6-3) (GCC) 
```

系统版本：

```
2.6.32-220.23.2.xxx.el6.x86_64
```


1.c 文件内容：


```
#include <stdio.h>

//extern void * get_ptr();

int main()
{
    void *p;
    
    p = NULL;

    p = get_ptr();

    printf("%#p\n", p);

    return 0;
}

```


2.c 文件内容：

```
#include <stdio.h>

void * get_ptr()
{
    int *p;


    p = (int*) 0x80000000;

    printf("%#p\n", p)

    return p;
}
```


编译：


```
gcc -g -c 1.c 
gcc -g -c 2.c 

gcc -o test 1.o  2.o

```


运行：


./test

输出：
```

0x80000000
0xffffffff80000000
```


gdb汇编分析
=======


看到输出结果诡异吧（看到高32位都被1填充了），下面就gdb一步步的分析看看：

```
Breakpoint 1, main () at 1.c:11
11        p = get_ptr();
(gdb) x/i $pc
=> 0x4004d4 <main+16>:	mov    $0x0,%eax
(gdb) si
0x00000000004004d9	11	    p = get_ptr();
(gdb) x/i $pc
=> 0x4004d9 <main+21>:	callq  0x400504 <get_ptr>
(gdb) si
get_ptr () at 2.c:4
4	{
(gdb) n
8	    p = (int*) 0x80000000;
(gdb) 
10	    printf("%#p\n", p);
(gdb) 
0x80000000
12	    return p;
(gdb) p p
$1 = (int *) 0x80000000
(gdb) info registers eax
eax            0xb	11
(gdb) x/i $pc
=> 0x400533 <get_ptr+47>:	mov    -0x8(%rbp),%rax
(gdb) si
13	}
(gdb) info registers eax
eax            0x80000000	-2147483648
(gdb) info registers rax
rax            0x80000000	2147483648
(gdb) x/i $pc
=> 0x400537 <get_ptr+51>:	leaveq 
(gdb) si
0x0000000000400538 in get_ptr () at 2.c:13
13	}
(gdb) x/i $pc
=> 0x400538 <get_ptr+52>:	retq   
(gdb) si
0x00000000004004de in main () at 1.c:11
11	    p = get_ptr();
(gdb) x/i $pc
=> 0x4004de <main+26>:	cltq              // 注意这条指令
(gdb) info registers rax
rax            0x80000000	2147483648    // 在执行cltq指令之前，rax寄存器的值还是0x80000000属于正常的
(gdb) si                                  // 执行cltq指令
0x00000000004004e0	11	    p = get_ptr();
(gdb) info registers rax
rax            0xffffffff80000000	-2147483648   // 此时发现执行cltq指令后rax寄存器的值就变为0xffffffff80000000，所以罪魁祸首就是这个cltq指令，为啥会执行这条指令
(gdb) p p
$2 = (void *) 0x0
(gdb) x/i $pc
=> 0x4004e0 <main+28>:	mov    %rax,-0x8(%rbp)
(gdb) si
13	    printf("%#p\n", p);
(gdb) p p
$3 = (void *) 0xffffffff80000000

```


回想起在编译1.c的时候包了一个warning错误
$gcc -g -c 1.c
 
```
1.c: In function ‘main’:
1.c:11: warning: assignment makes pointer from integer without a cast
```

但是看了1.c的第11行代码 `p = get_ptr();` 并没有直观的看到将一个integer赋值给一个指针恩难道是gcc编译器把函数get_ptr的返回值搞错了。。。。
带着疑问去gg发现是由于： 未申明的全局函数默认返回值是int类型的，于是申明了一下这个函数对比了一下其汇编代码：

```


+ +-- 13 lines: .file "1.c"---------------------------------|+ +-- 13 lines: .file "1.c"---------------------------------
          movq    %rsp, %rbp                                |          movq    %rsp, %rbp
          .cfi_def_cfa_register 6                           |          .cfi_def_cfa_register 6
          subq    $16, %rsp                                 |          subq    $16, %rsp
          movq    $0, -8(%rbp)                              |          movq    $0, -8(%rbp)
          movl    $0, %eax                                  |          movl    $0, %eax
          call    get_ptr                                   |          call    get_ptr
  ----------------------------------------------------------|          cltq                                              
          movq    %rax, -8(%rbp)                            |          movq    %rax, -8(%rbp)
          movl    $.LC0, %eax                               |          movl    $.LC0, %eax
          movq    -8(%rbp), %rdx                            |          movq    -8(%rbp), %rdx
          movq    %rdx, %rsi                                |          movq    %rdx, %rsi
          movq    %rax, %rdi                                |          movq    %rax, %rdi
          movl    $0, %eax                                  |          movl    $0, %eax

		 
```

的确申明了一下全局函数后，这个`cltq`汇编指令就不见了；

函数申明与未申明为啥在此场景会多一条`cltq`指令，那就先看看`ctlq`这个指令的作用是干啥的：

```
cltq R[%rax ] <- SignExtend(R[%eax]) Convert %eax to quad word

```

`cltq`之所以发生在这个场景是由于：`p = get_ptr();` 等号左边是一个64位的无符号数、而等号右边是一个32位有符号的int类型
之所以这种隐式的32位有符号数向64位无符号数转换的时候就编译器就使用了cltq汇编指令做的；


那为什么`0x80000000`变为 `0xffffffff80000000`高32为都填充的是1，这个原因就要看看32位有符号数向64位无符号转换的相关概念了（若最高位是1就使用1填充高32位、否则使用0）；


简单的总结下吧：
====  

其实这个错误归根结底还是由于使用了未申明的全局函数的问题（代码的规范性问题），其实想想这个问题还是比较可怕的（1.在32位的系统上该场景不会有问题（32位系统指针也是32位）2.如果就算在64位系统上，如果最高位是0、则高32位也填充的是0也不会出问题）对于线上出现的这种问题不容易排查；那么就警惕我们平时编译代码的时候不放过任何的warning，其实很多公司代码编译都有自动化平台了，所以一般的warning都被忽略了、所以还是建议在自己的编译选项中使用`-Werror`此选项把warning当成error处理；


欢迎一起交流学习 
====
 
在使用中有任何问题，欢迎反馈给我，可以用以下联系方式跟我交流

* 邮件(1031379296#qq.com, 把#换成@)
* QQ: 1031379296
* weibo: [@王发康](http://weibo.com/u/2786211992/home)


Thx
====

* chunshengsterATgmail.com


Author
====
* Linux\nginx\golang\c\c++爱好者
* 欢迎一起交流  一起学习# 
* Others say good and Others good



