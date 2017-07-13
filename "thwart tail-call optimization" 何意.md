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
这是Runloop在对版本0类型的source进行回调的函数，实现简单即安全地调用perform.
但是最后一句代码
```c
asm __volatile__(""); // thwart tail-call optimization
```
的用意何在，百思不解。注释的意思是:"阻止尾递归调用优化".

并且，其他几个函数:
```c
static void __CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE1_PERFORM_FUNCTION__() __attribute__((noinline));
static void __CFRUNLOOP_IS_CALLING_OUT_TO_A_BLOCK__() __attribute__((noinline));
static void __CFRUNLOOP_IS_CALLING_OUT_TO_A_TIMER_CALLBACK_FUNCTION__() __attribute__((noinline));
static void __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__() __attribute__((noinline));
```

在函数尾部均有`asm __volatile__(""); // thwart tail-call optimization`



一说起递归，谈的最多的就是递归对栈空间的消耗问题，如果递归过深，很容易爆栈，比如不小心写了这样的代码:
```Objective-C
- (void)viewDidLoad {
    [self viewDidLoad];
    //xxxxx
}
```
瞬间就崩了。

那何为尾递归调用优化&何为尾递归?

https://stackoverflow.com/questions/310974/what-is-tail-call-optimization
http://blog.csdn.net/mr_listening/article/details/51106535

看了两篇资料后，总结起来就是“某种递归的算法+写法 可以被优化成循环”。如果编译器将递归优化成了循环，自然就不再有暴增的栈开销。

回到`__CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION__`,这个函数其实跟递归扯不上关系，何来的尾递归优化，带着疑惑进行如下操作：

1. 将`__CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION__`函数声明以及实现复制到一个demo工程并张贴到main.m 

2. 将xcode DEBUG优化选项设置为-O2 (开了优化才能看到究竟会怎样优化), 将arm64架构去掉(arm32汇编比arm64汇编易懂)
 
3. 调用`__CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION__`,

即:
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
0000c1ae         mov        r7, sp;R7 = SP
0000c1b0         mov        r0, r1;R0 = R1 = 第二个参数 = info
0000c1b2         blx        r2;子程序调用，地址在R2，即调用perform
0000c1b4         pop.w      {r7, lr};恢复R7,LR

             loc_c1b8:
0000c1b8         bx         lr;lr装有本次调用的返回地址，bx lr就是 return
```
调用栈截图: 

![Image](http://oem96wx6v.bkt.clouddn.com/屏幕快照%202017-07-13%20上午11.33.48.png)


然后，注释掉`asm __volatile__("");`再次生成，得到`___CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION__`:

```assembly

             ___CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION__:
0000c1b0         mov        r2, r0                                              ; CODE XREF=_main+32, _main+56, _main+80
0000c1b2         cmp        r2, #0x0
0000c1b4         it
0000c1b6         bx         lr
                        ; endp
0000c1b8         mov        r0, r1
0000c1ba         bx         r2
```

此时调用栈截图:

![Image](http://oem96wx6v.bkt.clouddn.com/屏幕快照%202017-07-13%20上午11.40.44.png)


