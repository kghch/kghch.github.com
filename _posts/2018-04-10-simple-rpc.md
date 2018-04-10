---
layout: post
title: 简单RPC
category: tech
---
{% include JB/setup %}


![sRPC主要部件图](https://i.loli.net/2018/04/10/5accbaabe99e4.jpg)


客户端侧代码：

```Java
public class ClientApp {

    public static void main(String[] args) {
        ServiceProxy.setZookeeperHost("127.0.0.1:2181");

        AddService addService = ServiceProxy.newServiceProxy(AddService.class);
        try {
            int c = addService.add(999, 888);
            System.out.println("计算结果是：" + c);
        } catch (Exception e){
            System.out.println(e);
        }

    }

}

```

服务端侧代码：

```Java
public class ServerApp {

    public static void main(String[] args) {

        RpcServer rpcServer = new RpcServerImpl(9999, 1, "127.0.0.1:2181");

        rpcServer.register(AddService.class.getName(), AddServiceImpl.class, "127.0.0.1:9999");

        rpcServer.start();

    }
}
```

流程（客户端发起）：

1. `AddService addService = ServiceProxy.newServiceProxy(AddService.class)`
2. 上述`newServiceProxy`会调用`Proxy.newProxyInstance(classload, interfaceList, handler)`
3. `addService.add(10, 20)`方法执行，此时会触发步骤2中handler的`invoke`方法。
4. `invoke`方法：通过`NettyClient`将步骤3中的`服务名`、`方法名`、`参数类型`、`参数`发送给`NettyServer`。
5. `NettyServer`收到消息：
	- 服务名，拿到 `clazz`
	- 方法名、参数类型， 拿到 `method`
6. 然后调用`method.invoke(clazz.newInstance, args)`，将结果写回给`NettyClient`。
7. `NettyClient`收到结果，输入，OVER.


