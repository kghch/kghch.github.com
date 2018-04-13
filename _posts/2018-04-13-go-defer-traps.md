---
layout: post
title: Go defer的陷阱
category: tech
---
{% include JB/setup %}

起因是，今晚公司的内部分享中讲到了go defer的问题，引起了很多人的争讨。用事实说话，我整理了下比较容易出困惑的点。

```Go
package main

import (
    "fmt"
)

func main() {
    res := test_A()
    fmt.Println(res)

    test_B()
    test_C()
    
    fmt.Println("Over.")
}


func test_A() []int{
    // 这里证明了：遇到defer时先压入变量的值到list中，return时再执行。这里变量的值是指针。
    s := []int {1,2,3}
    defer fmt.Println(s)
    s[0] = 4
    return s
}

func test_B() {
    // 这里证明了：压入的defer list其实是stack。
    for i:=0; i<5; i++ {
        defer fmt.Println("In test B: ",i)
    }
}

func test_C() {
    // 闭包只有在调用时，才执行内部表达式。 用defer时还是写的易读些吧，别弄成看笔试题一样。
    for j:=0; j<5; j++ {
        defer func() {
            fmt.Println("In test C: ", j)
            }()
    }
}

```
