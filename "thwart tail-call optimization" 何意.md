在阅读CoreFoundation开源代码时，有c函数:
```
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
