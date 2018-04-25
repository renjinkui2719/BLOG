
#### 在iOS平台实现新的异步解决方案async/await
async,await是ES7提出的异步解决方案，对比回调链和Promise.then链的传统异步编程模式，基于async,await可以完全以同步风格编写异步代码，程序逻辑更加清晰明了,维护与调试都很方便.

接触完毕后，深感如果在平时的iOS项目中也能像JS这般编写异步代码，那该多好。经过研究发现要实现这些特性其实并不是很困难，因此本文主旨便是描述async,await在iOS平台的实现过程，并给出了一个成果项目.

##### 切换到JavaScript
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


##### 切换回iOS
