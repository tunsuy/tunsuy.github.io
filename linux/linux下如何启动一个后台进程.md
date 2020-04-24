## linux下如何启动一个后台进程

使用工具：startproc
为更方便地对自定义进程进行起停、查询等操作，我们可将自定义进程设置为守护进程，并利用service或者自己编写脚本等工具进行进程管理工作。

startproc启动脚本需要注意一下几点：

### 1、脚本工具必须是绝对路径
比如，需要执行的脚本test.py在/root/目录下，那么在用startproc时，不能这样：startproc test.py 
需要这样 startproc /root/test.py

### 2、脚本工具文件的开头必须注明执行环境
比如需要执行的脚本是python脚本，那么该脚本的开头必须加上这样一行：

```py
#!/usr/bin/python
```

### 3、执行时不能带语言工具
比如脚本是python语言的，正常执行可以是这样：python test.py
但是使用startproc时，不能这样：startproc python test.py
而是直接要这样：startproc test.py

注：startproc还可以指定用户执行：
startproc -u root xx

startproc可以启动任何的可执行文件，包括脚本、二进制等等