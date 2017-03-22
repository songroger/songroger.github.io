---
layout: post
title:  "MIPS来跑Go应用"
date:   2017-03-20 18:15:14 +0000
categories: self words
tail: 
description: 
---
### 起因：

手上有个1代极路由，mips的cpu.
```
root@Hiwifi:/tmp/storage/sda/songroger/godir/mipsgo# cat /proc/cpuinfo
system type     : Atheros AR9330 rev 1
machine         : Turbo Wireless TW150V1 Board
processor       : 0
cpu model       : MIPS 24Kc V7.4
BogoMIPS        : 265.42
wait instruction    : yes
microsecond timers  : yes
tlb_entries     : 16
extra interrupt vector  : yes
hardware watchpoint : yes, count: 4, address/irw mask: [0x0000, 0x0100, 0x0418, 0x0088]
ASEs implemented    : mips16
shadow register sets    : 1
kscratch registers  : 0
core            : 0
VCED exceptions     : not available
VCEI exceptions     : not available
```
用了这么久才发现可以开openwrt，于是觉得不再折腾下实在是对不起它，想着搭一个文件服务器。
路由已有的环境是lua＋nginx，对于可以写点lua的童鞋来说，其实也够了。但非lua就只能懵圈了。于是开始搭python环境，安装过程真是一把心酸，放弃了...
openwrt有它自己的生态，但是跟完整的linux版本来比，要玩转跑各种语言还是很麻烦或者不支持。
最后选定了go，代码整理好了才发现编译的移到上面都跑不起来.本来也就这样结束了。
直到发现了[go-mips32](https://github.com/gomini/go-mips32)这个项目，
算是带来了点希望,而且最后还成功了，纪录下。

---
1. 下载 

    - `git clone https://github.com/gomini/go-mips32.git`
    - `cd go-mips32/src`

2. 配置编译参数

    - `export GOOS=linux` 
    - `export GOARCH=mips32`

3. 执行编译(mac下好像不行，也没搜到解决方案，最后老实换linux才编成功)
`./make.bash`
直到看到如下类似信息，说明成功了。
```
---
Installed Go for linux/mips32 in /tmp/mips/go-mips32
Installed commands in /tmp/mips/go-mips32/bin
```

4. 把编译好的这些文件复制到某个文件夹

    `sudo cp -R * /opt/mipsgo`

5. 把复制到的文件夹设置为go编译参数

    - `export GOROOT=/opt/mipsgo` 
    - `export PATH=/opt/mipsgo/bin:$PATH`

然后就可以去代码中`go build`你的mips应用了。

