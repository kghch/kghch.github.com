---
layout: post
title: 数据结构
categoAry: tech
---

这里记录一些常用的高级数据结构，目的是提升自身使用姿势，做查漏补缺。

# 数据结构

### 1. cuckoo hash

为了解决hash冲突问题。

常规的解决hash冲突的方法是**拉链法**，有冲突的就接在链表后。比如java的hashMap就是用数组+链表 实现的

思路：多哈希函数，减少hash冲突概率。

适用于：读多写少的场景。（因为布谷鸟hash查两次必出结果，但拉链hash查多少次取决于拉链长度）

`[]*Node`  + 多个hash方法（初始是2）

- **插入key**：计算 hash1(key), hash2(key) ，任一位置为空即可插入；若都不为空，则随机占领一个位置，把原key’做剔除，key’按一样步骤选择自己的新位置。如果key’选新位置时仍然被占满，则继续剔除。   如果剔除次数超过阈值，则认为数组太满了，需要 添加新的hash函数+扩容+做rehash
- **读取key：**计算 hash1(key), hash2(key) if arr[idx]==key 则返回
- **删除key**：先读取，再清除

### 2. bloom filter

为了查询海量数据中是否存在某数据。 

思路：海量数据就意味着无法在内存中存放下所有数据。做数据压缩，把数据通过hash映射成一些比特位. 

`[]byte` + 多个hash方法（减少冲突）

不支持删除

### 3. cuckoo filter

解决bloom filter 不支持删除问题

`[]string` + 2个hash方法 （string用于存储指纹）

思路：类似cuckoo hash，有hash1 & hash2，若任一不为空，则插入；若都不空，随机选择一个位置将原key’踢出。 这里不同于cuckoo hash的问题是，这里bitmap存储的是哈希后的值，而非原key’ ，那怎么知道要踢出的原key’是什么呢 

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/59f856b0-4ba5-45f2-acf9-9bb6d83424a0/Untitled.png)

x = “axby......mmm”  

p2 = hash(x) ^ hash(fp)

p1 = p2 ^hash(fp) = hash(x)

所以对偶位置永远是 当前位置 ^ hash(fp)

常见的指纹方法：crc32, md5, sha1

### 4.skip list

[https://leetcode-cn.com/problems/design-skiplist/](https://leetcode-cn.com/problems/design-skiplist/) 

### 5. sync/pool

[这篇文章](https://medium.com/swlh/go-the-idea-behind-sync-pool-32da5089df72#id_token=eyJhbGciOiJSUzI1NiIsImtpZCI6IjU4YjQyOTY2MmRiMDc4NmYyZWZlZmUxM2MxZWIxMmEyOGRjNDQyZDAiLCJ0eXAiOiJKV1QifQ.eyJpc3MiOiJodHRwczovL2FjY291bnRzLmdvb2dsZS5jb20iLCJuYmYiOjE2NDg2OTM2ODQsImF1ZCI6IjIxNjI5NjAzNTgzNC1rMWs2cWUwNjBzMnRwMmEyamFtNGxqZGNtczAwc3R0Zy5hcHBzLmdvb2dsZXVzZXJjb250ZW50LmNvbSIsInN1YiI6IjExODE5MTgwMzMzMDE4NjA5ODgwMyIsImVtYWlsIjoid2FuZ2hhbjAzMDdAZ21haWwuY29tIiwiZW1haWxfdmVyaWZpZWQiOnRydWUsImF6cCI6IjIxNjI5NjAzNTgzNC1rMWs2cWUwNjBzMnRwMmEyamFtNGxqZGNtczAwc3R0Zy5hcHBzLmdvb2dsZXVzZXJjb250ZW50LmNvbSIsIm5hbWUiOiLnjovmmZciLCJwaWN0dXJlIjoiaHR0cHM6Ly9saDMuZ29vZ2xldXNlcmNvbnRlbnQuY29tL2EvQUFUWEFKem5IVjVoSnd0LW85ekhNT3B3SktlQ2FFaHE1UnhDT2EwUklsM289czk2LWMiLCJnaXZlbl9uYW1lIjoi5pmXIiwiZmFtaWx5X25hbWUiOiLnjosiLCJpYXQiOjE2NDg2OTM5ODQsImV4cCI6MTY0ODY5NzU4NCwianRpIjoiNmM4MGU3YWY3MTExMWY4ZjBkZmU1NDk4YTIzODQ3MDRmY2QzMjZmOCJ9.CXlYYHtxIkiPN2ZIWe_fKMj9Nv41RjY2EY6Gy96TRypztzRUlqcI8YOIal2gW-mvdmnM5MhqxYVM6MN28nXMhjHAF-5vEoffbwlxHmm1vs6LbtCz4ifIIP8M50RzA-G8dTxdAMhOok7UaqiJKwcqLqsbV8T7eYJmgSn2Z9XM3UnoQpkGUYzVFq1w8olOepWLQtL7R8jJupV75WnEj1NFgMDSqD-yQX5Cl9BTs9i5TvKwBtBEmD4Xxiezuk86TQqn77GIf71du3GfG3FvWom5Qym5fEE9hK7zUsbM5_GVyCOKEG0WsGpuG7oe1cV8EkwuE7L1KsNcBbz_-NFzZMMmyQ)讲的很好

是什么？对象缓存池

能解决什么问题？

Pool's purpose is to cache allocated but unused items for later reuse, relieving pressure on the garbage collector.

频繁新建、销毁对象，会造成频繁GC；而sync.Pool可以缓存这些暂时不用的对象，下次用时直接拿，而无需内存分配

怎么用？New , Put, Get

Get 存在时则直接返回，不存在则会调用New创建一个

Put

是否线程安全？ 是

优缺点：减少内存分配， 但单次put/get 时间更高。

实现原理：

注册runtime_registerPoolCleanup到runtime上，GC触发时，会执行自定义策略。分为local pool & victim cache.（相当于二次存活机会）

GC触发时，会Remove victim cache中的对象，并把local pool中的对象放到victim cache中；

put时，会把对象放到local pool中

get时，会把victime cache中的对象放到local pool中

保证线程安全：1.12 加mutex → 1.13 use lock-free structures ([https://github.com/golang/go/commit/d5fd2dd6a17a816b7dfd99d4df70a85f1bf0de31#diff-491b0013c82345bf6cfa937bd78b690d](https://github.com/golang/go/commit/d5fd2dd6a17a816b7dfd99d4df70a85f1bf0de31#diff-491b0013c82345bf6cfa937bd78b690d)) 

best practice : 

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/ed3f8f22-a3a3-45c9-9b8d-d4f5ab7d6e83/Untitled.png)

fmt package 、 gin  都使用到了sync.Pool

- Gin：先调用 Get 取出来缓存的对象，然后会做一些 reset 操作，再执行 `handleHTTPRequest`
，最后再 Put 回 Pool。

### 6.singleflight

作用：singleflight可抑制对下游的多次并发请求

场景：当多个请求都来请求一个key，但cache miss，则同时打到后端db，造成服务压力，此时应使用singleflight

抑制周期：一个方法执行周期内，抑制所有并发。但一次执行完成后则不再抑制

使用姿势：1. var g [singleflight.Group](http://singleflight.Group)  2. g.Do(”key”, func) ，有返回值

实现原理：mutex+map[key]val+ waitGroup

核心代码：

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/3967edae-8783-4f21-859b-4031c10056c2/Untitled.png)

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/f38125f0-d110-4327-a261-51ceaadeab29/Untitled.png)

### 7.sync/once

场景：和singleflight类似

抑制周期：声明一个once对象，则此对象下永久抑制

使用姿势：1. var once sync.Once 2. [once.Do](http://once.Do)(f func()) ，无返回值

实现原理：mutex+一个标记位标记是否执行过

### 8.slice

数据结构：1.指向底层数组的指针 2.容量 3.当前长度

9.channel

10.context