---
layout: post
title:  "TCP和UDP的一些区别"
date:   2021-03-03 03:35:49 +0800
categories: network
---

TCP 一般使用`SOCK_STREAM`，UDP一般使用`SOCK_DGRAM`

TCP是一种面向连接的协议，连接会一直保持，双方会一直通话直到一方终止连接或者网络错误。

UDP是面向数据包的协议，你发送一个包得到一个恢复然后连接终止。

- 如果你发送多个包，TCP承诺按顺序投递它们。UDP不会，所以接收者如果介意顺序的话需要自己检查
 - 如果TCP包丢失，发送者会告知，但UDP不会
 - UDP的数据包有大小限制，可能是512bytes。TCP可以发送更大的包
 - TCP鲁棒性更好，有更多检查，UDP较少


在编写socket程序时，一般有如下代码
```c++
serv_sock = socket(PF_INET, SOCK_STREAM, 0);
```
AF = Address Family
PF = Protocol Family
这俩可认为时相同的
