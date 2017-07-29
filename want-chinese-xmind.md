以很好用的思维导图工具Xmind为主例，研究与试验app的修改与重签名，仅作学习目的，未行任何不轨缺德之事。

## 材料
越狱手机一部,itunes store 下载的xMind.ipa,[砸壳工具](https://github.com/chenhengjie123/iOS_resign_scripts).
## 思路
![](http://oem96wx6v.bkt.clouddn.com/屏幕快照%202017-07-28%20下午5.18.00.png)
## 下载并解压
下载的ipa实际上是个压缩包，可以直接把扩展名改为.zip然后用解压工具解压，或者直接用uzip命令解压:
```shell
unzip xmin_cloud.ipa -d xmin_cloud_unziped
```
解压完成后，xmind_cloud_unziped文件夹应当如此: 

![](http://oem96wx6v.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-07-28%20%E4%B8%8B%E5%8D%886.04.24.png)

## 砸壳
1.在越狱手机上安装xMind，杀掉所有其他应用，单独打开xMind.

2.Terminal 执行 `ssh root@ip`,登录到手机. ip为手机ip, 密码为越狱后打开SSH端口设置的密码，如果没改过，默认是"alpine".
![](http://oem96wx6v.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-07-28%20%E4%B8%8B%E5%8D%886.23.06.png)

登录成功:

![](http://oem96wx6v.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-07-28%20%E4%B8%8B%E5%8D%886.23.58.png)

3.找出xMind进程的沙盒目录Documents

（1）cyctipt方法 :

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
即可在该进程下执行
```
[[NSFileManager defaultManager] URLsForDirectory:NSDocumentDirectory inDomains:NSUserDomainMask][0]
```
找出Documents目录.

因此运行 `cycript -p Xmind Cloud`, 实际上不会成功，因为进程名中间有空格，经过试验，就算 `cycript -p "Xmind Cloud"`, `cycript -p Xmind\ Cloud`.均不能成功，总是报找不到进程的错误, 而像QQ，进程名就叫QQ，中间没有任何空格，是可以成功的. 后经[高手](https://github.com/AloneMonkey)提点，`cycript -p 进程ID`也是可以的，因此在这种情况下，应该执行
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

(2)find命令方法
如果没有安装cycript,用find命令也可以找出Documents目录:

进入上一步解压出的`xmind_cloud_unziped`目录，进入`Payload/XMind Cloud.app`目录，用xCode打开Info.plist文件，看到该应用的Bundld ID是:""
获取沙盒目录:Documentts
获取沙盒目录:

