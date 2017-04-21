---
title: 谈谈linux下Bash中的函数返回值
date: 2016-05-25 19:28:34
tags: [shell,linux]
categories: linux
keywords: [linux,shell,bash,函数返回值]
description:
---

Bash函数与大多数编程语言中的函数不同，不允许你向调用者返回值。当bash函数结束时，返回值为其状态：成功为零，失败为非零。要返回值，可以使用结果设置一个全局变量，或者使用命令替换，也可以传入一个变量的名称作为结果变量。下面的例子描述了这些不同的机制。

# 一、 全局变量法
从bash函数返回值的最简单的方法是将全局变量设置为结果。由于bash中的所有变量默认为全局变量（不管是在哪个块中定义的）  
示例如下：

<!-- more -->

```sh
lockdir="somedir"
retval=-1
testlock(){
    if mkdir "$lockdir"
    then # directory did not exist, but was created successfully
         echo >&2 "successfully acquired lock: $lockdir"
         retval=0
    else
         echo >&2 "cannot acquire lock, giving up on $lockdir"
         retval=1
    fi
}

testlock
if [ "$retval" == 0 ]
then
     echo "directory not created"
else
     echo "directory already created"
fi
```
缺点：  
我们都知道，使用全局变量，特别是在大型程序中，可能导致难以发现错误

# 二、 echo法
更好的方法是在函数中使用局部变量。那么问题就是如何把结果传给调用者。一种机制是使用命令替换：
```sh
lockdir="somedir"
testlock(){
    retval=""
    if mkdir "$lockdir"
    then # directory did not exist, but was created successfully
         echo >&2 "successfully acquired lock: $lockdir"
         retval="true"
    else
         echo >&2 "cannot acquire lock, giving up on $lockdir"
         retval="false"
    fi
    echo "$retval"
}

retval=$( testlock )  #也可以 `testlock`
if [ "$retval" == "true" ]
then
     echo "directory not created"
else
     echo "directory already created"
fi
```

# 三、 函数参数法
返回值的另一种方法是编写函数，以便它接受变量名作为其命令行的一部分，然后将该变量设置为该函数的结果
```sh
function myfunc()
{
    local  __resultvar=$1
    local  myresult='some value'
    eval $__resultvar="'$myresult'"
}

myfunc result
echo $result
```
这里我们必须使用eval来实际设置。 eval语句基本上告诉bash将该行解释为两次，上面的第一个解释会导致字符串result ='some value'，然后再解释一次，最后设置调用者的变量。

以下两种方式等效：  
方式二
```sh
function myfunc()
{
    local  __resultvar=$1
    eval $__resultvar="some value"
}

myfunc result
echo $result
```
方式三
```sh
function myfunc()
{
    eval $1="'some value'"
}

myfunc result
echo $result
```

# 四、 返回状态法
虽然bash有一个返回语句`return`，但你可以指定的唯一的事情是函数的状态，它是一个 `数值`，如在exit语句中指定的值。 状态值存储在$？变量中。如果函数不包含 `return` 语句，则其状态将基于函数中执行的最后一条语句的状态设置。
```sh
lockdir="somedir"
testlock(){
    if mkdir "$lockdir"
    then # directory did not exist, but was created successfully
         echo >&2 "successfully acquired lock: $lockdir"
         retval=0
    else
         echo >&2 "cannot acquire lock, giving up on $lockdir"
         retval=1
    fi
    return "$retval"
}

testlock
retval=$?
if [ "$retval" == 0 ]
then
     echo "directory not created"
else
     echo "directory already created"
fi
```

