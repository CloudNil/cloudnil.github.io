---
layout: post
title:  Go语言defer的使用
categories: [Language]
date: 2014-05-19 10:58:30 +0800
keywords: [Go,defer]
---

>Go语言的defer是个非常巧妙的东西，使用过程中经常给我一种很高大上的感觉。

Go语言中没有类似于java中的try…catch…finally的语句块，但是却有一个非常优雅的defer。

defer关键字用来标记最后执行的Go语句，一般用在资源释放、关闭连接等操作，会在函数关闭前调用。

多个defer的定义与执行类似于栈的操作：先进后出，最先定义的最后执行。

在defer的使用中，碰到过许多坑，尤其是在defer与匿名函数搭配使用的时候，本文就来讲讲这些坑。

请先看下边几段代码，然后判断一下各自输出内容：

```golang
#示例代码一：
func funcA() int {
    x := 5
    defer func() {
        x += 1
    }()
    return x
}
#示例代码二：
func funcB() (x int) {
    defer func() {
        x += 1
    }()
    return 5
}
<!--more-->
#示例代码三：
func funcC() (y int) {
    x := 5
    defer func() {
        x += 1
    }()
    return x
}
 
#示例代码四：
func funcD() (x int) {
    defer func(x int) {
        x += 1
    }(x)
    return 5
}
```

将代码运行一下试试看，相信有不少读者都会发现自己给出的答案是错误的。

解析这几段代码，主要需要理解清楚以下几点知识：

1、return语句的处理过程

return xxx 语句并不是一条原子指令，其在执行的时候会进行语句分解成 返回变量=xxx return，最后执行return

2、defer语句执行时机

上文说过，defer语句是在函数关闭的时候调用，确切的说是在执行return语句的时候调用，注意，是return 不是return xxx

3、函数参数的传递方式

Go语言中普通的函数参数的传递方式是值传递，即新辟内存拷贝变量值，不包括slice和map，这两种类型是引用传递

4、变量赋值的传递方式

Go语言变量的赋值跟函数参数类似，也是值拷贝，不包括slice和map，这两种类型是内存引用

按照以上原则，解析代码：

```golang
#示例代码一：
func funcA() int {
    x := 5
    temp=x      #temp变量表示未显示声明的return变量
    func() {
        x += 1
    }()
    return
}
```

返回temp的值，在将x赋值给temp后，temp未发生改变，最终返回值为5。

```golang
#示例代码二：
func funcB() (x int) {
    x = 5
    func() {
        x += 1
    }()
    return
}
```

返回x的值，先对其复制5，接着函数中改变为6，最终返回值为6。

```golang
#示例代码三：
func funcC() (y int) {
    x := 5
    y = x       #这里是值拷贝
    func() {
        x += 1
    }()
    return
}
```

返回y的值，在将x赋值给y后，y未发生改变，最终返回值为5。

```golang
#示例代码四：
func funcD() (x int) {
    x := 5
    func(x int) { #这里是值拷贝
        x += 1
    }(x)
    return
}
```

返回x的值，传递x到匿名函数中执行的时候，传递的是x的拷贝，其内部修改不影响外部x的值，最终返回值为5。

可以看到，defer语句都插到了分解后的return语句前执行，结合赋值及参数值传递方式，可以很清晰的理解最后的结果。

>版权声明：允许转载，请注明原文出处：http://cloudnil.com/2014/05/19/defer-in-go/。