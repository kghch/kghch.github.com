---
layout: post
title: 简单RPC
category: tech
---
{% include JB/setup %}


![主要部件图](https://www.draw.io/?lightbox=1&highlight=0000ff&edit=_blank&layers=1&nav=1#R7Vpdb9sgFP01ea0C2Pl4bNN2raZN1fKwdm%2FUpgmqYyJMmmS%2FfviD2AayOi5u02WpVJkLxnDO4d4Ldg9NFpsvHC%2Fn31hIoh7sh5seuuxBOPaH8n9q2OYGb%2BznhhmnYW4CpWFKf5PC2C%2BsKxqSpNZQMBYJuqwbAxbHJBA1G%2BacrevNnlhUf%2BoSz4hhmAY4Mq0%2FaSjmuXUEh6X9htDZXD0ZDMZ5zSMOnmecreLieT2InrJfXr3Aqq9ioskch2xdMaGrHppwxkR%2BtdhMSJRCq2DL77veU7sbNyexaHIDzG94wdGKqBFn4xJbhUU2G5K27%2FfQxXpOBZkucZDWriX50jYXi0iWgLwMcTLP2qaFom%2FCBdnsHR%2FYzVqKibAFEXwrmxQ3DAucCh0p2NYlKUg1mVcJQYURF0KY7XouwZAXBR52bJCBzZRwORsDITk9UYchEZw9kwmLGJeWmMWy5cUTjSLNhCM6i2UxkHDIjtFFChaVOjwvKhY0DNPHWHGvM%2FPEYjEtBgXeBXrPgjx0ALxnAP9jGeTY3y6W0cH4y0XYz34aCeXidMuEA%2FAB8mvo%2Byb6447Q9w30OZnRRByt8F3A3a%2BLfVd%2BB7UPDLy%2FEyG2LX3NJ9Q6BP6r4A%2F9bsAffq74h1DTAKgSrreAMzLAmUQ0Hfhx%2BoGOA6AN%2B658wtiaetCA3HG22Z6CUwAjDX1LBBx1hL7ahVTgT4j49fWGJceqfScph4b4yH83vQNgIJ4p%2FSwm69s4ETiWU%2F9nkd9ldx%2BBvLkDzJC%2FwbFE4DTyD68Ov2dx9J25GnOvc0mTgMkJFR7%2FFBgAUGNg1HC74yDJAeZ%2BJ8u%2FW6Y6nxB9Q%2F8W9G36d3HGAs1Q%2B4tJOMnS4nuOKRO3nYcAm5P2XKBkhkcDHBKH5%2BnJZxnlKmDIOfLtfQrcma%2BKD6puQ0WlSpYeatoioXFWqmEmx8FWPNDOcAXmMyIq%2BawJbQU73wKdsnESYUFf6qOw4Vk84Y7RdOXuYW7Q1xjJB1%2FcBCtnpVo%2Fupfyfa2jfMZGRxm7u1k3I9yMygcSvpfUihSGhhZaMq6O6I%2BFcTDQEip9N9yUcgj1zKw7ys3j5tZr3CTW%2BSKHJuVgjwP9v8r3UW4mf%2B7cektefQuv8CN53SV%2BKjvRA2rjtax3BDSn4JBY80zXHbHOl7J39JSPXVE%2B6G4tNzhJDlb8ZZeSVvgPIpwkNGghgaoAsmaJnJEw%2B83M1zRSnes7Ez%2F9s%2B1lBtnPqbYq77lPMExIbvC20myZNkj2j1dXsF97ny8v8g5by9Y84z%2FQUzVJNB0FJ0ts8j5SS%2FqBqS6BplrSj7p1TTr0UuaLheOle3BkdOsH5AC09R3D%2BgvP7jYVqMHBQbuoBKocl2q4r%2BjkzVFpFJAgsEWlx5GfOkJVoz7Rgk7T3sGHuhbNJ3hqe3io1jz9ywY9l3IUp5DxDqP%2F13Hp7ZUzdRXYEDzI01nFXlfXfvf3nhmVRainnVC9rg1ZLD%2BozJuXH62iqz8%3D)


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


