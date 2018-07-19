#### 再谈\_\_bridge,\_\_bridge_transfer,\_\_bridge_retained

\_\_bridge,\_\_bridge_transfer,\_\_bridge_retained几乎是一百年多前的知识点了，关于它们在ARC下的用法网上也有大量文章，但是这东西乍一看好简单，实际上也确实简单，但过段时间不接触再来用却难免又会犯迷糊. 因此本文以更加详细的方式梳理一遍，旨在讲清楚来龙去脉，而不止是用法.

#### 0.一些说明

针对后面会用到的描述做一些说明.

CF对象:  由CoreFoundation创建,管理的对象,比如CFStringRef, CFArrayRef等，或者其他由系统SDK的C风格API创建,管理的对象，比如ABAddressBookRef.

OC对象:  就是OC对象.

所有权:  管理对象生命周期的权利, 准确说其实是: 将对象进行Retain Count -1的权利和义务.

ARC管理: 对象生命周期的retain与release操作由编译器生成的代码进行管理，不需要手动管理.

CF手动管理: 对象生命周通过手动调用CFRetain/CFRelease来管理期.

#### 1.__bridge 

**用以将 CF对象转换为OC对象，或者OC对象转换为CF对象,但是不会对对象的Retain Count,所有权 产生任何影响。**

(1)CF对象转换为OC对象

两行代码的简单例子: 

```objective-c
CFStringRef cfString = CFStringCreateMutable(kCFAllocatorDefault, 10);
NSString *ocString = (__bridge NSString *)cfString;
```

__bridge可以理解为：只是为了让编译通过,  其他毫无影响, 最终还是需要手动调用CFRelease(cfString)来释放cfString.

如果细心的跟一下，通过CFGetRetainCount，会发现cfString赋值给ocString后引用计数立即+1, 注意这跟\_\_bridge没有关系, 而是因为ARC 下 ocString默认为__strong类型,  因此在赋值给ocString前ARC生成的代码会对cfString进行Retain Count + 1操作, 最终通过objc_stroreStrong(ocString, nil) 再进行 -1操作,这个过程是ARC的本职工作范畴,跟cfString是CF对象还是OC对象以及\_\_bridge都没有关系.  

更多细节可以看这两行代码对应的汇编:

![](https://raw.githubusercontent.com/renjinkui2719/BLOG/master/PICS/__bridge_cf_to_oc.tiff)



(2)OC对象转CF对象

两行代码的简单例子: 

```objective-c
NSString *ocString = [[NSString alloc] init];
CFStringRef cfString = (__bridge CFStringRef)ocString;
```

__bridge可以理解为：只是为了让编译通过,  其他毫无影响, 不需要手动调用CFRelease(cfString)来释放cfString,因为对象的所有权没有改变,生命周期管理还是靠ARC.

而且这个赋值并不会改变ocString的Retain Count，和前面（1）的情况的差别就是，这里cfString不属于ARC管理的范畴，ARC不会为它生成管理代码.

更多细节可以看这两行代码对应的汇编:

![](https://raw.githubusercontent.com/renjinkui2719/BLOG/master/PICS/__bridge_oc_to_cf.tiff)





#### 2.\_\_bridge_transfer

**__bridge_transfer 等价于 CFBridgingRelease(),  将CF对象转换为OC对象,并将所有权转移给ARC.  所有权转移给ARC的本质含义就是：最终CF对象会被ARC生成的代码进行Retain Count -1操作或者释放，不需要手动调用CFRelease **

CFBridgingRelease中的Release不是真的会进行Release操作，而应该理解为释放所有权:本来属于CF的，现在放权给ARC，CF不管了,我猜测这也就是为什么CFBridgingRelease的对应语法关键字不叫\_\_bridge_release,而叫\_\_bridge_transfer的原因，即transfer所有权.

两行代码的简单例子: 

```objective-c
StringRef cfString = CFStringCreateMutable(kCFAllocatorDefault, 10);
NSString *ocString = (__bridge_transfer NSString *)cfString;
```

此时不能再调用CFRelease(cfString)了，否则将造成崩溃，因为通过__bridge_transfer已经将cfString所有权交给ARC，ARC会生成相应的代码对它进行管理。道理就好像你明确告诉我桌子上的苹果🍎归我吃了，但又自己悄悄吃了，我去吃的时候发现苹果🍎没了，就会崩溃的。

注: cfString 赋值给  ocString这个操作不会改变retainCount，这里虽然ocString默认是__strong属性的，但是因为 “ARC已经被\_\_bridge_transfer明确告知拥有了内存管理权”,因此编译器不会为赋值操作生成额外的retain代码.

感兴趣可以细看这两句代码对应汇编:

![](https://raw.githubusercontent.com/renjinkui2719/BLOG/master/PICS/__bridge_transfer.tiff) 

##### 使用__bridge_transfer有2个重要原则：

（1）不属于自己的CF对象不要随便给ARC，否则会造成尝试释放已释放的对象而崩溃。

比如如下代码:

```objective-c
CFArrayRef cfArray = [xxxxx];
NSString *value = (__bridge_transfer NSString *)CFArrayGetValueAtIndex(cfArray, 0);
```

这是必崩的。

根据CoreFundation内存管理的三原则：

![](https://raw.githubusercontent.com/renjinkui2719/BLOG/master/PICS/cf_mem_policy.tiff)

通过Create/Copy方法得到的对象我们是有所有权的，但是通过Get得到的，是没有所有权的.

回到这个例子,Get得到的对象,假如叫对象O,这里直接将O通过__bridge_transfer/CFBridgingRelease()交给ARC管理，ARC就会为其生成对应的释放代码,结果就是释放了不属于自己的对象O, 等cfArray真正释放的时候，也会对O进行释放操作，但其实这时候O已经被ARC给释放了，所以就崩了.  本例如果在交给ARC前通过CFRetain得到所有权，就没毛病了.

（2）属于自己的CF对象但是不想管了，需明确指定给ARC，否则内存泄漏.

比如如下代码：

```objective-c
NSString *suffix = (__bridge NSString *)ABRecordCopyValue(record, kABPersonSuffixProperty);
```

这里通过Copy方法得到了一个CFStringRef对象，是有所有权的，换句话说，是有显式CFRelease它的义务的.如果不想手动CFRelease它，那么可以将它交给ARC来管理:

```objective-c
NSString *suffix = (__bridge_transfer NSString *)ABRecordCopyValue(record, kABPersonSuffixProperty);
```

这时候就不会泄露了，因为__bridge_transfer会告诉编译器，我不想管了，你用ARC机制来处理他的生命周期吧，而前面第一种直接\_\_bridge的结果就是,Copy得到的对象永远得不到释放，因为\_\_bridge不对所有权产生任何影响。或者改成如下就没有毛病了:

```objective-c
CFStringRef cfSuffix = ABRecordCopyValue(record, kABPersonSuffixProperty);
NSString *suffix = (__bridge NSString *)cfSuffix;
CFRelease(cfSuffix);
```



#### 3.\_\_bridge_retained

\_\_bridge_retained 等价于 CFBridgingRetain (),用以将OC对象转换为CF对象，并且Retain Count + 1.

注意和__bridge_transfer转移所有权的差别，\_\_bridge_retained不存在转移什么所有权，而是简单粗暴的Retain Count + 1：编译器看到\_\_bridge_retained指示符，会生成一条对OC对象的retain语句并在赋值前调用它.因此在不需要该CF对象的时候,必须手动调用CFRelease对其进行Retain Count -1。 

 两行代码的简单例子: 

```objective-c
NSString *ocString = [[NSString alloc] init];
CFStringRef cfString = (__bridge_retained CFStringRef)ocString;
```

本例中，不需要cfString的时候，必须要CFRelease(cfString),否则cfString得不到释放，内存泄漏.

感兴趣可以细看这两句代码对应汇编:

![](https://raw.githubusercontent.com/renjinkui2719/BLOG/master/PICS/__bridge_retained.tiff)



#### 总结

三句话可以说清楚的事情写如此一大堆，只期望能对清楚的认识：\_\_bridge,\_\_bridge_transfer,\_\_bridge_retaine有所帮助.