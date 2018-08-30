# 分支预测

### static branch prediction & dynamic branch prediction

如果CPU没有足够信息，那么会使用static branch prediction，我认为dynamic xx是用在循环中的，如果是if else这种，应该只使用static xx.

Static branch prediction 的规则是依赖于处理器的，一般的规则:

1. A forward branch defaults to not taken
2. A backward branch defaults to taken (A backward branch is one that has a target address that is lower than its own address)

Forward branch 是地址在前的分支，默认不会跳转；如果地址在后的分支，默认跳转。[见wiki](https://en.wikipedia.org/wiki/Branch_predictor)

我们看循环的汇编代码会发现判断逻辑基本在后，所以这个规则一般可以理解为循环的分支处理器会认为应该继续循环。


### long __builtin_expect（long, long) 

分支预测是处理器的行为，__builtin_expect只是改变了编译器如何生成汇编代码，之所以可以work是因为刚刚讨论的static branch predition的一般规则。

在dispatch_once的源码中使用过这个方法，它用来告诉编译器生成什么样的代码，基于之前的讨论，

```
If (__builtin_expect(x, 1)) {
    xxxx
}
```

会认为x的值很可能是1，返回值也是1，则需要进入循环，那么编译器生成的汇编会把 xxxx 生成指令 在紧挨着条件判断(jmp)的代码后，跳转的地址在更高的地址处，属于forward branch，不taken，这样static branch prediction会让其进入xxxx。

```
#define likely(x)    __builtin_expect(!!(x), 1)
#define unlikely(x)  __builtin_expect(!!(x), 0)

int main(int argc, char *argv[])
{
    int a;
    
    /* Get the value from somewhere GCC can't optimize */
    a = atoi (argv[1]);
    
    if (likely (a == 2))
        a++;
    else
        a--;
    
    return a;
}

gcc xxx -O2

a.out:
(__TEXT,__text) section
_main:
0000000100000f70	pushq	%rbp
0000000100000f71	movq	%rsp, %rbp
0000000100000f74	movq	0x8(%rsi), %rdi
0000000100000f78	xorl	%eax, %eax
0000000100000f7a	callq	0x100000f96
0000000100000f7f	movl	$0x1, %ecx
0000000100000f84	cmpl	$0x2, %eax
0000000100000f87	jne	0x100000f8f
0000000100000f89	addl	%eax, %ecx
0000000100000f8b	movl	%ecx, %eax
0000000100000f8d	popq	%rbp
0000000100000f8e	retq
0000000100000f8f	movl	$0xffffffff, %ecx
0000000100000f94	jmp	0x100000f89
```

```
int main(int argc, char *argv[])
{
    int a;
    
    /* Get the value from somewhere GCC can't optimize */
    a = atoi (argv[1]);
    
    if (unlikely (a == 2))
        a++;
    else
        a--;
    
    return a;
}

_main:
0000000100000f70	pushq	%rbp
0000000100000f71	movq	%rsp, %rbp
0000000100000f74	movq	0x8(%rsi), %rdi
0000000100000f78	xorl	%eax, %eax
0000000100000f7a	callq	0x100000f96
0000000100000f7f	cmpl	$0x2, %eax
0000000100000f82	je	0x100000f8f
0000000100000f84	movl	$0xffffffff, %ecx
0000000100000f89	addl	%eax, %ecx
0000000100000f8b	movl	%ecx, %eax
0000000100000f8d	popq	%rbp
0000000100000f8e	retq
0000000100000f8f	movl	$0x1, %ecx
0000000100000f94	jmp	0x100000f89
```

一些看过的链接，随便贴贴

https://en.wikipedia.org/wiki/Branch_predictor

https://kernelnewbies.org/FAQ/LikelyUnlikely

http://linuxkernelhacker.blogspot.com/2014/07/branch-prediction-logic-in-arm.html

https://stackoverflow.com/questions/30130930/is-there-a-compiler-hint-for-gcc-to-force-branch-prediction-to-always-go-a-certa

https://software.intel.com/en-us/articles/branch-and-loop-reorganization-to-prevent-mispredicts
