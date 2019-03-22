---
title: cjdns源码分析--使用SocketAddress来维护的EndpointsBySockaddr map机制中EndpointsBySockaddr session的建立过程
category:
  - cjdns
  - cjdns源码分析
tags:
  - cjdns
  - cjdns源码分析
  - C/C++
description: "这篇文章主要记录一下网内的点和peer之间的session map的维护"
date: 2017-11-22 16:11:01
---
## 简介 ##
源码位置：net/InterfaceController.c
首先，一个点要连入网内，肯定是依靠连接到一个已知点，将这个已知点作为自己的inbound。体现在配置中，就是
```
	"interfaces":{
		"UDPInterface":[
			{
				"bind":"0.0.0.0:26808",
				"connectTo":{
                	"192.168.2.82:29509":{
                	    "password":"GDY6nag7aI1ArqVK08Yr8hGRw2IDKB7",
                        "publicKey":"n2q6nuf0d5jsw7108qg1sxkc2hjp702hwlt2g3q379u9x5tsnr70.k"
                	}
				}
			}
		],
		"ETHInterface":[
			{
				"bind":"all",
				"beacon":0,
				"connectTo":{}
			}
		]
	},
```
比如这个192.168.2.82，就是inbound。
同样，网内的点也可能被别的点作为inbound，来主动连接。
无论是主动连接，还是被动连接，所有直接连接到我的点，都是我的peer。

## EndpointsBySockaddr的结构 ##
当前网内用来维护peer状态的，就是这个EndpointsBySockaddr map，首先看一下，这个map的结构定义
```
// ---------------- Map ----------------
#define Map_NAME EndpointsBySockaddr
#define Map_ENABLE_HANDLES
#define Map_KEY_TYPE struct Sockaddr*
#define Map_VALUE_TYPE struct Peer*
#define Map_USE_HASH
#define Map_USE_COMPARATOR
#include "util/Map.h"
static inline uint32_t Map_EndpointsBySockaddr_hash(struct Sockaddr** key)
{
    return Sockaddr_hash(*key);
}
static inline int Map_EndpointsBySockaddr_compare(struct Sockaddr** keyA, struct Sockaddr** keyB)
{
    return Sockaddr_compare(*keyA, *keyB);
}
// ---------------- EndMap ----------------
```
用Sockaddr*作为key，支持handles。所谓支持handles，其实就是增加了一种在map中寻找条目的方法。除了使用key来寻找对应条目，还可以使用handle来寻找条目。
handle是一个uint32_t类型的值，非常便于存储和传递。

## 建立过程 ##
依照点连入网内的过程，来分析session建立的过程。
### 连接inbound ###
调用过程：
client/Configurator.c:udpInterface
->  interface/UDPInterface_admin.c:beginConnection
->  net/InterfaceController.c:InterfaceController_bootstrapPeer
这里只分析InterfaceController.c文件中的内容。
```
int InterfaceController_bootstrapPeer(struct InterfaceController* ifc,
                                      int interfaceNumber,
                                      uint8_t* herPublicKey,
                                      const struct Sockaddr* lladdrParm,
                                      String* password,
                                      String* login,
                                      String* user,
                                      struct Allocator* alloc)
{
    ......
    //这里新建一个Peer
    struct Peer* ep = Allocator_calloc(epAlloc, sizeof(struct Peer), 1);
    ep->addrAlloc = Allocator_child(epAlloc);
    //作为key的Sockaddr
    struct Sockaddr* lladdr = Sockaddr_clone(lladdrParm, ep->addrAlloc);
    //加入到map当中
    int index = Map_EndpointsBySockaddr_put(&lladdr, &ep, &ici->peerMap);
    Assert_true(index >= 0);
    ep->alloc = epAlloc;
    //handle字段就是map中的handler
    ep->handle = ici->peerMap.handles[index];
    ......
    //向这个peer（也就是inbound）发送ping
    sendPing(ep);

    return 0;
}
```
### inbound收到这个ping ###
```
static Iface_DEFUN handleIncomingFromWire(struct Message* msg, struct Iface* addrIf)
{
    ......
	//拿到Sockaddr
    struct Sockaddr* lladdr = (struct Sockaddr*) msg->bytes;
    ......
    epIndex = Map_EndpointsBySockaddr_indexForKey(&lladdr, &ici->peerMap);
    ......
    if (epIndex == -1) {
        return handleUnexpectedIncoming(msg, ici);
    }
}
```
因为是第一个包，所以会进入到handleUnexpectedIncoming
```
static Iface_DEFUN handleUnexpectedIncoming(struct Message* msg,
                                            struct InterfaceController_Iface_pvt* ici)
{
    ......
    //拿到Socketaddr
    struct Sockaddr* lladdr = (struct Sockaddr*) msg->bytes;
    //新建一个peer
    ......
    struct Peer* ep = Allocator_calloc(epAlloc, sizeof(struct Peer), 1);
	......
    //加入到map
    Assert_true(Map_EndpointsBySockaddr_indexForKey(&lladdr, &ici->peerMap) == -1);
    int index = Map_EndpointsBySockaddr_put(&lladdr, &ep, &ici->peerMap);
    Assert_true(index >= 0);
    ep->handle = ici->peerMap.handles[index];
    ......
}
```


