在阅读CoreFoundation开源代码时，有c函数:
```c
static void __CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION__() __attribute__((noinline));
static void __CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION__(void (*perform)(void *), void *info)
{
    if (perform)
    {
        perform(info);
    }
    asm __volatile__(""); // thwart tail-call optimization
}
```
这是Runloop在对版本0类型的source进行回调的函数，实现较简单,即安全地调用perform.
但是最后一句代码
```c
asm __volatile__(""); // thwart tail-call optimization
```
的用意何在，百思不解。注释的意思是:"阻止尾部调用优化".

并且，其他几个函数:
```c
static void __CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE1_PERFORM_FUNCTION__() __attribute__((noinline));
static void __CFRUNLOOP_IS_CALLING_OUT_TO_A_BLOCK__() __attribute__((noinline));
static void __CFRUNLOOP_IS_CALLING_OUT_TO_A_TIMER_CALLBACK_FUNCTION__() __attribute__((noinline));
static void __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__() __attribute__((noinline));
```

在函数尾部均有`asm __volatile__(""); // thwart tail-call optimization`

回到`__CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION__`,为了弄清楚:编译器究竟会怎么进行"尾部调用优化"?,优化和不优化的区别是什么?，带着疑惑进行如下操作：

1. 将`__CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION__`函数声明以及实现复制到一个demo工程并粘贴到main.m 

2. 将xcode DEBUG优化选项设置为-O2 (开了优化才能看到究竟会怎样优化), 将arm64架构去掉(arm32汇编比arm64汇编易懂)
 
3. 调用`__CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION__`

例子如下:
```c
static void __CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION__() __attribute__((noinline));
static void __CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION__(void (*perform)(void *), void *info)
{
    if (perform)
    {
        perform(info);
    }
    asm __volatile__(""); // thwart tail-call optimization
}

static void my_perform0(void *p) __attribute__((noinline));
static void my_perform0(void *p) {
    NSLog(@"%s", __FUNCTION__);
}

static void my_perform1(void *p) __attribute__((noinline));
static void my_perform1(void *p) {
    NSLog(@"%s", __FUNCTION__);
}

static void my_perform2(void *p) __attribute__((noinline));
static void my_perform2(void *p) {
    NSLog(@"%s", __FUNCTION__);
}

int main(int argc, char * argv[])
{
    __CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION__(my_perform0, "aaa");
    __CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION__(my_perform1, "bbb");
    __CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION__(my_perform2, "ccc");
    
    return 0;
 }
```
生成app，拖进反汇编工具Hopper，得到`___CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION__`实现如下
```assembly

             ___CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION__:
0000c1a8         mov        r2, r0;arm32汇编调用约定里R0装有第一个参数，R1第二个，R2第三个，R3第四个，其余放在栈空间
0000c1aa         cbz        r2, loc_c1b8;if R2 == 0 jump to loc_c1b8, 即如果perform为NULL，跳到loc_c1b8,否则继续下一条指令

0000c1ac         push       {r7, lr};保护R7,LR
0000c1ae         mov        r7, sp;R7 = SP,建立栈帧
0000c1b0         mov        r0, r1;R0 = R1 = 第二个参数 = info
0000c1b2         blx        r2;子程序调用，地址在R2，即调用perform, 相当于bx perform 
0000c1b4         pop.w      {r7, lr};恢复R7,LR

             loc_c1b8:
0000c1b8         bx         lr;lr装有本次调用的返回地址，bx lr就是 return
```
调用栈截图: 

![Image](http://oem96wx6v.bkt.clouddn.com/屏幕快照%202017-07-13%20上午11.33.48.png)


**然后，注释掉`asm __volatile__("");`**

再次生成，得到`___CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION__`:

```assembly

             ___CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION__:
0000c1b0         mov        r2, r0
0000c1b2         cmp        r2, #0x0
0000c1b4         it
0000c1b6         bx         lr;如果R2( = R0 = 第一个参数 = perform) == NULL则 return
                        ; endp
0000c1b8         mov        r0, r1
0000c1ba         bx         r2;直接跳转到perform函数入口，参数为R0(= R1 = info),相当于bx perform
```

此时调用栈截图:

![Image](http://oem96wx6v.bkt.clouddn.com/屏幕快照%202017-07-13%20上午11.40.44.png)



神奇的事情发生了，在注释了`asm __volatile__("");`(进行过尾部调用优化)的版本里，调试器调用栈里是没有`___CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION__`的。换句话说，在被优化的版本里，调试器是获取不到 `___CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION__`的栈帧的。

由此引出了一个问题:"获取函数调用栈的原理是什么?",找了篇资料，是linux平台的，但是原理却是说得清晰的:

http://blog.csdn.net/study_live/article/details/43274271

总结说来，每个函数要建立自己的“栈帧”，才会在获取调用栈的时候被识别到。

而在ARM32下，R7正是用作禎指针(BP), 那么怎么算建立了栈帧呢，其实就是这两条指令:

```assembly
push       {r7, lr}；//保存上一级的栈帧
mov        r7, sp;//建立自己的栈帧
```

“这样就算建立起栈帧了？”，深入研究一下刚刚提到的这篇文章，就会理解了。

准备回到原例子继续分析，但是在这之前，还需要插入一个点:"`BLX`指令与`BX`指令"， 查ARM文档得:

BLX (register)
Branch with Link and Exchange (register) calls a subroutine at an address and instruction set specified by a register.

BX
Branch and Exchange causes a branch to an address and instruction set specified by a register.

除了具有ARM，THUM指令集的切换功能外，BLX是带链接（返回地址）的跳转，即子程序调用(call)，执行BLX相当于:

1. 将BLX下一条指令的地址， 即返回地址保存到LR
2. 跳转到函数入口

而BX仅仅是跳转到指定地址执行(JMP)。

正式回到 `___CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION__`进行分析:

当通过`asm __volatile__("");`阻止了尾部调用优化后,生成的汇编代码中，调用perform的方式是 :
```assembly
push       {r7, lr};
mov        r7, sp;
mov        r0, r1;
blx        r2; 
pop.w      {r7, lr};
```
进入perform前，`___CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION__`栈帧已经建立，所以在调试器里，可以识别出perform的上一级函数是:`___CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION__`

反之，去掉`asm __volatile__("");`，编译器对`___CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION__`进行了优化，导致在`___CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION__`里，不再建立调用栈帧，而是直接跳转到perform入口地址，再结合前面说到的BX指令的功能，在perform里“看来”，此时的上一级栈帧，以及返回地址跟`___CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION__`毫无关系，在此例中在perform“看来”,它的上一级调用函数应该是"main"。因此此时从调试器里是看不到`___CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION__`
的。

再提一个问题：“为什么要进行尾部调用优化?”,这恐是编译器的某种规则，具体细节,由于才疏学浅，我也无法准确回答.但是只要GET到一个重点:

`asm __volatile__("");`放在函数尾部，确实阻止了编译器对`___CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION__`进行优化，而这种优化会导致在调试器调用栈里看不到`___CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION__`函数。但是，这种优化是不影响程序功能的，因为它还是更简洁正确地跳转到了perform。但是如果不阻止，是一定会被优化的，因为苹果发布出来的库，肯定是Relase版本的，Release版本默认的优化级别很高。

可以猜测，苹果之所以将这些函数命名的如此特殊便于识别:
```c
static void __CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE1_PERFORM_FUNCTION__() __attribute__((noinline));
static void __CFRUNLOOP_IS_CALLING_OUT_TO_A_BLOCK__() __attribute__((noinline));
static void __CFRUNLOOP_IS_CALLING_OUT_TO_A_TIMER_CALLBACK_FUNCTION__() __attribute__((noinline));
static void __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__() __attribute__((noinline));
```

并且将其强制"noinline"，又阻止编译器进行”尾部调用优化“,有很大的原因是为了在其他模块，或app，在与Runloop模块交互的时候，整个调用路径“看起来/调试起来”更加清晰分明。

## 总结:
**如果不加`asm __volatile__("");`来阻止优化，经高级别优化过的库，在使用者的调试器调用栈里将看不到`___CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION__`.**
