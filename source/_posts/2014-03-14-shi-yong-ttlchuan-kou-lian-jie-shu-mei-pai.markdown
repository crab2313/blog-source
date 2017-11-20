---
layout: post
title: "Linux上使用TTL串口连接树梅派"
---

由于实际情况，没有显示器，usb键盘一类的外设，也没有路由器，而且本机架`DHCP`服务器也很麻烦，所以我试图使用`UART`连接树梅派

所需工具:


1.  `FT232`模块的`USB`转`TTL`电平小板一块

2. 自配的杜邦线

3. 树梅派

4. `minicom`这个大家都懂的东西


  需要注意的是，树梅派的电平需要3.3V而不是5V。将树梅派的`GND`接板子的`GND`， `RXD`接板子的`TXD`， `TXD`接板子的`RXD`, 开机插入`USB`接口。我使用的是`Archlinux`，查看一下`dmesg`,发现板子已经自动识别，`/dev`下面多了一个`/dev/ttyUS0`, 现在运行`minicom -s`，将`Filename and paths`改为`/dev/ttyUSB0`, 将`Serial port setup`中的`Hardware control flow`关掉，退出。最后运行`minicom`，成功看到树梅派的登录界面。
