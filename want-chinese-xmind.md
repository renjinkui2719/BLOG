以很好用的思维导图工具Xmind为主例,其他几个App为辅例，试验与记录app字符串的修改与重签名，仅作学习目的，未行不轨之事。

## 材料
越狱手机一部,itunes store 下载的xMind.ipa,[砸壳工具](https://github.com/stefanesser/dumpdecrypted)(下载后编译出dumpdecrypted.dyld)，[重签名工具](https://github.com/chenhengjie123/iOS_resign_scripts).
## 思路
![](http://oem96wx6v.bkt.clouddn.com/屏幕快照%202017-07-28%20下午5.18.00.png)
## 下载并解压
下载的ipa实际上是个压缩包，可以直接把扩展名改为.zip然后用解压工具解压，或者直接用uzip命令解压:

```shell
unzip xmin_cloud.ipa -d xmin_cloud_unziped
```
解压完成后，xmind_cloud_unziped文件夹应当如此: 

![](http://oem96wx6v.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-07-28%20%E4%B8%8B%E5%8D%886.04.24.png)

进入`xmind_cloud_unziped/Payload/XMind Cloud.app`目录,删除可执行文件`XMind Cloud`,这个可执行文件是加密加壳过的，派不上用途，需要的是砸壳过的可执行文件。

## 砸壳
#### 1.在越狱手机上安装xMind，杀掉所有其他应用，单独打开xMind.

#### 2.Terminal 执行 `ssh root@ip`,登录到手机.

ip为手机ip, 密码为越狱后打开SSH端口设置的密码，如果没改过，默认是"alpine".

![](http://oem96wx6v.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-07-28%20%E4%B8%8B%E5%8D%886.23.06.png)

登录成功:

![](http://oem96wx6v.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-07-28%20%E4%B8%8B%E5%8D%886.23.58.png)

#### 3.找出xMind进程的沙盒目录Documents

(1）cyctipt方法得到Documents目录:

前提:已在手机上安装cycript命令。

执行 
`ps -e`命令,得到系统中所以进程，并且很容易找到Xmind进程信息:

![](http://oem96wx6v.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-07-28%20%E4%B8%8B%E5%8D%886.26.41.png)

得到Xmind的进程ID为:1429,可执行文件的路径为:

![](http://oem96wx6v.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-07-29%20%E4%B8%8A%E5%8D%888.58.38.png)

进程名就是可执行文件的名称"Xmind Cloud".

执行 
```
cycript -p 进程名
```
如果注入成功，即可在该进程下执行:
```
[[NSFileManager defaultManager] URLsForDirectory:NSDocumentDirectory inDomains:NSUserDomainMask][0]
```
来得到Documents目录.

因此运行命令 `cycript -p Xmind Cloud`, 实际上不会成功，因为进程名中间有空格，经过试验，就算 `cycript -p "Xmind Cloud"`, `cycript -p Xmind\ Cloud`.均不能成功，总是报找不到进程的错误, 而像QQ，进程名就叫QQ，中间没有任何空格，是可以成功的. 后经[高手](https://github.com/AloneMonkey)提点，`cycript -p 进程ID`也是可以的，因此此例应该执行:

```
cycript -p 1429
```

成功注入到进程：

![](http://oem96wx6v.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-07-29%20%E4%B8%8A%E5%8D%889.11.39.png)

获取Documents目录:

![](http://oem96wx6v.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-07-29%20%E4%B8%8A%E5%8D%889.12.07.png)

得到Documents目录为：

```
/var/mobile/Containers/Data/Application/7A2AAC43-55F5-41A2-B648-43B0C2453BF2/Documents/
```

(2)find命令方法得到Documents目录

如果没有安装cycript,用find命令也可以找出Documents目录:

进入上一步解压出的`xmind_cloud_unziped`目录，进入`Payload/XMind Cloud.app`目录，用xCode或其他能识别plist文件的工具打开Info.plist文件，看到该应用的Bundld ID是:"net.xmind.sunlight"

![](http://oem96wx6v.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-07-29%20%E4%B8%8A%E5%8D%889.38.39.png)

进入所有App沙盒目录的上级目录:

```
cd /var/mobile/Containers/Data/Application
```

看到所有APP沙盒目录列表:

![](http://oem96wx6v.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-07-29%20%E4%B8%8A%E5%8D%889.42.36.png)

肉眼是无法找出Xmind对应目录的.

find:

```
find . -name "net.xmind.sunlight"
```

得到：

![](http://oem96wx6v.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-07-29%20%E4%B8%8A%E5%8D%889.45.52.png)

`7A2AAC43-55F5-41A2-B648-43B0C2453BF2`就是该App的沙盒根目录,因此得到Documents目录应为:

```
/var/mobile/Containers/Data/Application/7A2AAC43-55F5-41A2-B648-43B0C2453BF2/Documents
```

#### 4.拷贝dumpdecrypted.dylib到Documents目录

![](http://oem96wx6v.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-07-29%20%E4%B8%8A%E5%8D%889.57.08.png)

进入Documents查看，拷贝成功

![](http://oem96wx6v.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-07-29%20%E4%B8%8A%E5%8D%889.58.29.png)

#### 5.砸壳

在Documents目录下执行

```
DYLD_INSERT_LIBRARIES=dumpdecrypted.dylib 可执行文件路径
```
即:

```
DYLD_INSERT_LIBRARIES=dumpdecrypted.dylib "/var/mobile/Containers/Bundle/Application/EE3623BB-E0C4-4A2D-9B6D-CD75EFF47BB0/XMind Cloud.app/XMind Cloud"
```
因可执行文件路径中含有空格，因此整个路径要用双引号括起来.其中DYLD_INSERT_LIBRARIES的作用，可参考[这篇文章](http://blog.csdn.net/zhangmiaoping23/article/details/50081915)

执行完如下过程，说明砸壳成功:

![](http://oem96wx6v.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-07-29%20%E4%B8%8A%E5%8D%8810.05.42.png)

XMind Cloud.decrypted即为砸壳后的可执行文件:

![](http://oem96wx6v.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-07-29%20%E4%B8%8A%E5%8D%8810.10.38.png)

将'XMind Cloud.decrypted'重命名为`XMind Cloud.decrypted`,否则下一步拷贝无法成功

```
mv "XMind Cloud.decrypted" XMindCloud.decrypted
```

![](http://oem96wx6v.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-07-29%20%E4%B8%8A%E5%8D%8810.18.25.png)

#### 6.拷贝砸壳文件

用scp命令将 XMindCloud.decrypted拷贝到解压包目录`xmind_cloud_unziped/Payload/XMind\ Cloud.app/`里，同时去掉.decrypted后缀:

```shell
scp root@192.168.2.15:/var/mobile/Containers/Data/Application/7A2AAC43-55F5-41A2-B648-43B0C2453BF2/Documents/XMindCloud.decrypted xmind_cloud_unziped/Payload/XMind\ Cloud.app/XMindCloud
```
编辑Info.plist,将可执行文件名由 "XMind Cloud"改为"XMindCloud":

![](http://oem96wx6v.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-07-29%20%E4%B8%8A%E5%8D%8810.34.48.png)

至此，砸壳过程完全结束，可以开始重签名，或者反汇编分析XMindCloud程序了

### 重签名
虽然目前什么都没改动，仅仅是砸了壳，但是可以立刻验证一下重签名后是否可以安装.

#### 1.证书与工具
本例用开发证书进行重签名. 将`mobileprovision`文件和`ios_resign_from_app_to_ipa`命令 拷贝到xmind_cloud_unziped同级目录下:

![](http://oem96wx6v.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-07-29%20%E4%B8%8A%E5%8D%8810.52.36.png)

到keyChain找到与该mobileprovision对应的证书，我的为:`iPhone Developer: JinKui Ren (MHR5XEJ2E4)`

![](http://oem96wx6v.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-07-29%20%E4%B8%8A%E5%8D%8810.48.33.png)

#### 2.按需修改Bundle ID，Display Name
如果希望设备上安装一个原来的App,和另一个重签名过的App,应当修改Info.plist中的Bundle Id值：

![](http://oem96wx6v.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-07-29%20%E4%B8%8A%E5%8D%8810.56.10.png)

否则因为证书冲突，需要卸载掉原来的App，才能安装相同ID的重签名App。

#### 2.重签名

```
 ./ios_resign_from_app_to_ipa 解压目录名 证书名 mobileprovision路径 重新名后的ipa名
```

![](http://oem96wx6v.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-07-29%20%E4%B8%8A%E5%8D%8811.00.13.png)

![](http://oem96wx6v.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-07-29%20%E4%B8%8A%E5%8D%8811.00.32.png)

成功生成ipa：

![](http://oem96wx6v.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-07-29%20%E4%B8%8A%E5%8D%8811.02.17.png)

#### 3.安装测试
iTools 或其他工具，直接拖入安装

![](http://oem96wx6v.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-07-29%20%E4%B8%8B%E5%8D%8812.52.19.png)

安装且运行成功，说明重签名没问题，接下来就是修改字符串了.


### 修改字符串
App的字符串要么在.strings资源文件,或者在Nib，或者在其他资源文件，或者写死在可执行程序.

#### 1.字符串在.strings文件
这种情况是最简短的，要替换字符串，直接编辑.strings文件即可.

strings文件实际上就是plist文件，假设文件全名为:Localizable.strings,直接改为:Localizable.strings.plist即可用Xcode直接编辑.

以另一个XX APP为例,我找到国家化文件夹en.lproj:

![](http://oem96wx6v.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-07-29%20%E4%B8%8A%E5%8D%8811.33.22.png)

将Localizable.strings改名为Localizable.strings.plist, 用Xcode打开:

![](http://oem96wx6v.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-07-29%20%E4%B8%8A%E5%8D%8811.36.20.png)

我希望他在英文界面下，"联系人"一项显示成中文，因此做如图修改.保存，文件名改回Localizable.strings,重新签名，安装打开后效果:

![](http://oem96wx6v.bkt.clouddn.com/ScreenShot_20170729_113919.png)

#### 2.字符串在.nib文件

编译过的nib文件无法直接编辑，研究ing...

#### 3.字符串在其他资源文件
"其他"二字范围甚广，如果app硬伤要自己开发一套字符串加载机制，是完全可行的，或者像本例子Xmind一样，核心功能是javascript写的,用的是cordova框架，体验完全原生级别，但是无法设置成中文界面:

![](http://oem96wx6v.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-07-29%20%E4%B8%8B%E5%8D%881.07.50.png)

核心逻辑在www文件夹下

![](http://oem96wx6v.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-07-29%20%E4%B8%8A%E5%8D%8811.52.36.png)

经过分析与查找，发现它的字符串写死在“main.4bbfd22776.js”文件的末端:

![](http://oem96wx6v.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-07-29%20%E4%B8%8B%E5%8D%8812.59.20.png)

逐一翻译替换掉:

![](http://oem96wx6v.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-07-29%20%E4%B8%8B%E5%8D%881.04.44.png)

重签名，安装,成功汉化:

![](http://oem96wx6v.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-07-29%20%E4%B8%8B%E5%8D%881.09.38.png)

#### 4.字符串写死在程序



本例用开发证书进行重签名. 将mobileprovision文件拷贝到xmind_cloud_unziped同级目录下，按需重命名一个简短一点的名字，我看的是[这篇教程](http://www.jianshu.com/p/f32e7c2c1628)，因此也像作者一样重命名为"embeded.mobileprovision"，key
本例用开发证书进行重签名. 将mobileprovision文件拷贝到xmind_cloud_unziped同级目录下，按需重命名一个简短一点的名字，我看的是[这篇教程](http://www.jianshu.com/p/f32e7c2c1628)，因此也像作者一样重命名为"embeded.mobileprovision"，与
本例用开发证书进行重签名. 将mobileprovision文件拷贝到xmind_cloud_unziped同级目录下，按需重命名一个简短一点的名字，我看的是[这篇教程](http://www.jianshu.com/p/f32e7c2c1628)，因此也像作者一样重命名为"embeded.mobileprovision"，
获取沙盒目录:Documentts
获取沙盒目录:

