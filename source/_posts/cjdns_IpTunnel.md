---
title: cjdns源码分析--IpTunnel
category:
  - cjdns
   - cjdns源码分析 
tags:
  - cjdns
  - cjdns源码分析
  - C/C++
description: “IpTunnel中主要是和离岸点相关的代码。结合最近的需求，分析一下源码，设计一下需求的实现方案”
date: 2017-07-25 09:48:00
---
## 源码分析 ##
### 结构体 ###
首先看一下IpTunnel的结构体定义
```
struct IpTunnel
{
    /** The interface used to send and receive messages to the TUN device. */
    struct Iface tunInterface;

    /**
     * The interface used to send and receive messages to other nodes.
     * All messages sent on this interface shall be preceeded with the RouterHeader and DataHeader.
     */
    struct Iface nodeInterface;

    /**
     * The list of registered connections, do not modify manually.
     * Will be reorganized from time to time so pointers are ephemeral.
     */
    struct {
        uint32_t count;
        struct IpTunnel_Connection* connections;
    } connectionList;
};
```
两个Iface，一个负责跟tun设备沟通，一个负责跟其他node沟通
一个connectionList，保存现有的所有connection。

然后看一下IpTunnel_Connection的结构体定义
```
struct IpTunnel_Connection
{
    /** The header for routing to this node. */
    struct RouteHeader routeHeader;

    /** The IPv6 address used for this connection or all zeros if none was assigned. */
    uint8_t connectionIp6[16];

    /** The IPv4 address used for this connection or all zeros if none was assigned. */
    uint8_t connectionIp4[4];

    /** The IPv6 netmask/prefix length, in bits. Defaults to 128 if none was assigned. */
    uint8_t connectionIp6Prefix;

    /** The IPv6 prefix length in, in bits, defining netmask. 0xff if not used. */
    uint8_t connectionIp6Alloc;

    /** The IPv4 address prefix length, in bits. Defaults to 32 if none was assigned. */
    uint8_t connectionIp4Prefix;

    /** The IPv6 prefix length in, in bits, defining netmask. 0xff if not used. */
    uint8_t connectionIp4Alloc;

    /** non-zero if the connection was made using IpTunnel_connectTo(). */
    int isOutgoing : 1;

    /** The number of the connection so it can be identified when removing. */
    int number : 31;

    bool reachable;
};
```
 这个结构体记录的是当前点与其他node的连接信息。对于普通的点来说，这里保存着与离岸点的连接信息。每个在conf中定义的离岸点（outgoingConnections），对应着这里的一个IpTunnel_Connection对象，且该对象的isOutgoing字段为1，表明这是一个与离岸点的连接。对于离岸点来说，这里保存着所有连接到该离岸点的连接信息。每个在conf中定义的允许连接的点（allowedConnections），对应着这里的一个IpTunnel_Connection对象，且该对向的isOutgoing字段为0，表明这不是一个向离岸点的连接。
 
结构体中还有六个与ip相关的字段，3个ip4，3个ipv6，根据需要，使用其中3个。这些ip字段，记录着该连接中，非离岸点的ip信息。

结构体中的routeHeader字段，记录着去往对方的路由信息。

### conf文件起到的作用 ###
有两种途径来新建一个IpTunnel_Connection。
1. 作为普通点，建立一个与离岸点的IpTunnel_Connection
对应着conf文件中的outgoingConnections配置，调用到IpTunnel_admin中的connectTo方法，继而调用到IpTunnel中的IpTunnel_connectTo方法。
```
			"outgoingConnections":[
				"wfzyzrc0q4g83y0dgzxx1l862u0lscucj75yw9q1ymbltzwh2fq0.k"
			]
```
2. 作为离岸点，为每个允许连接过来的普通点建立一个IpTunnel_Connection，对应着conf文件中的allowedConnections配置，调用到IpTunnel_admin中的allowConnection方法，继而调用到IpTunnel中的IpTunnel_allowConnection方法。
```
"allowedConnections": [
        {
            "ip4Address": "192.168.254.2",
            "ip4Prefix": 0,
            "ip4Alloc": 32,
            "publicKey": "uhmhts49tdm1ryb3q0pw95291uwt4xgvyk5s84vt2z2mnv3zp230.k"
        },
        {
            "ip4Address": "192.168.254.3",
            "ip4Prefix": 0,
            "ip4Alloc": 32,
            "publicKey": "9c3x7hp181dv91tfkbngyhhu2uc3xhxuuh539l3g0gdjgjg1bs10.k"
        }
]
```

### 普通点建立一个离岸点的IpTunnel_Connection ###
#### conf文件配置 ####
```
			"outgoingConnections":[
				"wfzyzrc0q4g83y0dgzxx1l862u0lscucj75yw9q1ymbltzwh2fq0.k"
			]
```
#### 读取配置文件 ####
Configurator.c
```
    List* outgoing = Dict_getListC(ifaceConf, "outgoingConnections");
    if (outgoing) {
        String* s;
        for (int i = 0; (s = List_getString(outgoing, i)) != NULL; i++) {
            Log_debug(ctx->logger, "Initiating IpTunnel connection to [%s]", s->bytes);
            Dict requestDict =
                Dict_CONST(String_CONST("publicKeyOfNodeToConnectTo"), String_OBJ(s), NULL);
            Dict* resp = NULL;
            rpcCall0(String_CONST("IpTunnel_connectTo"), &requestDict, ctx, tempAlloc, &resp, true);
            int64_t* num = Dict_getIntC(resp, "connection");
            Log_debug(ctx->logger,"noti outgoing callback %d",(int)*num);
        }
    }
```
读取配置文件中outgoingConnections中的离岸点pubkey信息，rpcall调用IpTunnel_connectTo方法。

#### IpTunnel_connectTo方法 ####
IpTunnel_admin.c
该方法在文件中注册为connectTo函数。
```
static void connectTo(Dict* args, void* vcontext, String* txid, struct Allocator* requestAlloc)
{
    struct Context* context = vcontext;
    String* publicKeyOfNodeToConnectTo =
        Dict_getStringC(args, "publicKeyOfNodeToConnectTo");

    uint8_t pubKey[32];
    uint8_t ip6[16];
    int ret;
    if ((ret = Key_parse(publicKeyOfNodeToConnectTo, pubKey, ip6)) != 0) {
        sendError(Key_parse_strerror(ret), txid, context->admin);
        return;
    }
    int conn = IpTunnel_connectTo(pubKey, context->ipTun);
    sendResponse(conn, txid, context->admin);
}
```
解析离岸点的pubkey，验证其合法性，调用IpTunnel_connectTo方法，建立IpTunnel_Connection。

#### IpTunnel_connectTo方法 ####
IpTunnel.c
```
int IpTunnel_connectTo(uint8_t publicKeyOfNodeToConnectTo[32], struct IpTunnel* tunnel)
{
    struct IpTunnel_pvt* context = Identity_check((struct IpTunnel_pvt*)tunnel);
    Log_debug(context->logger, "noti outgoing call by IpTunnel_connectTo");
    struct IpTunnel_Connection* conn = newConnection(true, context);
    Bits_memcpy(conn->routeHeader.publicKey, publicKeyOfNodeToConnectTo, 32);
    AddressCalc_addressForPublicKey(conn->routeHeader.ip6, publicKeyOfNodeToConnectTo);

    if (Defined(Log_DEBUG)) {
        uint8_t addr[40];
        AddrTools_printIp(addr, conn->routeHeader.ip6);
        Log_debug(context->logger, "Trying to connect to [%s]", addr);
    }

    requestAddresses(conn, context);
    return conn->number;
}
```
这里主要有两个操作：
1. 调用newConnection(true, context)创建一个IpTunnel_Connection类型的conn，并将conf配置中的离岸点pubkey赋值给conn->routeHeader.publicKey。
2. 调用requestAddresses方法，向离岸点申请ipv4地址。

#### 创建一个IpTunnel_Connection类型的conn ####
调用newConnection方法。
IpTunnel.c
```
static struct IpTunnel_Connection* newConnection(bool isOutgoing, struct IpTunnel_pvt* context)
{
    if (context->pub.connectionList.count == context->connectionCapacity) {
        uint32_t newSize = (context->connectionCapacity + 4) * sizeof(struct IpTunnel_Connection);
        context->pub.connectionList.connections =
            Allocator_realloc(context->allocator, context->pub.connectionList.connections, newSize);
        context->connectionCapacity += 4;
    }
    struct IpTunnel_Connection* conn =
        &context->pub.connectionList.connections[context->pub.connectionList.count];

    // If it's an incoming connection, it must be lower on the list than any outgoing connections.
    if (!isOutgoing) {
        for (int i = (int)context->pub.connectionList.count - 1; i >= 0; i--) {
            if (!context->pub.connectionList.connections[i].isOutgoing
                && conn != &context->pub.connectionList.connections[i + 1])
            {
                Bits_memcpy(conn,
                                 &context->pub.connectionList.connections[i + 1],
                                 sizeof(struct IpTunnel_Connection));
                conn = &context->pub.connectionList.connections[i + 1];
            }
        }
    }

    context->pub.connectionList.count++;

    Bits_memset(conn, 0, sizeof(struct IpTunnel_Connection));
    conn->number = context->nextConnectionNumber++;
    conn->isOutgoing = isOutgoing;
    conn->reachable = false;

    // if there are 2 billion calls, die.
    Assert_true(context->nextConnectionNumber < (UINT32_MAX >> 1));

    return conn;
}
```
第一个参数指定了是否为离岸点，其他部分没什么特别，普通的赋值操作。

#### 向离岸点申请ipv4地址 ####
调用requestAddresses方法
IpTunnel.c
```
static void requestAddresses(struct IpTunnel_Connection* conn, struct IpTunnel_pvt* context)
{
    if (Defined(Log_DEBUG)) {
        uint8_t addr[40];
        AddrTools_printIp(addr, conn->routeHeader.ip6);
        Log_debug(context->logger, "Requesting addresses from [%s] for connection [%d]",
                  addr, conn->number);
    }

    int number = conn->number;
    Dict d = Dict_CONST(
        String_CONST("q"), String_OBJ(String_CONST("IpTunnel_getAddresses")), Dict_CONST(
        String_CONST("txid"), String_OBJ((&(String){ .len = 4, .bytes = (char*)&number })),
        NULL
    ));
    struct Allocator* msgAlloc = Allocator_child(context->allocator);
    sendControlMessage(&d, conn, msgAlloc, context);
    Allocator_free(msgAlloc);
}
```
Dict中放入key为q，value为IpTunnel_getAddresses的键值对，调用sendControlMessage方法。
```
static void sendControlMessage(Dict* dict,
                               struct IpTunnel_Connection* connection,
                               struct Allocator* requestAlloc,
                               struct IpTunnel_pvt* context)
{
    struct Message* msg = Message_new(0, 1024, requestAlloc);
    BencMessageWriter_write(dict, msg, NULL);

    int length = msg->length;

    // do UDP header.
    Message_shift(msg, Headers_UDPHeader_SIZE, NULL);
    struct Headers_UDPHeader* uh = (struct Headers_UDPHeader*) msg->bytes;
    uh->srcPort_be = 0;
    uh->destPort_be = 0;
    uh->length_be = Endian_hostToBigEndian16(length);
    uh->checksum_be = 0;

    uint16_t payloadLength = msg->length;

    Message_shift(msg, Headers_IP6Header_SIZE, NULL);
    struct Headers_IP6Header* header = (struct Headers_IP6Header*) msg->bytes;
    header->versionClassAndFlowLabel = 0;
    header->flowLabelLow_be = 0;
    header->nextHeader = 17;
    header->hopLimit = 0;
    header->payloadLength_be = Endian_hostToBigEndian16(payloadLength);
    Headers_setIpVersion(header);

    // zero the source and dest addresses.
    Bits_memset(header->sourceAddr, 0, 32);

    uh->checksum_be = Checksum_udpIp6(header->sourceAddr,
                                      (uint8_t*) uh,
                                      msg->length - Headers_IP6Header_SIZE);

    Iface_CALL(sendToNode, msg, connection, context);
}
```
一些初始化操作，然后Iface_CALL调用sendToNode方法
```
static Iface_DEFUN sendToNode(struct Message* message,
                              struct IpTunnel_Connection* connection,
                              struct IpTunnel_pvt* context)
{
    Message_push(message, NULL, DataHeader_SIZE, NULL);
    struct DataHeader* dh = (struct DataHeader*) message->bytes;
    DataHeader_setContentType(dh, ContentType_IPTUN);
    DataHeader_setVersion(dh, DataHeader_CURRENT_VERSION);
    Message_push(message, &connection->routeHeader, RouteHeader_SIZE, NULL);
    return Iface_next(&context->pub.nodeInterface, message);
}
```
设置了ContentType和Version，调用Message_push方法调整消息内容。最后调用 Iface_next(&context->pub.nodeInterface, message)方法，将消息发出去。
此处的context->pub.nodeInterface是一个Iface，它的回调函数在IpTunnel_new方法中，被设为incomingFromNode。
```
        .pub = {
		..........
            .nodeInterface = { .send = incomingFromNode }
        },
```
它的connectedIf在Core.c的Core_init方法中通过Iface_plumb方法与nc->upper->ipTunnelIf绑定。
```
    Iface_plumb(&nc->upper->ipTunnelIf, &ipTunnel->nodeInterface);
```
```
static inline void Iface_plumb(struct Iface* a, struct Iface* b)
{
    Assert_true(!a->connectedIf);
    Assert_true(!b->connectedIf);
    a->connectedIf = b;
    b->connectedIf = a;
}
```

#### 收到离岸点分配的ipv4地址 ####
上面讲到，回调函数被设为incomingFromNode，所以，所有从其他点发送过来，进入IpTunnel的数据，都先到incomingFromNode方法中
```
static Iface_DEFUN incomingFromNode(struct Message* message, struct Iface* nodeIf)
{
    struct IpTunnel_pvt* context =
        Identity_containerOf(nodeIf, struct IpTunnel_pvt, pub.nodeInterface);

    //Log_debug(context->logger, "Got incoming message");

    Assert_true(message->length >= RouteHeader_SIZE + DataHeader_SIZE);
    struct RouteHeader* rh = (struct RouteHeader*) message->bytes;
    struct DataHeader* dh = (struct DataHeader*) &rh[1];
    Assert_true(DataHeader_getContentType(dh) == ContentType_IPTUN);
    struct IpTunnel_Connection* conn = connectionByPubKey(rh->publicKey, context);
    if (!conn) {
        if (Defined(Log_DEBUG)) {
            uint8_t addr[40];
            AddrTools_printIp(addr, rh->ip6);
            Log_debug(context->logger, "Got message from unrecognized node [%s]", addr);
        }
        return 0;
    }

    Message_shift(message, -(RouteHeader_SIZE + DataHeader_SIZE), NULL);

    if (message->length > 40 && Headers_getIpVersion(message->bytes) == 6) {
        return ip6FromNode(message, conn, context);
    }
    if (message->length > 20 && Headers_getIpVersion(message->bytes) == 4) {
        return ip4FromNode(message, conn, context);
    }

    if (Defined(Log_DEBUG)) {
        uint8_t addr[40];
        AddrTools_printIp(addr, rh->ip6);
        Log_debug(context->logger,
                  "Got message of unknown type, length: [%d], IP version [%d] from [%s]",
                  message->length,
                  (message->length > 1) ? Headers_getIpVersion(message->bytes) : 0,
                  addr);
    }
    return 0;
}
```
1. 通过publicKey找到对应的conn，也就是离岸点的conn。
```
struct IpTunnel_Connection* conn = connectionByPubKey(rh->publicKey, context);
```
2. 调用ip6FromNode
```
    if (message->length > 40 && Headers_getIpVersion(message->bytes) == 6) {
        return ip6FromNode(message, conn, context);
    }
```
```
static Iface_DEFUN ip6FromNode(struct Message* message,
                               struct IpTunnel_Connection* conn,
                               struct IpTunnel_pvt* context)
{
    struct Headers_IP6Header* header = (struct Headers_IP6Header*) message->bytes;
    if (Bits_isZero(header->sourceAddr, 16) || Bits_isZero(header->destinationAddr, 16)) {
        if (Bits_isZero(header->sourceAddr, 32)) {
            return incomingControlMessage(message, conn, context);
        }
        ......
        return 0;
    }
    if (!isValidAddress6(header->sourceAddr, false, conn)) {
        ......
        return 0;
    }
    ......
    return Iface_next(&context->pub.tunInterface, message);
}
```
此时header->sourceAddr为全0，会调用到incomingControlMessage方法。

```
static Iface_DEFUN incomingControlMessage(struct Message* message,
                                          struct IpTunnel_Connection* conn,
                                          struct IpTunnel_pvt* context)
{
    ......

    struct Allocator* alloc = Allocator_child(message->alloc);

    Dict* d = NULL;

    if (Dict_getDictC(d, "addresses")) {
        return incomingAddresses(d, conn, alloc, context);
    }
    ......
    return 0;
}
```
调用到incomingAddresses

```
static Iface_DEFUN incomingAddresses(Dict* d,
                                     struct IpTunnel_Connection* conn,
                                     struct Allocator* alloc,
                                     struct IpTunnel_pvt* context)
{
      //一些数据合法性判断

    Dict* addresses = Dict_getDictC(d, "addresses");

    String* ip4 = Dict_getStringC(addresses, "ip4");
    int64_t* ip4Prefix = Dict_getIntC(addresses, "ip4Prefix");
    int64_t* ip4Alloc = Dict_getIntC(addresses, "ip4Alloc");

    if (ip4 && ip4->len == 4) {
        Bits_memcpy(conn->connectionIp4, ip4->bytes, 4);

        if (ip4Prefix && *ip4Prefix >= 0 && *ip4Prefix <= 32) {
            conn->connectionIp4Prefix = (uint8_t) *ip4Prefix;
        } else {
            conn->connectionIp4Prefix = 32;
        }

        if (ip4Alloc && *ip4Alloc >= 0 && *ip4Alloc <= 32) {
            conn->connectionIp4Alloc = (uint8_t) *ip4Alloc;
        } else {
            conn->connectionIp4Alloc = 32;
        }

        struct Sockaddr* sa = Sockaddr_clone(Sockaddr_LOOPBACK, alloc);
        uint8_t* addrBytes = NULL;
        Sockaddr_getAddress(sa, &addrBytes);
        Bits_memcpy(addrBytes, ip4->bytes, 4);
        char* printedAddr = Sockaddr_print(sa, alloc);

        Log_info(context->logger, "Got issued address [%s/%d:%d] for connection [%d]",
                 printedAddr, conn->connectionIp4Alloc, conn->connectionIp4Prefix, conn->number);

        addAddress(printedAddr, conn->connectionIp4Prefix, conn->connectionIp4Alloc, context);
        Notification_doNotify_af(context->notification,
                OUTGOING_REACHABLE,REACHABLE,Ethernet_TYPE_IP4);
        conn->reachable = true;
    }

    //与ipv4类似的，ipv6的处理。不做具体分析。
    
    return 0;
}
```
从消息中取出ipv4相关的三个字段，这是离岸点分配给当前点的ipv4地址相关信息。设置conn中相关字段的内容。然后调用addAddress方法。

```
static void addAddress(char* printedAddr, uint8_t prefixLen,
                       uint8_t allocSize, struct IpTunnel_pvt* ctx)
{
    struct Sockaddr_storage ss;
    if (Sockaddr_parse(printedAddr, &ss)) {
        Log_error(ctx->logger, "Invalid ip, setting ip address on TUN");
        return;
    }
    ss.addr.flags |= Sockaddr_flags_PREFIX;
    ss.addr.prefix = prefixLen;

    bool installRoute = false;
    if (Sockaddr_getFamily(&ss.addr) == Sockaddr_AF_INET) {
        installRoute = (prefixLen < 32);
    } else if (Sockaddr_getFamily(&ss.addr) == Sockaddr_AF_INET6) {
        installRoute = (prefixLen < 128);
    } else {
        Assert_failure("bad address family");
    }
    if (installRoute) {
        RouteGen_addPrefix(ctx->rg, &ss.addr);
    }
    if (!ctx->ifName) {
        Log_error(ctx->logger, "Failed to set IP address because TUN interface is not setup");
        return;
    }

    ss.addr.prefix = allocSize;
    struct Jmp j;
    Jmp_try(j) {
        NetDev_addAddress(ctx->ifName->bytes, &ss.addr, ctx->logger, &j.handler);
    } Jmp_catch {
        Log_error(ctx->logger, "Error setting ip address on TUN [%s]", j.message);
        return;
    }
}
```
这里主要调用了两个方法：
```
RouteGen_addPrefix(ctx->rg, &ss.addr);
NetDev_addAddress(ctx->ifName->bytes, &ss.addr, ctx->logger, &j.handler);
```
已经不是IpTunnel范围内的代码了，具体内容不分析了。
至此，向离岸点申请ipv4，收到回复后设置ipv4的过程就结束了。

### 离岸点为每个允许连接过来的普通点建立一个IpTunnel_Connection ###
#### conf文件配置 ####
```
"allowedConnections": [
        {
            "ip4Address": "192.168.254.2",
            "ip4Prefix": 0,
            "ip4Alloc": 32,
            "publicKey": "uhmhts49tdm1ryb3q0pw95291uwt4xgvyk5s84vt2z2mnv3zp230.k"
        },
        {
            "ip4Address": "192.168.254.3",
            "ip4Prefix": 0,
            "ip4Alloc": 32,
            "publicKey": "9c3x7hp181dv91tfkbngyhhu2uc3xhxuuh539l3g0gdjgjg1bs10.k"
        }
]
```
#### 读取配置文件 ####
Configurator.c
```
    List* incoming = Dict_getListC(ifaceConf, "allowedConnections");
    if (incoming) {
        Dict* d;
        for (int i = 0; (d = List_getDict(incoming, i)) != NULL; i++) {
            String* key = Dict_getStringC(d, "publicKey");
            String* ip4 = Dict_getStringC(d, "ip4Address");
            // Note that the prefix length has to be a proper int in the config
            // (not quoted!)
            int64_t* ip4Prefix = Dict_getIntC(d, "ip4Prefix");
            String* ip6 = Dict_getStringC(d, "ip6Address");
            int64_t* ip6Prefix = Dict_getIntC(d, "ip6Prefix");
            if (!key) {
                Log_critical(ctx->logger, "In router.ipTunnel.allowedConnections[%d]"
                                          "'publicKey' required.", i);
                exit(1);
            }
            if (!ip4 && !ip6) {
                Log_critical(ctx->logger, "In router.ipTunnel.allowedConnections[%d]"
                                          "either 'ip4Address' or 'ip6Address' required.", i);
                exit(1);
            } else if (ip4Prefix && !ip4) {
                Log_critical(ctx->logger, "In router.ipTunnel.allowedConnections[%d]"
                                          "'ip4Address' required with 'ip4Prefix'.", i);
                exit(1);
            } else if (ip6Prefix && !ip6) {
                Log_critical(ctx->logger, "In router.ipTunnel.allowedConnections[%d]"
                                          "'ip6Address' required with 'ip6Prefix'.", i);
                exit(1);
            }
            Log_debug(ctx->logger, "Allowing IpTunnel connections from [%s]", key->bytes);

            if (ip4) {
                Log_debug(ctx->logger, "Issue IPv4 address %s", ip4->bytes);
                if (ip4Prefix) {
                    Log_debug(ctx->logger, "Issue IPv4 netmask/prefix length /%d",
                        (int) *ip4Prefix);
                } else {
                    Log_debug(ctx->logger, "Use default netmask/prefix length /0");
                }
            }

            if (ip6) {
                Log_debug(ctx->logger, "Issue IPv6 address [%s]", ip6->bytes);
                if (ip6Prefix) {
                    Log_debug(ctx->logger, "Issue IPv6 netmask/prefix length /%d",
                        (int) *ip6Prefix);
                } else {
                    Log_debug(ctx->logger, "Use default netmask/prefix length /0");
                }
            }

            Dict_putStringC(d, "publicKeyOfAuthorizedNode", key, tempAlloc);
            rpcCall0(String_CONST("IpTunnel_allowConnection"), d, ctx, tempAlloc, NULL, true);
        }
    }
```
读取配置文件中allowedConnections中各个点的相关信息，rpcall调用IpTunnel_allowConnection方法。

#### IpTunnel_allowConnection方法 ####
IpTunnel_admin.c
该方法在文件中注册为allowConnection函数。
```
static void allowConnection(Dict* args,
                            void* vcontext,
                            String* txid,
                            struct Allocator* requestAlloc)
{
    struct Context* context = (struct Context*) vcontext;
    String* publicKeyOfAuthorizedNode =
        Dict_getStringC(args, "publicKeyOfAuthorizedNode");
    String* ip6Address = Dict_getStringC(args, "ip6Address");
    int64_t* ip6Prefix = Dict_getIntC(args, "ip6Prefix");
    int64_t* ip6Alloc = Dict_getIntC(args, "ip6Alloc");
    String* ip4Address = Dict_getStringC(args, "ip4Address");
    int64_t* ip4Prefix = Dict_getIntC(args, "ip4Prefix");
    int64_t* ip4Alloc = Dict_getIntC(args, "ip4Alloc");

    uint8_t pubKey[32];
    uint8_t ip6Addr[16];

    struct Sockaddr_storage ip6ToGive;
    struct Sockaddr_storage ip4ToGive;

    char* error;
    int ret;
    if (!ip6Address && !ip4Address) {
        error = "Must specify ip6Address or ip4Address";
    } else if ((ret = Key_parse(publicKeyOfAuthorizedNode, pubKey, ip6Addr)) != 0) {
        error = Key_parse_strerror(ret);

    } else if (ip6Prefix && !ip6Address) {
        error = "Must specify ip6Address with ip6Prefix";
    } else if (ip6Alloc && !ip6Address) {
        error = "Must specify ip6Address with ip6Alloc";
    } else if (ip6Prefix && (*ip6Prefix > 128 || *ip6Prefix < 0)) {
        error = "ip6Prefix out of range: must be 0 to 128";
    } else if (ip6Alloc && (*ip6Alloc > 128 || *ip6Alloc < 1)) {
        error = "ip6Alloc out of range: must be 1 to 128";

    } else if (ip4Prefix && !ip4Address) {
        error = "Must specify ip4Address with ip4Prefix";
    } else if (ip4Alloc && !ip4Address) {
        error = "Must specify ip4Address with ip4Alloc";
    } else if (ip4Prefix && (*ip4Prefix > 32 || *ip4Prefix < 0)) {
        error = "ip4Prefix out of range: must be 0 to 32";
    } else if (ip4Alloc && (*ip4Alloc > 32 || *ip4Alloc < 1)) {
        error = "ip4Alloc out of range: must be 1 to 32";

    } else if (ip6Address
        && (Sockaddr_parse(ip6Address->bytes, &ip6ToGive)
            || Sockaddr_getFamily(&ip6ToGive.addr) != Sockaddr_AF_INET6))
    {
        error = "malformed ip6Address";
    } else if (ip4Address
        && (Sockaddr_parse(ip4Address->bytes, &ip4ToGive)
            || Sockaddr_getFamily(&ip4ToGive.addr) != Sockaddr_AF_INET))
    {
        error = "malformed ip4Address";
    } else {
        int conn = IpTunnel_allowConnection(pubKey,
                                            (ip6Address) ? &ip6ToGive.addr : NULL,
                                            (ip6Prefix) ? (uint8_t) (*ip6Prefix) : 128,
                                            (ip6Alloc) ? (uint8_t) (*ip6Alloc) : 128,
                                            (ip4Address) ? &ip4ToGive.addr : NULL,
                                            (ip4Prefix) ? (uint8_t) (*ip4Prefix) : 32,
                                            (ip4Alloc) ? (uint8_t) (*ip4Alloc) : 32,
                                            context->ipTun);
        sendResponse(conn, txid, context->admin);
        return;
    }

    sendError(error, txid, context->admin);
}
```
获取conf中该连接的相关信息，验证其合法性，调用IpTunnel_allowConnection方法，建立IpTunnel_Connection。

#### IpTunnel_allowConnection方法 ####
IpTunnel.c
```
int IpTunnel_allowConnection(uint8_t publicKeyOfAuthorizedNode[32],
                             struct Sockaddr* ip6Addr,
                             uint8_t ip6Prefix,
                             uint8_t ip6Alloc,
                             struct Sockaddr* ip4Addr,
                             uint8_t ip4Prefix,
                             uint8_t ip4Alloc,
                             struct IpTunnel* tunnel)
{
    struct IpTunnel_pvt* context = Identity_check((struct IpTunnel_pvt*)tunnel);

    Log_debug(context->logger, "IPv4 Prefix to allow: %d", ip4Prefix);

    uint8_t* ip6Address = NULL;
    uint8_t* ip4Address = NULL;
    if (ip6Addr) {
        Sockaddr_getAddress(ip6Addr, &ip6Address);
    }
    if (ip4Addr) {
        Sockaddr_getAddress(ip4Addr, &ip4Address);
    }
    Log_debug(context->logger, "noti outgoing call by IpTunnel_allowConnection");
    struct IpTunnel_Connection* conn = newConnection(false, context);
    Bits_memcpy(conn->routeHeader.publicKey, publicKeyOfAuthorizedNode, 32);
    AddressCalc_addressForPublicKey(conn->routeHeader.ip6, publicKeyOfAuthorizedNode);
    if (ip4Address) {
        Bits_memcpy(conn->connectionIp4, ip4Address, 4);
        conn->connectionIp4Prefix = ip4Prefix;
        conn->connectionIp4Alloc = ip4Alloc;
        Assert_true(ip4Alloc);
    }

    if (ip6Address) {
        Bits_memcpy(conn->connectionIp6, ip6Address, 16);
        conn->connectionIp6Prefix = ip6Prefix;
        conn->connectionIp6Alloc = ip6Alloc;
        Assert_true(ip6Alloc);
    }

    return conn->number;
}
```
这里主要三个操作：
1. 调用newConnection(true, context)创建一个IpTunnel_Connection类型的conn。
2. 将conf配置中的publicKey信息赋值给conn->routeHeader.publicKey
3. 将conf配置中的ip相关信息赋值给conn相关字段 

#### 创建一个IpTunnel_Connection类型的conn ####
调用newConnection方法。
IpTunnel.c
```
static struct IpTunnel_Connection* newConnection(bool isOutgoing, struct IpTunnel_pvt* context)
{
    if (context->pub.connectionList.count == context->connectionCapacity) {
        uint32_t newSize = (context->connectionCapacity + 4) * sizeof(struct IpTunnel_Connection);
        context->pub.connectionList.connections =
            Allocator_realloc(context->allocator, context->pub.connectionList.connections, newSize);
        context->connectionCapacity += 4;
    }
    struct IpTunnel_Connection* conn =
        &context->pub.connectionList.connections[context->pub.connectionList.count];

    // If it's an incoming connection, it must be lower on the list than any outgoing connections.
    if (!isOutgoing) {
        for (int i = (int)context->pub.connectionList.count - 1; i >= 0; i--) {
            if (!context->pub.connectionList.connections[i].isOutgoing
                && conn != &context->pub.connectionList.connections[i + 1])
            {
                Bits_memcpy(conn,
                                 &context->pub.connectionList.connections[i + 1],
                                 sizeof(struct IpTunnel_Connection));
                conn = &context->pub.connectionList.connections[i + 1];
            }
        }
    }

    context->pub.connectionList.count++;

    Bits_memset(conn, 0, sizeof(struct IpTunnel_Connection));
    conn->number = context->nextConnectionNumber++;
    conn->isOutgoing = isOutgoing;
    conn->reachable = false;

    // if there are 2 billion calls, die.
    Assert_true(context->nextConnectionNumber < (UINT32_MAX >> 1));

    return conn;
}
```
第一个参数指定了是否为离岸点，其他部分没什么特别，普通的赋值操作。

#### 收到其他点发送来的分配地址的请求 ####
所有从其他点发送过来，进入IpTunnel的数据，都先到incomingFromNode方法中
```
static Iface_DEFUN incomingFromNode(struct Message* message, struct Iface* nodeIf)
{
    struct IpTunnel_pvt* context =
        Identity_containerOf(nodeIf, struct IpTunnel_pvt, pub.nodeInterface);

    //Log_debug(context->logger, "Got incoming message");

    Assert_true(message->length >= RouteHeader_SIZE + DataHeader_SIZE);
    struct RouteHeader* rh = (struct RouteHeader*) message->bytes;
    struct DataHeader* dh = (struct DataHeader*) &rh[1];
    Assert_true(DataHeader_getContentType(dh) == ContentType_IPTUN);
    struct IpTunnel_Connection* conn = connectionByPubKey(rh->publicKey, context);
    if (!conn) {
        if (Defined(Log_DEBUG)) {
            uint8_t addr[40];
            AddrTools_printIp(addr, rh->ip6);
            Log_debug(context->logger, "Got message from unrecognized node [%s]", addr);
        }
        return 0;
    }

    Message_shift(message, -(RouteHeader_SIZE + DataHeader_SIZE), NULL);

    if (message->length > 40 && Headers_getIpVersion(message->bytes) == 6) {
        return ip6FromNode(message, conn, context);
    }
    if (message->length > 20 && Headers_getIpVersion(message->bytes) == 4) {
        return ip4FromNode(message, conn, context);
    }

    if (Defined(Log_DEBUG)) {
        uint8_t addr[40];
        AddrTools_printIp(addr, rh->ip6);
        Log_debug(context->logger,
                  "Got message of unknown type, length: [%d], IP version [%d] from [%s]",
                  message->length,
                  (message->length > 1) ? Headers_getIpVersion(message->bytes) : 0,
                  addr);
    }
    return 0;
}
```
1. 通过publicKey找到对应的conn。
```
struct IpTunnel_Connection* conn = connectionByPubKey(rh->publicKey, context);
```
2. 调用ip6FromNode
```
    if (message->length > 40 && Headers_getIpVersion(message->bytes) == 6) {
        return ip6FromNode(message, conn, context);
    }
```
```
static Iface_DEFUN ip6FromNode(struct Message* message,
                               struct IpTunnel_Connection* conn,
                               struct IpTunnel_pvt* context)
{
    struct Headers_IP6Header* header = (struct Headers_IP6Header*) message->bytes;
    if (Bits_isZero(header->sourceAddr, 16) || Bits_isZero(header->destinationAddr, 16)) {
        if (Bits_isZero(header->sourceAddr, 32)) {
            return incomingControlMessage(message, conn, context);
        }
        ......
        return 0;
    }
    if (!isValidAddress6(header->sourceAddr, false, conn)) {
        ......
        return 0;
    }
    ......
    return Iface_next(&context->pub.tunInterface, message);
}
```
此时header->sourceAddr为全0，会调用到incomingControlMessage方法。

```
static Iface_DEFUN incomingControlMessage(struct Message* message,
                                          struct IpTunnel_Connection* conn,
                                          struct IpTunnel_pvt* context)
{
       ......
    Dict* d = NULL;
    char* err = BencMessageReader_readNoExcept(message, alloc, &d);
    if (err) {
        Log_info(context->logger, "Failed to parse message [%s]", err);
        return 0;
    }

     ....
    if (String_equals(String_CONST("IpTunnel_getAddresses"),
                      Dict_getStringC(d, "q")))
    {
        return requestForAddresses(d, conn, alloc, context);
    }
    Log_warn(context->logger, "Message which is unhandled");
    return 0;
}
```
因为普通点在向离岸点发送分配ipv4地址的请求时，会将q字段的值设为IpTunnel_getAddresses，所以，在离岸点收到这类信息时，会调用到requestForAddresses方法。
```
static Iface_DEFUN requestForAddresses(Dict* request,
                                       struct IpTunnel_Connection* conn,
                                       struct Allocator* requestAlloc,
                                       struct IpTunnel_pvt* context)
{
    ......
    Dict* addresses = Dict_new(requestAlloc);
    bool noAddresses = true;
    if (!Bits_isZero(conn->connectionIp6, 16)) {
        ......
    }
    if (!Bits_isZero(conn->connectionIp4, 4)) {
        Dict_putStringC(addresses,
                       "ip4",
                       String_newBinary((char*)conn->connectionIp4, 4, requestAlloc),
                       requestAlloc);
        Dict_putIntC(addresses,
                    "ip4Prefix", (int64_t)conn->connectionIp4Prefix,
                    requestAlloc);
        Dict_putIntC(addresses,
                    "ip4Alloc", (int64_t)conn->connectionIp4Alloc,
                    requestAlloc);

        noAddresses = false;
    }
    if (noAddresses) {
        Log_warn(context->logger, "no addresses to provide");
        return 0;
    }

    Dict* msg = Dict_new(requestAlloc);
    Dict_putDictC(msg, "addresses", addresses, requestAlloc);

    String* txid = Dict_getStringC(request, "txid");
    if (txid) {
        Dict_putStringC(msg, "txid", txid, requestAlloc);
    }

    sendControlMessage(msg, conn, requestAlloc, context);
    return 0;
}
```
将conf文件中，分配给这个连接的ipv4地址相关信息，填入addresses中，再将key为addresses，value为addresses的键值对，放入Dict中，调用sendControlMessage发送出去。当普通点收到这个回复后，会从中取出addresses，并进行ipv4相关字段的设置。
```
static void sendControlMessage(Dict* dict,
                               struct IpTunnel_Connection* connection,
                               struct Allocator* requestAlloc,
                               struct IpTunnel_pvt* context)
{
    struct Message* msg = Message_new(0, 1024, requestAlloc);
    BencMessageWriter_write(dict, msg, NULL);

    int length = msg->length;

    // do UDP header.
    Message_shift(msg, Headers_UDPHeader_SIZE, NULL);
    struct Headers_UDPHeader* uh = (struct Headers_UDPHeader*) msg->bytes;
    uh->srcPort_be = 0;
    uh->destPort_be = 0;
    uh->length_be = Endian_hostToBigEndian16(length);
    uh->checksum_be = 0;

    uint16_t payloadLength = msg->length;

    Message_shift(msg, Headers_IP6Header_SIZE, NULL);
    struct Headers_IP6Header* header = (struct Headers_IP6Header*) msg->bytes;
    header->versionClassAndFlowLabel = 0;
    header->flowLabelLow_be = 0;
    header->nextHeader = 17;
    header->hopLimit = 0;
    header->payloadLength_be = Endian_hostToBigEndian16(payloadLength);
    Headers_setIpVersion(header);

    // zero the source and dest addresses.
    Bits_memset(header->sourceAddr, 0, 32);

    uh->checksum_be = Checksum_udpIp6(header->sourceAddr,
                                      (uint8_t*) uh,
                                      msg->length - Headers_IP6Header_SIZE);

    Iface_CALL(sendToNode, msg, connection, context);
}
```
调用sendToNode
```
static Iface_DEFUN sendToNode(struct Message* message,
                              struct IpTunnel_Connection* connection,
                              struct IpTunnel_pvt* context)
{
    Message_push(message, NULL, DataHeader_SIZE, NULL);
    struct DataHeader* dh = (struct DataHeader*) message->bytes;
    DataHeader_setContentType(dh, ContentType_IPTUN);
    DataHeader_setVersion(dh, DataHeader_CURRENT_VERSION);
    Message_push(message, &connection->routeHeader, RouteHeader_SIZE, NULL);
    return Iface_next(&context->pub.nodeInterface, message);
}
```
至此，离岸点在收到普通点的ipv4分配申请后的执行流程分析完毕。

## 处理从其他点发来的消息 ##
上面讲到，所有从其他点发送过来，进入IpTunnel的数据，都先到incomingFromNode方法中
```
static Iface_DEFUN incomingFromNode(struct Message* message, struct Iface* nodeIf)
{
    struct IpTunnel_pvt* context =
        Identity_containerOf(nodeIf, struct IpTunnel_pvt, pub.nodeInterface);

    Assert_true(message->length >= RouteHeader_SIZE + DataHeader_SIZE);
    struct RouteHeader* rh = (struct RouteHeader*) message->bytes;
    struct DataHeader* dh = (struct DataHeader*) &rh[1];
    Assert_true(DataHeader_getContentType(dh) == ContentType_IPTUN);
    struct IpTunnel_Connection* conn = connectionByPubKey(rh->publicKey, context);
    if (!conn) {
        if (Defined(Log_DEBUG)) {
            uint8_t addr[40];
            AddrTools_printIp(addr, rh->ip6);
            Log_debug(context->logger, "Got message from unrecognized node [%s]", addr);
        }
        return 0;
    }

    Message_shift(message, -(RouteHeader_SIZE + DataHeader_SIZE), NULL);

    if (message->length > 40 && Headers_getIpVersion(message->bytes) == 6) {
        return ip6FromNode(message, conn, context);
    }
    if (message->length > 20 && Headers_getIpVersion(message->bytes) == 4) {
        return ip4FromNode(message, conn, context);
    }

    if (Defined(Log_DEBUG)) {
        uint8_t addr[40];
        AddrTools_printIp(addr, rh->ip6);
        Log_debug(context->logger,
                  "Got message of unknown type, length: [%d], IP version [%d] from [%s]",
                  message->length,
                  (message->length > 1) ? Headers_getIpVersion(message->bytes) : 0,
                  addr);
    }
    return 0;
}
```
以ipv4为例，进入到ip4FromNode
```
static Iface_DEFUN ip4FromNode(struct Message* message,
                               struct IpTunnel_Connection* conn,
                               struct IpTunnel_pvt* context)
{
    struct Headers_IP4Header* header = (struct Headers_IP4Header*) message->bytes;
    if (Bits_isZero(header->sourceAddr, 4) || Bits_isZero(header->destAddr, 4)) {
        Log_debug(context->logger, "Got message with zero address");
        return 0;
    } else if (!isValidAddress4(header->sourceAddr, false, conn)) {
        Log_debug(context->logger, "Got message with wrong address [%d.%d.%d.%d] for connection "
                                   "[%d.%d.%d.%d/%d:%d]",
                  header->sourceAddr[0], header->sourceAddr[1],
                  header->sourceAddr[2], header->sourceAddr[3],
                  conn->connectionIp4[0], conn->connectionIp4[1],
                  conn->connectionIp4[2], conn->connectionIp4[3],
                  conn->connectionIp4Alloc, conn->connectionIp4Prefix);
        return 0;
    }
    Log_debug(context->logger, "jin Got message with address [%d.%d.%d.%d] for connection "
                                   "[%d.%d.%d.%d/%d:%d]",
                  header->sourceAddr[0], header->sourceAddr[1],
                  header->sourceAddr[2], header->sourceAddr[3],
                  conn->connectionIp4[0], conn->connectionIp4[1],
                  conn->connectionIp4[2], conn->connectionIp4[3],
                  conn->connectionIp4Alloc, conn->connectionIp4Prefix);
    TUNMessageType_push(message, Ethernet_TYPE_IP4, NULL);
    return Iface_next(&context->pub.tunInterface, message);
}
```
此时，header->sourceAddr是对方的ipv4地址，不会为全0，代码进入到一个ip地址合法性判断：isValidAddress4(header->sourceAddr, false, conn)。如果地址合法，消息进一步发送到context->pub.tunInterface中。
tunInterface是一个Iface，它的回调函数在IpTunnel_new中设置为incomingFromTun
```
        .pub = {
            .tunInterface = { .send = incomingFromTun },
        },
```
它的connectedIf在Core.c的Core_init方法中通过Iface_plumb方法与nc->tunAdapt->ipTunnelIf绑定。
```
 Iface_plumb(&nc->tunAdapt->ipTunnelIf, &ipTunnel->tunInterface);
```
```
static inline void Iface_plumb(struct Iface* a, struct Iface* b)
{
    Assert_true(!a->connectedIf);
    Assert_true(!b->connectedIf);
    a->connectedIf = b;
    b->connectedIf = a;
}
```

### 查看地址合法性判断 ###
```
static bool isValidAddress4(uint8_t sourceAndDestIp4[8],
                            bool isFromTun,
                            struct IpTunnel_Connection* conn)
{
    uint8_t* compareAddr = (isFromTun)
        ? ((conn->isOutgoing) ? sourceAndDestIp4 : &sourceAndDestIp4[4])
        : ((conn->isOutgoing) ? &sourceAndDestIp4[4] : sourceAndDestIp4);
    return prefixMatches4(compareAddr, conn->connectionIp4, conn->connectionIp4Alloc);
}
```
ifFromTun为false，接下来根据isOutgoing的值来决定compareAddr的值，分为两种情况
1. 当前点是普通点，那么他收到的消息是从离岸点发送过来的，isOutgoing为
true，compareAddr赋值为sourceAndDestIp4[4]，也就是dest ip,从离岸点发来的包的dest ip就是当前这个普通点的ipv4地址。
2. 当前点是离岸点，那么它收到的消息是从普通点发送过来的，isOutgoing为
false，compareAddr赋值为sourceAndDestIp4，也就是source ip，从普通点发来的包的source ip就是普通点的ipv4地址。
综上，compareAddr一定会被赋值为普通点的ipv4地址。接下来调用prefixMatches4方法。
```
static bool prefixMatches4(uint8_t* addressA, uint8_t* refAddr, uint32_t prefixLen)
{
    if (!prefixLen) {
        Assert_true(Bits_isZero(refAddr, 4));
        return false;
    }
    Assert_true(prefixLen && prefixLen <= 32);
    uint32_t a = GET32(addressA);
    uint32_t b = GET32(refAddr);
    return !((a ^ b) >> (32 - prefixLen));
}
```
这就只是一个简单的地址比较了。

## 处理从Tun设备发来的消息 ##
```
static Iface_DEFUN incomingFromTun(struct Message* message, struct Iface* tunIf)
{
    struct IpTunnel_pvt* context = Identity_check((struct IpTunnel_pvt*)tunIf);

    if (message->length < 20) {
        Log_debug(context->logger, "DROP runt");
    }

    struct IpTunnel_Connection* conn = NULL;
    if (!context->pub.connectionList.connections) {
        // No connections authorized, fall through to "unrecognized address"
    } else if (message->length > 40 && Headers_getIpVersion(message->bytes) == 6) {
        struct Headers_IP6Header* header = (struct Headers_IP6Header*) message->bytes;
        conn = findConnection(header->sourceAddr, NULL, true, context);
    } else if (message->length > 20 && Headers_getIpVersion(message->bytes) == 4) {
        struct Headers_IP4Header* header = (struct Headers_IP4Header*) message->bytes;
        conn = findConnection(NULL, header->sourceAddr, true, context);
    } else {
        Log_info(context->logger, "Message of unknown type from TUN");
        return 0;
    }

    if (!conn) {
        Log_info(context->logger, "Message with unrecognized address from TUN");
        return 0;
    }

    return sendToNode(message, conn, context);
}
```
这里主要进行了两个操作：
1. 调用findConnection查找conn
2. 调用sendToNode

### 查找conn ###
```
static struct IpTunnel_Connection* findConnection(uint8_t sourceAndDestIp6[32],
                                                  uint8_t sourceAndDestIp4[8],
                                                  bool isFromTun,
                                                  struct IpTunnel_pvt* context)
{
    for (int i = 0; i < (int)context->pub.connectionList.count; i++) {
        struct IpTunnel_Connection* conn = &context->pub.connectionList.connections[i];
        if (sourceAndDestIp6 && isValidAddress6(sourceAndDestIp6, isFromTun, conn)) {
            return conn;
        }
        if (sourceAndDestIp4 && isValidAddress4(sourceAndDestIp4, isFromTun, conn)) {
            return conn;
        }
    }
    return NULL;
}
```
以ipv4为例，调用isValidAddress4检查地址合法性
```
static bool isValidAddress4(uint8_t sourceAndDestIp4[8],
                            bool isFromTun,
                            struct IpTunnel_Connection* conn)
{
    uint8_t* compareAddr = (isFromTun)
        ? ((conn->isOutgoing) ? sourceAndDestIp4 : &sourceAndDestIp4[4])
        : ((conn->isOutgoing) ? &sourceAndDestIp4[4] : sourceAndDestIp4);
    return prefixMatches4(compareAddr, conn->connectionIp4, conn->connectionIp4Alloc);
}
```
ifFromTun为true，接下来根据isOutgoing的值来决定compareAddr的值，分为两种情况
1. 当前点是普通点，那么他的消息是发往离岸点的，isOutgoing为
true，compareAddr赋值为sourceAndDestIp4，也就是source ip,是当前这个普通点的ipv4地址。
2. 当前点是离岸点，那么他的消息是发往普通点的，isOutgoing为
false，compareAddr赋值为sourceAndDestIp4[4]，也就是dest ip，发往普通点的包的dest ip就是普通点的ipv4地址。
综上，compareAddr一定会被赋值为普通点的ipv4地址。接下来调用prefixMatches4方法，进行地址对比。

### sendToNode ###
```
static Iface_DEFUN sendToNode(struct Message* message,
                              struct IpTunnel_Connection* connection,
                              struct IpTunnel_pvt* context)
{
    Message_push(message, NULL, DataHeader_SIZE, NULL);
    struct DataHeader* dh = (struct DataHeader*) message->bytes;
    DataHeader_setContentType(dh, ContentType_IPTUN);
    DataHeader_setVersion(dh, DataHeader_CURRENT_VERSION);
    Message_push(message, &connection->routeHeader, RouteHeader_SIZE, NULL);
    return Iface_next(&context->pub.nodeInterface, message);
}
```
至此，IpTunnel收到从Tun上和从其他点发来的包，并进行处理的过程分析完成。

