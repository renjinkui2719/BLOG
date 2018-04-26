
#### 在iOS平台实现新的异步解决方案async/await
async,await是ES7提出的异步解决方案，对比回调链和Promise.then链的异步编程模式，基于async,await可以同步风格编写异步代码，程序逻辑清晰明了.
如顺序读取三个文件:
```JS
function readFile(name) {
  return new Promise((resolve, reject) => {
    //异步读取文件
    fs.readFile(name, (err, data) => {
        if (err) reject(err);
        else resolve(data);
    });
  });
}

async function read3Files() {
  try {
      //读取第1个文件
      let data1 = await readFile('file1.txt');
      //读取第2个文件
      let data2 = await readFile('file2.txt');
      //读取第3个文件
      let data3 = await readFile('file3x.txt');
      //3个文件读取完毕
    } catch (error) {
       //读取出错
    }
}
```
读取文件本身是异步操作，而在要求顺序读取的前提下，基于callback实现将造成很深的回调嵌套:
```JS

function readFile(name, callback) {
  //异步读取文件
  fs.readFile(name, (err, data) => {
     callback(err, data);
  });
}

function read3Files() {
  //读取第1个文件
  readFile('file1.txt', (err, data) => {
    //读取第2个文件
    readFile('file2.txt', (err, data) => {
      //读取第3个文件
      readFile('file3.txt', (err, data) => {
        //3个文件读取完毕
      });
    });
  });
}
```
基于Promise.then链需要将逻辑分散在过多的代码块:
```JS
function readFile(name) {
  return new Promise((resolve, reject) => {
    //异步读取文件
    fs.readFile(name, (err, data) => {
       if (err) reject(err);
       else resolve(data);
    });
  });
}

function read3Files() {
  //读取第1个文件
  readFile('file1.txt')
  .then(data => {
    //读取第2个文件
    return readFile('file2.txt');
  })
  .then(data => {
    //读取第3个文件
    return readFile('file3.txt');
  })
  .then(data => {
    //3个文件读取完毕
  })
  .catch(error => {
    //读取出错
  });
}
```
对比足见aync/await模式的优雅与简洁,接触完毕后，深感如果在iOS项目中也能像JS这般编写异步代码真是极好。经过研究发现要在iOS平台实现这些特性其实并不是很困难，因此本文主旨便是描述async,await在iOS平台的实现过程，并给出了一个成果项目.

### 暂时继续讨论JavaScript

#### 1.生成器与迭代器
要明白async/await的机制及运用,需从生成器与迭代器逐步说起.在ES6中，生成器是一个函数，和普通函数的区别是:

##### (1)生成器函数function关键字后多了个*:
```JS
function *numbers() {
}
```

##### (2)生成器函数内可以yield语法多次返回值:
```JS
function *numbers() {
    yield 1;
    yield 2;
    yield 3;
}
```

##### (3)直接调用生成器函数得到的是一个迭代器，通过迭代器的next方法控制生成器的执行:
```JS
let iterator = numbers();

let result = iterator.next();
```
每一次next调用将得到结果result, result对象包含两个属性:`value`和`done`.value表示此次迭代得到的结果值,done表示是否迭代结束.
比如此例:
```JS
function *numbers() {
    yield 1;
    yield 2;
    yield 3;
}

let iterator = numbers();
//第1次迭代
let result = iterator.next();
console.log(result);
//输出 => { value: 1, done: false }

//第2次迭代
result = iterator.next();
console.log(result);
//输出 => { value: 2, done: false }

//第3次迭代
result = iterator.next();
console.log(result);
//输出 => { value: 3, done: false }

//第4次迭代
result = iterator.next();
console.log(result);
//输出 => { value: undefined, done: true }
```

第1次调用next,生成器numbers开始执行,执行到第一个yield语句时，numbers将中断，并将结果值`1`返回给迭代器，由于numbers并没有执行完，所以done为false.

第2次调用next,生成器numbers从上次中断的位置恢复执行,继续执行到下一个yield语句时，numbers再次中断，并将结果值`2`返回给迭代器，由于numbers并没有执行完，所以done为false.

第3次调用next,生成器numbers从上次中断的位置恢复执行,继续执行到下一个yield语句时，numbers再次中断，并将结果值`3`返回给迭代器，由于numbers并没有执行完，所以done为false.

第4次调用next,生成器numbers从上次中断的位置恢复执行,此时已是函数尾，numbers将直接return, 由于numbers已经执行完成，所以done为true, 由于numbers并没有显式地返回任何值，因此此次迭代value为undefined.

到此迭代结束，此后通过此迭代器的next方法，都将得到相同的结果`{ value: undefined, done: true }`

##### (4)通过迭代器可向生成器内部传值
```JS
function *hello() {
  let age =  yield 'want age';
  let name = yield 'want name';
  console.log(`Hello, my age: ${age}, name:${name}`);
}

let iterator = hello();
```
创建迭代器并开始如下迭代过程:

第1次迭代,生成器开始执行，到达第一个yield语句时，返回`value = want age, done = false`给迭代器, 并中断。
```JS
let result = iterator.next();
console.log(result);
//输出 => { value: 'want age', done: false }
```
第2次迭代,给next传参28, 生成器从上次中断的地方恢复执行，并将28作为苏醒后yield的内部返回值赋给age; 然后生成器继续执行，再次遇到yield，返回`value = want name, done = false`给迭代器, 并中断。
```JS
result = iterator.next(28);
console.log(result);
//输出 => { value: 'want name', done: false }
```
第3次迭代,给next传参'LiLei', 生成器从上次中断的地方恢复执行，并将'LiLei'作为苏醒后yield的内部返回值赋给name; 然后生成器继续执行，打印log:
```
Hello, my age: 28, name:LiLei
```
然后到达函数尾，彻底结束生成器，并返回`value = undefined, done = true`给迭代器。
```JS
result = iterator.next('LiLei');
console.log(result);
//输出 => { value: undefined, done: true }
```

##### 可见通过迭代器可与生成器“互相交换数据”，生成器通过yield返回数据A给迭代器并中断，而通过迭代器又可以把数据B传给生成器并让此yied语句苏醒且以B作为右值. 这个特性是下一步"改进异步编程"的重要基础.

至此已基本了解了生成器与迭代器的语法与运用,总结起来:

 ###### 生成器是一个函数，直接调用得到其对应的迭代器，用以控制生成器的逐步执行;
 ###### 生成器内部通过yield语法向迭代器返回值，而且可以多次返回,并多次恢复执行，有别于传统函数"返回便消亡"的特点;
 ###### 可以通过迭代器向生成器内部传值，传入的值将作为本次生成器苏醒后的右值.
 
 
#### 2.通过生成器与迭代器改进异步编程
回想本文开头提到的读取文件例子,如果以callback模式编写:
```JS
function readFile(name, callback) {
  //异步读取文件
  fs.readFile(name, (err, data) => {
     callback(err, data);
  });
}

function read3Files() {
  //读取第1个文件
  readFile('file1.txt', (err, data) => {
    //读取第2个文件
    readFile('file2.txt', (err, data) => {
      //读取第3个文件
      readFile('file3.txt', (err, data) => {
        //3个文件读取完毕
      });
    });
  });
}
```
基于前面起到的"与生成器交换数据"的特性,拓展出新思路:

(1)把读取文件这个动作封装为一个异步操作，通过callback输出结果:err和data.

(2)把read3Files改变为生成器,内部通过yield返回异步操作给执行器(执行器第3步描述).

(3)执行器通过迭代器接收read3Files返回的异步操作,拿到异步操作后，发起该异步操作，得到结果后再“交换”给生成器read3Files.

即:
```JS
function readFile(name) {
  //返回一个闭包作为异步操作
  return function(callback) {
    fs.readFile(name, (err, data) => {
       callback(err, data);
    });
  };
}

function executor(generator) {
  //创建迭代器
  let iterator = generator();
  //开始第一次迭代
  let result = iterator.next();

  let nextStep = function() {
    //迭代还没结束
    if (!result.done) {
      //从生成器拿到的是一个异步操作
      if (typeof result.value === "function") {
        //发起异步操作
        result.value((err, data) => {
          if(err) {
            //在生成器内部引发异常
            iterator.throw(err);
          }
          else {
            //得到结果值,传给生成器
            result = iterator.next(data);
            //继续下一步迭代
            nextStep();
          }
        });
      }
      //从生成器拿到的是一个普通对象
      else {
        //什么都不做，直接传回给生成器
        result = iterator.next(result.value);
        //继续下一步迭代
        nextStep();
      }
    }
  };
  //开始后续迭代
  nextStep();
}

executor(function *() {
  try {
    //读取第1个文件
    let data1 = yield readFile('file1.txt');
    //读取第2个文件
    let data2 = yield readFile('file2x.txt');
    //读取第3个文件
    let data3 = yield readFile('file3.txt');
  } catch (e) {
    //读取出错
  }
});
```
此时已经把callback模式改进为同步模式。

暂且把传给执行器的生成器函数叫做"异步函数"，执行过程总结起来就是:

异步函数但凡遇到异步操作，就通过yield交给执行器, 执行器但凡拿到异步操作，就发起该操作，拿到实际结果后再将其交还给异步函数。那么在异步函数内，就可以同步风格编写异步代码，因为有了执行器在背后，异步函数内的yield就具有了“你给我异步操作,我还你实际结果”的能力.

Promoise同样可作为异步操作:
```JS
function readFile(name) {
  //返回一个Promise作为异步操作
  return new Promise((resolve, reject) => {
    fs.readFile(name, (err, data) => {
        if (err) reject(err);
        else resolve(data);
    });
  });
}
```
在执行器中新增识别Promise的代码
```JS
function executor(generator) {
  //创建迭代器
  let iterator = generator();
  //开始第一次迭代
  let result = iterator.next();

  let nextStep = function() {
    //迭代还没结束
    if (!result.done) {
      if (typeof result.value === "function") {
        ....
      }
      //从生成器拿到的是一个Promise异步操作
      else if (result.value instanceof Promise) {
        //执行该Promise
        result.value.then(data => {
          //得到结果值,传给生成器
          result = iterator.next(data);
          //继续下一步迭代
          nextStep();
        }).catch(err => {
          //在生成器内部引发异常
          iterator.throw(err);
        });
      }
      else {
        ...
      }
    }
  };
  ...
}
```

到此已经成功把异步编程化为同步风格，但或许有个疑问:这个例子倒是化为同步风格了，但是那个执行器executor看起来好大一坨，并不优雅.实际上执行器当然是复用的，不用每次都实现执行器.

#### 3.async,await语法糖
async,await终于出来,async与await是上述执行器，生成器模式的语法糖,运用async,await,再也不需要每次都定义生成器作为异步函数,然后显式传给执行器,只要简单在函数定义前增加async,如下:
```JS
async function foo() {
    let value = await 异步操作;
    let value = await 异步操作;
    let value = await 异步操作;
    let value = await 异步操作;
}
```
如读取文件例子:
```JS
async function read3Files() {
    //读取第1个文件
    let data1 = await readFile('file1.txt');
    //读取第2个文件
    let data2 = await readFile('file2.txt');
    //读取第3个文件
    let data3 = await readFile('file3x.txt');
    //3个文件读取完毕
}
```
然后直接调用即可:
```JS
read3Files();
```
async表示该函数内部包含异步操作，需要把它交给内置执行器;

await表示等待异步操作的实际结果。

至此，async/await的来龙去脉已基本描述完毕.


### 回到iOS
光描述JS生成器，迭代器，async,await就花了如此大篇幅，因为在iOS上将以它们的JS特性为目标，最终实现OC版的迭代器，生成器，async,await.

#### 1.类型定义
暂时无需在意怎么实现,既然是以前面描述的特性为目标，则可以根据其特性先做如下定义:

先定义yield如下:
```Objective-C
id yield(id value);
```
yield接受一个对象value作为返回给迭代器的值，同时返回一个迭代器设置的新值或者原本值value.

以及每次迭代的Result:
```Objective-C
@interface Result: NSObject
@property (nonatomic, strong, readonly) id value;
@property (nonatomic, readonly) BOOL done;
@end
```
value表示迭代的结果，为yield返回的对象，或者nil. done指示是否迭代结束.

根据前面描述的生成器特性,那么在OC里，生成器首先应该是一个C函数/OC方法/block,且内部通过调用yield来返回Result对象给迭代器:
```Objective-C
void generator() {
    yield(value);
    yield(value);
}

- (void)generator {
    yield(value);
    yield(value);
}

^{
    yield(value);
    yield(value);
}
```
实际上不论是OC方法，还是block,底层调用时都与调用C函数无异，所以只要实现了C函数版生成器，其实现机制将也无缝适用于OC方法，block.

迭代器定义:
```Objective-C
@interface Iterator : NSObject
{
    void (*_func)(void);
}

- (id)initWithFunc:(void (*)(void))func;

- (Result *)next;
- (Result *)next:(id)value;
@end
```
迭代器的创建无法做到像JS一样直接调用生成器即可创建，需要显式创建:

```Objective-C
void generator() {
    yield(value);
    yield(value);
}

Iterator *iterator = [[Iterator alloc] initWithFunc: generator];
```
然后就可以像JS一样调用next来进行迭代:
```Objective-C
Result *result = iterator.next;
//迭代并传值
Result *result = [iterator next: value];
```

#### 2.实现生成器与迭代器

根据需求，yield调用会中断当前执行流，并期望将来能够从中断处恢复继续执行,那么必定要在触发中断时保存现场，包括:

(1)当前指令地址

(2)当前寄存器信息，包括当前栈帧栈顶。

而且 中断后到恢复的这段时间内，应当确保yield以及生成器generator的栈帧不会被销毁.

而恢复执行的过程是保存现场的逆过程，即恢复相关寄存器,恢复跳转到保存的指令地址处继续执行.

上述过程描述起来看似简单，但是如果要自己写汇编代码去保存与恢复现场，并适配各种平台，要保证稳定性还是很难的，好在有C标准库提供的现成利器:setjmp/longjmp。

setjmp/longjmp可以实现跨函数的远程跳转，对比goto只能实现函数内跳转，setjmp/longjmp实现远程跳转给予的就是保存现场与恢复现场,非常符合此处的需求.

#### 迭代器实现
迭代器是与生成器直接交互的对象,交互方式是通过next方法，那么在next方法内部必定以某种机制将流程“切入”到生成器，而生成器通过yield将返回值交给迭代器，并将执行流切换回next方法，next方法拿到这个值，正常返回给调用者.因此为迭代器新增属性如下:
```Objective-C
@interface Iterator : NSObject
{
    int *_ev_leave; //迭代器在next方法内保存的现场
    int *_ev_entry; //生成器通过yield保存的现场
    BOOL _ev_entry_valid; //指示生成器现场是否可用
    void *_stack; //为生成器新分配的栈
    int _stack_size; //为生成器新分配的栈大小
    void (*_func)(void);
    BOOL _done; //是否迭代结束
    id _value; //生成器传给的值
}
```

为生成器分配新栈,因为next方法是要返回的，如果直接在next自己的调用栈上调用生成器，那么next返回后，生成器就算保护了寄存器现场，它的栈帧也被破坏了，再次恢复执行将产生无法预料的结果.

```Objective-C
//默认为生成器分配256K的执行栈
#define DEFAULT_STACK_SIZE (256 * 1024)

- (id)init {
    if (self = [super init]) {
        _stack = malloc(DEFAULT_STACK_SIZE);
        memset(_stack, 0x00, DEFAULT_STACK_SIZE);
        _stack_size = DEFAULT_STACK_SIZE;
        
        _ev_leave = malloc(sizeof(jmp_buf));
        memset(_ev_leave, 0x00, sizeof(jmp_buf));
        _ev_entry = malloc(sizeof(jmp_buf));
        memset(_ev_entry, 0x00, sizeof(jmp_buf));
    }
    return self;
}
```

next方法:
```Objective-C
#define JMP_CONTINUE 1
#define JMP_DONE 2

- (Result *)next:(id)value {
    if (_done) {
       //迭代器已结束
       return [Result resultWithValue:_value error:_error done:_done];
    }    
    //保存next执行环境
    int leave_value = setjmp(_ev_leave);
    //非跳转返回
    if (leave_value == 0) {
        //已经设置了生成器进入点
        if (_ev_entry_valid) {
            //直接从生成器进入点进入
            if (value) {
                self.value = value;
            }
            longjmp(_ev_entry, JMP_CONTINUE);
        }
        else {
            //wrapper进入
            
            //next栈会销毁,所以为wrapper启用新栈
            intptr_t sp = (intptr_t)(_stack + _stack_size);
            //预留安全空间，防止直接move [sp] 传参 以及msgsend向上访问堆栈
            sp -= 256;
            //对齐sp
            sp &= ~0x07;
            
#if defined(__arm__)
            asm volatile("mov sp, %0" : : "r"(sp));
#elif defined(__arm64__)
            asm volatile("mov sp, %0" : : "r"(sp));
#elif defined(__i386__)
            asm volatile("movl %0, %%esp" : : "r"(sp));
#elif defined(__x86_64__)
            asm volatile("movq %0, %%rsp" : : "r"(sp));
#endif
            //在新栈上调用wrapper,至此可以认为wrapper,以及生成器函数的运行栈和next无关
            [self wrapper];
        }
    }
    //生成器内部跳转返回
    else if (leave_value == JMP_CONTINUE) {
        //还可以继续迭代
    }
    //生成器wrapper跳转返回
    else if (leave_value == JMP_DONE) {
        //生成器结束，迭代完成
        _done = YES;
    }
        
    return [RJResult resultWithValue:_value error:_error done:_done];
}
```
调用迭代器方法next时,先保存next

为保证生成器执行完后,
```Objective-C
- (void)wrapper {
    if (_func) {
        _func();
    }
    //从生成器返回，说明生成器完全执行结束
    //直接返回到迭代器设置的返回点
    self.value = nil;
    
    longjmp(_ev_leave, JMP_DONE);
    //不会到此
    assert(0);
}
```


是




而恢复执行的过程是保存现场的逆过程，即恢复相关寄存器,恢复跳转到保存的指令地址处继续执行.guo
而恢复执行的过程是保存现场的逆过程，即恢复相关寄存器,恢复跳转到保存的指令地址处继续执行.

setjmp和longjmp

对于如下生成器:
```C
void generator() {
    yield(value);
    yield(value);
}
```
在执行进入yield后，调用栈布局应该如下:

![](http://oem96wx6v.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202018-04-26%20%E4%B8%8B%E5%8D%882.32.39.png)






假设函数foo定义如下:
```Objective-C
void foo() {
    int a = 0x10;
    int b = 0x20;
    int r = a + b;
    printf("a + b = %d", r);
}
```
对应的ARM32位汇编:
```asm
foo:
    0x82448 <+0>:  push   {r7, lr}
    0x8244a <+2>:  mov    r7, sp
    0x8244c <+4>:  sub    sp, #0x10
    0x8244e <+6>:  movw   r0, #0x37bf
    0x82452 <+10>: movt   r0, #0x0
    0x82456 <+14>: add    r0, pc;   ;定位常量字符串: "a + b = %d"
    0x82458 <+16>: movs   r1, #0x20 
    0x8245a <+18>: movs   r2, #0x10 
    0x8245c <+20>: str    r2, [sp, #0xc] ;a = 0x10
    0x8245e <+22>: str    r1, [sp, #0x8] ;b = 0x10
    0x82460 <+24>: ldr    r1, [sp, #0xc]
    0x82462 <+26>: ldr    r2, [sp, #0x8]
    0x82464 <+28>: add    r1, r2  ;r = a + b
    0x82466 <+30>: str    r1, [sp, #0x4]
    0x82468 <+32>: ldr    r1, [sp, #0x4]
    0x8246a <+34>: blx    0x84400    ;调用printf
    0x8246e <+38>: str    r0, [sp]
    0x82470 <+40>: add    sp, #0x10
    0x82472 <+42>: pop    {r7, pc}

```
假设执行指令
```
0x82458 <+16>: movs   r1, #0x20 
```




第3次调用next,生成器numbers从上次中断的位置恢复执行,继续执行到下一个yield语句时，numbers再次中断，并将结果值`3`返回给迭代器，由于numbers并没有执行完，所以done为false.

```dia
`

先了解在JS中,async和await究竟是怎样运用的.
###### 以顺序读取三个文件举例 
假设要求顺序读取三个文件，即每读取完成一个才可以开始读取下一个

(1).基于callback
```JavaScript
var fs = require('fs');

function readFile(name, callback) {
  //异步读取文件
  fs.readFile(name, (err, data) => {
     callback(err, data);
  });
}

function read3Files() {
  //读取第1个文件
  readFile('file1.txt', (err, data) => {
    //读取第2个文件
    readFile('file2.txt', (err, data) => {
      //读取第3个文件
      readFile('file3.txt', (err, data) => {
        //3个文件读取完毕
      });
    });
  });
}

read3Files();
```
(2)基于Promise
```JavaScript
function readFile(name) {
  return new Promise((resolve, reject) => {
    //异步读取文件
    fs.readFile(name, (err, data) => {
       if (err) reject(err);
       else resolve(data);
    });
  });
}

function read3Files() {
  //读取第1个文件
  readFile('file1.txt')
  .then(data => {
    //读取第2个文件
    return readFile('file2.txt');
  })
  .then(data => {
    //读取第3个文件
    return readFile('file3.txt');
  })
  .then(data => {
    //3个文件读取完毕
  })
  .catch(error => {
    //读取出错
  });
}
```
(3)基于async,await
```JavaScript
function readFile(name) {
  return new Promise((resolve, reject) => {
    //异步读取文件
    fs.readFile(name, (err, data) => {
        if (err) reject(err);
        else resolve(data);
    });
  });
}

async function read3Files() {
  try {
      //读取第1个文件
      let data1 = await readFile('file1.txt');
      //读取第2个文件
      let data2 = await readFile('file2.txt');
      //读取第3个文件
      let data3 = await readFile('file3x.txt');
      //3个文件读取完毕
    } catch (error) {
       //读取出错
    }
}

```

至此可以体会到基于async/await异步模式的清晰优雅, read3Files被标记为async,在函数内部每个异步操作的结果都可以用await来“等到”，可直接理解await含义为:"等待异步操作结果",但是这个等待过程是不阻塞的。此处的异步操作是一个Promise,由readFile函数创建并返回，当异步操作结束,promise被resolve时，await成功等到结果，而promise被reject时，await等待失败，抛出错误，并被catch到.


### 切换回iOS
#### 1.美好的假设
假设async/await/Promise已经在OC中支持，那么如上的读取文件列子即可这样实现:
```Objective-C
- (Promise *)readFileWithPath:(NSString *)path {
    return [Promise promiseWithBlock: ^(void (^resolve)(id data), void (^reject)(id err)) {
        dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
            NSData *data =  [NSData dataWithContentsOfFile:path];
            if (data) {
                //读取成功
                resolve(data);
            }
            else {
                //读取失败
                reject([NSError new]);
            }
        });
    }];
}

- (void async)read3Files {
    @try {
        NSData *data1 = await [self readFileWithPath:@".../path1"];
        NSData *data2 = await [self readFileWithPath:@".../path2"];
        NSData *data3 = await [self readFileWithPath:@".../path3"];
        //全部读取完成
    }
    @catch(NSError *err) {
        //读取出错
    }
}
```
但事实是OC并不支持，那么如果靠自己写第三方实现，支持如上异步编程方式呢?

细想要完全如上是不可能的，用尽奇淫巧技也不可能以第三方库的形式让编译器支持如上新语法,因此稍微变换一下:
```Objective-C
- (void)read3Files {
    async(^{
        NSData *data1 = await( [self readFileWithPath:@".../path1"]);
        NSData *data2 = await( [self readFileWithPath:@".../path2"]);
        NSData *data3 = await( [self readFileWithPath:@".../path3"]);
    })
    .catch(^(error) {
        //出错处理
    })
    .finally(^{
        
    })
}
```
这样子就很有希望了, async/catch/fianlly链可以运用链式编程的思想实现，await只是一个普通c函数，async的参数也只是一个void (^)(void) 类型的block.因此基于完全可以实现的假设下，作如下约定:

###### (1).把传给async的代码块叫做异步块，表示块内的代码将支持await.

###### (2).await的含义仍然是等待异步操作的结果，等待成功后赋值给data。

###### (3).在某个await等待异步操作完成的过程中，如果异步操作出错了，便在此await处终止代码块的执行流，并将执行流将转向catch块，统一处理错误.

###### (4).不管异步块成功执行结束，还是中间出错，finally块总会执行，方便进行一些收尾的操作.

###### (5).Promise其实已经有很好的实现:PromiseKit. 但是还可以再抽象一下：什么是一个异步操作？

可以理解为:"操作的完成不阻塞当前流程的操作" 便是异步操作。 因此完全可以如下定义一个闭包类型`AsyncClosure`为异步操作
```Objective-C
typedef void (^AsyncClosure)(void (^resultCallback)(id value, id error));
```
`AsyncClosure`的参数`resultCallback`作为输出操作结果的渠道，至于`AsyncClosure`内部，想要实现读取文件也好，下载数据也好，只要以非阻塞方式进行，最后通过`resultCallback`回调出结果即可. 

也就是此情景下的Promise完全可以用`AsyncClosure`类型的闭包替代:
```Objective-C
- (AsyncClosure)readFileWithPath:(NSString *)path {
    return  ^(void (^resultCallback)(id value, id error)) {
        dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
            NSData *data =  [NSData dataWithContentsOfFile:path];
            resultCallback(data, [NSError new]);
        });
    };
}
```

#### 2.catch块引发的问题
先不管如何实现，假设上面提到的都是可以实现的，那么细细推敲便发现新的问题：catch机制将引发内存泄露.

正如上面对出错的约定：`在某个await等待异步操作完成的过程中，如果异步操作出错了，便在此await处终止代码块的执行流，并将执行流将转向catch块，统一处理错误.`，那么这时候的程序执行流便如下所示:

![](http://oem96wx6v.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202018-04-25%20%E4%B8%8B%E5%8D%884.49.40.png)

如上图所示,异步块首先分配了一个对象object,然后进行三个await操作，假设在第二个await的过程中，发生错误，那么程序执行流直接略过后面的所有代码，进入catch块处理错误。

看起来很美，实际上这时不管在MRC，还是ARC下，object都无法得到释放。在ARC下，大部分情况下编译器为object生成的释放代码都处于代码块末端：

![](http://oem96wx6v.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202018-04-25%20%E4%B8%8B%E5%8D%885.15.50.png)

如果执行流从中间断开了，那么这些释放代码永远执行不到，将造成“每逢出错必泄露”的局面。

#### 3.最终假设
为避免上一步提到的内存泄露问题，决定放弃catch块统一处理错误的机制，根本目的是确保异步块执行流不会从中间“异常断开”。改进的方式是:

(1)每次await将返回一个Result，Result包装了异步操作的实际结果值value,和错误对象error:
```Objective-C
@interface Result: NSObject
@property (nonatomic, strong, readonly) id  value;
@property (nonatomic, strong, readonly) id  error;
@end
```
(2)每一步await的错误具体要怎么处理由程序员决定，不再统一处理:
```Objective-C
async(^{
    Result *result1 = await( [self readFileWithPath:@".../path1"] );
    if (result1.error) {
      //第1步出错
    }
    NSData *data1 = result1.value;
    
   Result *result2 = await( [self readFileWithPath:@".../path2"] );
    if (result2.error) {
      //第2步出错
    }
    NSData *data2 = result2.value;
    
    
    Result *result3 = await( [self readFileWithPath:@".../path3"] );
    if (result3.error) {
      //第3步出错
    }
    NSData *data3 = result3.value;
})
.finally(^{

})
```
至此，整个假设已经没有什么逻辑上的硬伤，开始着手实现.

### 实现
```Objective-C
async(^{
    Result *result1 = await( [self readFileWithPath:@".../path1"] );
    Result *result2 = await( [self readFileWithPath:@".../path2"] );
    Result *result3 = await( [self readFileWithPath:@".../path3"] );
});
```
为满足async/await的定义. 异步块在执行的过程中要满足如下流程:

![](http://oem96wx6v.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202018-04-25%20%E4%B8%8B%E5%8D%886.29.29.png)


await在发起异步操作后，必须要暂停，如果不暂停，await将立刻返回，此时根本得不到异步操作结果。因此await以某种方式暂停，等待异步操作完成是必然.

只是该如何暂停?

##### (1)用信号量机制，await在发起异步操作后，等待信号量。异步操作结束并回调后，再激活信号量，即线程挂起机制。

如果当前是子线程，貌似没问题。但是这和直接dispatch_async(global_queue, ^{}),并在子线程进行同步操作没有区别，这种方式实现出来的await完全是无用功，因为搞半天不如直接dispatch_async子线程。

如果当前是主线程，这种方式是灾难性的,主线程挂起了就基本告别APP了。

因此线程挂起方式根本不可取。

##### (2)协程机制
协程的概念与原理资料颇多，总结来说，协程可以理解为可中断与恢复的函数。此处的中断与恢复并不是线程的挂起与恢复.假设当前函数A中断执行了，线程不会挂起，而是转而去执行函数B了，B函数也可能中断执行，然后又恢复执行函数A.就像cpu时间片轮转一样，一个线程“同时”执行着好几个函数，这种函数叫做“协程”。

此处如果运用协程机制，那么await中断而不挂起线程，异步操作后在以某种机制恢复await的执行，正是想要的效果.


#### 1.实现协程
根据前面协程的描述，实现协程的目的在于实现这样一个函数：

(1)函数执行过程中通过某种机制使函数中断执行。

(2)有某种机制恢复被中止的函数从第一步中断处继续执行.


为了实现这样的函数，需要清楚函数调用的基本原理.以arm32平台为例，在函执行过程中，需要在当前进程栈空间记录函数的执行信息，随着函数调用深度增加，栈空间向低地址扩展且最底端（栈顶）由寄存器sp记录。 每个函数有一片属于自己的记录区域，叫做栈桢，栈桢的起始地由桢指针寄存器fp记录，栈桢记录的主要信息包括：

(1)当前函数的返回地址: 从当前函数返回到上一级函数后，继续执行何处的指令. 该地址在上一级调用当前函数时会自动存入LR寄存器。

(2)当前函数保护的寄存器: 当前函数在执行过程中将会覆写大量寄存器，如果不保护, 返回到上一级函数后，数据就会混乱, 尤其需要保护上一级函数的桢指针fp.寄存器保护方式为将寄存器依次入栈，函数返回前又依次出栈.

(3)当前函数的局部变量: 分配局部变量的方式为下移栈顶指针，即伸展栈，比如定义了一个16字节数组数组的局部变量:
```
char arr[16];
```
只需sp = sp - 16,即可完成arr的分配.

(4)当前函数调用新函数所传的参数.在寄存器不够传参的情形下，剩余的参数将保存在栈空间，以供被调用函数访问。

比如如下调用过程:
```c
void bar (arg, ...) {
   //...
}

void foo(arg, ...) {
   bar(arg, arg,...);
   //...
}

int main() {
    foo(arg, arg ...);
    reutrn 0;
}
```
在执行到main内部的时候，栈空间布局如下:

![](http://oem96wx6v.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202018-04-25%20%E4%B8%8B%E5%8D%8810.25.40.png)

执行到foo的时候,栈空间布局:

![](http://oem96wx6v.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202018-04-25%20%E4%B8%8B%E5%8D%8810.36.20.png)

执行到bar的时候，栈空间继续伸展:

![](http://oem96wx6v.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202018-04-25%20%E4%B8%8B%E5%8D%8810.36.30.png)

反之，函数返回后，栈空间将向上收缩，收缩过程与如上的伸展过程完全相反.而当再次发生函数调用时，栈空间又会再次伸展,如此便是函数调用的基本过程，栈空间可以认为是函数活动记录的重要数据区.

##### 函数的中断与恢复
假设想要中断函数bar,那么必须记录足够的信息，将来才能恢复执行:

(1)此刻正执行的指令地址.

(2)此刻的寄存器环境.其中便包含了跟栈桢有关的寄存器fp,sp.

还有要想在将来恢复执行bar，就要保证中断后bar的栈桢不会被破坏,比如虽然保存了bar此刻的指令与寄存器信息，但是接着bar却返回了，或者bar的上一级foo返回了，那么整个栈会立刻收缩,bar的栈桢不复存在.

何时触发中断?


##### 切换回iO
##### 切换回iOS
