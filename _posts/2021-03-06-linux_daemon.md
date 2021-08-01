---
layout: post
title:  "linux 守护进程"
date:   2021-03-06 22:35:49 +0800
categories: linux
---

正常在terminal启动一个程序，在terminal关闭后，进程也随之退出

这样启动，称之为前台任务，独占了terminal窗口
能够在后台一直运行的，称之为`守护进程daemon`

```shell
xx.py &
```
只要在命令的尾部加上符号`&`，进程就会变成后台任务，如果想让正在运行的前台任务变为后台任务，可以先按`ctrl+z`, 然后执行`bg`命令（让最近一个暂停的后台任务继续执行）

后台任务有两个特点：
- 继承当前session的stdout和stder，因此后台任务的输出依然会同步的在命令行显示
- 不再继承stdin，如果试图读取标准输入，就会暂停

用户退出session后，系统向该sesion发出`SIGHUP`信号，session将该信号发给所有子进程
所以前台任务会随着session的退出而退出，后台任务是否会退出呢？
这取决于shell的`huponexit`参数
```shell
shopt | grep huponexit
```
一般默认关闭，因此session退出的时候不会给后台任务发sighup信号。

但当huponexit参数打开的时候，这个后台进程就可能退出了

`nohup`命令解决了这些事情
- 阻止sighup信号发送到这个进程
- 关闭标准输入
- 重定向标准输出和标准错误到文件nohup.out
```shell
nohup xx.py &
```

另一种思路是使用screen和tmux来复用terminal

终极方式设置为systemd自启动
[Linux 中设置进程通过 systemctl 启动](https://blog.csdn.net/kikajack/article/details/80508564)

