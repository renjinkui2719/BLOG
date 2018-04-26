
#### 在iOS平台实现新的异步解决方案async/await
async,await是ES7提出的异步解决方案，对比回调链和Promise.then链的传统异步编程模式，基于async,await可以完全以同步风格编写异步代码，程序逻辑更加清晰明了,维护与调试都很方便.

接触完毕后，深感如果在平时的iOS项目中也能像JS这般编写异步代码，那该多好。经过研究发现要实现这些特性其实并不是很困难，因此本文主旨便是描述async,await在iOS平台的实现过程，并给出了一个成果项目.

### 切换到JavaScript

#### 迭代器与生成器
要明白async/await的机制及运用,需从生成器与迭代器逐步说起.在ES6中，生成器是一个函数，和普通函数的区别是:

(1)生成器函数function关键字后多了个*:
```JS
function *numbers() {
}
```

(2)生成器函数内可以yield语法多次返回值:
```JS
function *numbers() {
    yield 1;
    yield 2;
    yield 3;
}
```

(3)直接调用生成器函数得到的是一个迭代器，通过迭代器的next方法控制生成器的执行:
```JS
let iterator = numbers();

let result = iterator.next();
```
每一次next调用将得到结果result, result对象包含两个属性:`value`和`done`.value表示此次迭代得到结果值,done表示是否迭代结束.
比如此例:
```JS
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

(4)通过迭代器向生成器内部传值
```JS
function *hello() {
  let age =  yield 'want age';
  let name = yield 'want name';
  console.log(`Hello, my age: ${age}, name:${name}`);
}

let iterator = hello();
```
创建迭代器并开始如下迭代过程:

(1)第1次迭代,生成器开始执行，到达第一个yield语句时，返回`value = want age, done = false`给迭代器, 并中断。
```JS
let result = iterator.next();
console.log(result);
//输出 => { value: 'want age', done: false }
```
(2)第2次迭代,给next传参28, 生成器从上次中断的地方恢复执行，并将28作为苏醒后yield的内部返回值赋给age; 然后生成器继续执行，再次遇到yield，返回`value = want name, done = false`给迭代器, 并中断。
```JS
result = iterator.next(28);
console.log(result);
//输出 => { value: 'want name', done: false }
```
(3)第3次迭代,给next传参'LiLei', 生成器从上次中断的地方恢复执行，并将'LiLei'作为苏醒后yield的内部返回值赋给name; 然后生成器继续执行，打印log:
```
Hello, my age: 28, name:LiLei
```
然后彻底结束生成器，返回`value = undefined, done = true`给迭代器。
```JS
result = iterator.next('LiLei');
console.log(result);
//输出 => { value: undefined, done: true }
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
