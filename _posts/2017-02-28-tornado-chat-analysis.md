---
layout: post
title: Tornado中的chatdemo分析
category: tech
---


[源码地址](https://github.com/tornadoweb/tornado/tree/master/demos/chat)

关于Tornado中[Future类](http://www.tornadoweb.org/en/stable/concurrent.html#tornado.concurrent.Future)。
> Placeholder for an asynchronous result.

***以下都是我的个人理解，理解不当之处请指出。***

代码主要集中在`chat.js`和`chatdemo.py`中。其中，`chat.js`是client端代码，`chatdemo.py`是server端代码。

#### chat.js

这里涉及到两种post请求，分别是/new和/update。/new是用来创建新消息，/update是更新用。

```javascript
$(document).ready(function() {
    if (!window.console) window.console = {};
    if (!window.console.log) window.console.log = function() {};

    $("#messageform").on("submit", function() {
        newMessage($(this));
        return false;
    });
    $("#messageform").on("keypress", function(e) {
        if (e.keyCode == 13) {
            newMessage($(this));
            return false;
        }
        return true;
    });
    $("#message").select();
    updater.poll();
});

var updater = {
    errorSleepTime: 500,
    cursor: null,

    poll: function() {
        var args = {"_xsrf": getCookie("_xsrf")};
        if (updater.cursor) args.cursor = updater.cursor;
        $.ajax({url: "/a/message/updates", type: "POST", dataType: "text",
                data: $.param(args), success: updater.onSuccess,
                error: updater.onError});
    },

    onSuccess: function(response) {
        try {
            updater.newMessages(eval("(" + response + ")"));
        } catch (e) {
            updater.onError();
            return;
        }
        updater.errorSleepTime = 500;
        window.setTimeout(updater.poll, 0);
    }
    //之后省略...
}

```

client端逻辑是：

1. 发送新消息时，调用`newMessage`函数，这个函数就是封装了下/new的post请求，顺便在页面上显示新消息，通过updater的`showMessage`方法实现。然后，**继续调用`poll`方法**。
2. `poll`方法是一个不断地轮询，向server发送/update的post请求，然后又作为执行成功的callback，所以就是看做一直轮询，直到接收到server返回的数据，再进入下一次轮询。

#### chatdemo.py

server端：

每有新client和server建立连接，就新建Future对象。waiters是由Future组成的集合，waiters数量等于active clients的数量，每个waiter都是一个Future。

1. Message_buffer类，实现了`new_message`方法和`wait_for_message`方法，具体作用下面会说。
2. 处理/new请求的handler。对于每个新消息分配一个uuid后，**调用`new_message`方法**，顺便把render出来的html string返回给client，以便显示。`new_message`方法给每个waiter塞入新消息。
3. 处理/update请求的handler。这里的方法被@tornado.gen.coroutine装饰成异步环境，原因是：返回更新信息是一个阻塞的过程，只有当**有新信息**时，才会继续之后的行为。所以**等待新信息到来这个行为是阻塞的**，如果不是异步环境，那么server在开一个进程的情况下只能处理一个client。这个函数在接收到/update请求之后，调用`wait_for_message`方法。通过`cursor`判断是否为新连接，新连接则新增一个waiter，旧连接则为其更新新消息（根据`id`把之后的消息塞入这个waiter)。*注意，这里waiter一直在增加，如果不去进行资源回收很容易耗尽内存，所以需要重写`tornado.web.RequestHandler`的`on_connection_close`方法，就是从waiters中移除当前waiter，把当前waiter清空。*


这应该算是HTTP long polling的一种写法。Tornado框架在其中起到的作用是提供异步环境，和异步结果的容器(Future)。也可能是刚接触的缘故，整体看下来这种解决方法还是有些复杂的。解决push updates需求还有websocket。[这里](http://stackoverflow.com/questions/10028770/in-what-situations-would-ajax-long-short-polling-be-preferred-over-html5-websock)可以看到详细的比较。下次我会试着放一份websocket实现chatdemo的对比。
