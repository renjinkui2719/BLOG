boss发现，画思维导图真是提高效率，理清思路的绝佳方式，于是下载了iPhone版xMind.可是界面全是英文，用起来较吃力，遂交予我任务，将其设置为中文。我接过pad，发现：app菜单根本没有语言切换选项，系统语言地区切换，均不管用，最终得出结论，此app不支持中文。在经历500字的进展后，我决定将其汉化，单独给boss装一个汉化版本。

## 材料
越狱手机一部,itunes store 下载的xMind.ipa,[砸壳工具](https://github.com/chenhengjie123/iOS_resign_scripts), 有WiFi.
## 思路
![](http://oem96wx6v.bkt.clouddn.com/屏幕快照%202017-07-28%20下午5.18.00.png)
## 下载并解压
下载的ipa实际上是个压缩包，可以直接把扩展名改为.zip然后用解压工具解压，或者直接用uzip命令解压:
```shell
unzip xmin_cloud.ipa -d xmin_cloud_unziped
```
解压完成后，xmin_cloud_unziped文件夹应当如此: 

![](http://oem96wx6v.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-07-28%20%E4%B8%8B%E5%8D%886.04.24.png)

## 砸壳
1.在越狱手机上安装xMind，杀掉所有其他应用，单独打开xMind.

2.Terminal 执行 `ssh root@ip`. ip为手机ip，手机与电脑连接同一个wifi, 密码为越狱后打开SSH端口设置的密码，如果没改过，默认是"alpine".
