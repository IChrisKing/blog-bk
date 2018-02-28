---
title: cjdns源码分析--SwitchPing的发送，接收与回复
category:
  - cjdns
  - cjdns源码分析
tags:
  - cjdns源码分析
  - cjdns
  - C/C++
description: “SwitchPing是peer之间保活的重要手段，分析这个过程，可以理解在cjdns中，peer之间是如何进行ping/pong操作，来保持联系的。ping是一个互相交互的动作，节点之间会互相发送ping，来保证连接的有效性。收到ping的节点会回复pong来回应对方的ping。”
date: 2017-01-12 17:50:41
---
## 总体概述
### 源码位置
SwitchPing相关的代码多数在net目录下，主要包括
ControlHandler.c
ControlHandler.h
InterfaceController.c
InterfaceController.h
SwitchPinger.c
SwitchPinger.h
还有一些在util目录下
Pinger.c
Pinger.h

### 调用过程
ping操作的调用过程
![image](/assets/img/ping_pong/ping.png) 
pong操作的调用过程
![image](/assets/img/ping_pong/pong.png) 

## ping发送
### sendPing
net/InterfaceController.c
static void sendPing(struct Peer* ep)  发送ping
```
static void sendPing(struct Peer* ep)
{
    struct InterfaceController_pvt* ic = Identity_check(ep->ici->ic);

    ep->pingCount++;

    struct SwitchPinger_Ping* ping =
        SwitchPinger_newPing(ep->addr.path,
                             String_CONST(""),
                             ic->timeoutMilliseconds,
                             onPingResponse,
                             ep->alloc,
                             ic->switchPinger);

    if (Defined(Log_DEBUG)) {
        uint8_t key[56];
        Base32_encode(key, 56, ep->caSession->herPublicKey, 32);
        if (!ping) {
            Log_debug(ic->logger, "Failed to ping [%s.k], out of ping slots", key);
        } else {
            Log_debug(ic->logger, "SwitchPing [%s.k]", key);
        }
    }

    if (ping) {
        ping->onResponseContext = ep;
    }
}
```
参数是Peer* ep，说明在发送ping的时候，肯定是知道对方的信息的。

最后
```
    if (ping) {
        ping->onResponseContext = ep;
    }
```
看来整个ping的过程都在SwitchPinger_newPing当中了，查看这个函数
### SwitchPinger_newPing
net/SwitchPinger.c

```
struct SwitchPinger_Ping* SwitchPinger_newPing(uint64_t label,
                                               String* data,
                                               uint32_t timeoutMilliseconds,
                                               SwitchPinger_ResponseCallback onResponse,
                                               struct Allocator* alloc,
                                               struct SwitchPinger* context)
```
先结合调用过程，看一下参数。
1. uint64_t label -- ep->addr.path
是address的path部分，形如0000.0000.0000.0023
2. String* data -- String_CONST("")
携带的数据，ping时，携带的数据是""，即没数据。
3. uint32_t timeoutMilliseconds -- ic->timeoutMilliseconds
超时时间
4. SwitchPinger_ResponseCallback onResponse -- onPingResponse
SwitchPinger_ResponseCallback是一个回调函数，它的类型定义在Switchinger.h中
```
typedef void (* SwitchPinger_ResponseCallback)(struct SwitchPinger_Response* resp, void* userData);
```
它的实现在InterfaceController.c中
```
static void onPingResponse(struct SwitchPinger_Response* resp, void* onResponseContext)
{
    if (SwitchPinger_Result_OK != resp->res) {
        return;
    }
    struct Peer* ep = Identity_check((struct Peer*) onResponseContext);
    struct InterfaceController_pvt* ic = Identity_check(ep->ici->ic);

    ep->addr.protocolVersion = resp->version;

    if (Defined(Log_DEBUG)) {
        String* addr = Address_toString(&ep->addr, resp->ping->pingAlloc);
        if (!Version_isCompatible(Version_CURRENT_PROTOCOL, resp->version)) {
            Log_debug(ic->logger, "got switch pong from node [%s] with incompatible version",
                                  addr->bytes);
        } else if (ep->addr.path != resp->label) {
            uint8_t sl[20];
            AddrTools_printPath(sl, resp->label);
            Log_debug(ic->logger, "got switch pong from node [%s] mismatch label [%s]",
                                  addr->bytes, sl);
        } else {
            Log_debug(ic->logger, "got switch pong from node [%s]", addr->bytes);
        }
    }

    if (!Version_isCompatible(Version_CURRENT_PROTOCOL, resp->version)) {
        return;
    }

    if (ep->state == InterfaceController_PeerState_ESTABLISHED) {
        sendPeer(0xffffffff, PFChan_Core_PEER, ep);
    }

    ep->timeOfLastPing = Time_currentTimeMilliseconds(ic->eventBase);

    if (Defined(Log_DEBUG)) {
        String* addr = Address_toString(&ep->addr, resp->ping->pingAlloc);
        Log_debug(ic->logger, "Received [%s] from lazy endpoint [%s]",
                  SwitchPinger_resultString(resp->res)->bytes, addr->bytes);
    }
}
```
5. struct Allocator* alloc -- ep->alloc
6. struct SwitchPinger* context -- ic->switchPinger
SwitchPinger的定义在SwitchPinger.h中。
```
struct SwitchPinger
{
    struct Iface controlHandlerIf;
};
```

接下来分析SwitchPinger_newPing的内容。
### SwitchPinger_newPing
```
struct SwitchPinger_Ping* SwitchPinger_newPing(uint64_t label,
                                               String* data,
                                               uint32_t timeoutMilliseconds,
                                               SwitchPinger_ResponseCallback onResponse,
                                               struct Allocator* alloc,
                                               struct SwitchPinger* context)
{
    struct SwitchPinger_pvt* ctx = Identity_check((struct SwitchPinger_pvt*)context);
    if (data && data->len > Control_Ping_MAX_SIZE) {
        return NULL;
    }

    if (ctx->outstandingPings > ctx->maxConcurrentPings) {
        Log_debug(ctx->logger, "Skipping switch ping because there are already [%d] outstanding",
                  ctx->outstandingPings);
        return NULL;
    }

    struct Pinger_Ping* pp =
        Pinger_newPing(data, onPingResponse, sendPing, timeoutMilliseconds, alloc, ctx->pinger);

    struct Ping* ping = Allocator_clone(pp->pingAlloc, (&(struct Ping) {
        .pub = {
            .pingAlloc = pp->pingAlloc
        },
        .label = label,
        .data = String_clone(data, pp->pingAlloc),
        .context = ctx,
        .onResponse = onResponse,
        .pingerPing = pp
    }));
    Identity_set(ping);
    Allocator_onFree(pp->pingAlloc, onPingFree, ping);
    pp->context = ping;
    ctx->outstandingPings++;

    return &ping->pub;
}
```
先看
```
struct Pinger_Ping* pp =
        Pinger_newPing(data, onPingResponse, sendPing, timeoutMilliseconds, alloc, ctx->pinger);
```
#### Pinger_newPing
util/Pinger.c
```
struct Pinger_Ping* Pinger_newPing(String* data,
                                   Pinger_ON_RESPONSE(onResponse),
                                   Pinger_SEND_PING(sendPing),
                                   uint32_t timeoutMilliseconds,
                                   struct Allocator* allocator,
                                   struct Pinger* pinger)
```
首先也分析一下参数
1. String* data -- data
ping时携带的数据
2. Pinger_ON_RESPONSE(onResponse) -- onPingResponse
Pinger_SEND_PING(sendPing) -- sendPing
这是两个函数，定义在Pinger.h
```
#define Pinger_ON_RESPONSE(x) \
    void (* x)(String* data, uint32_t milliseconds, void* context)

#define Pinger_SEND_PING(x) void (* x)(String* data, void* context)
```
实现在SwitchPinger.c，暂时不分析具体实现代码。
3. struct Pinger* pinger --  ctx->pinger

接下来分析代码
```
struct Pinger_Ping* Pinger_newPing(String* data,
                                   Pinger_ON_RESPONSE(onResponse),
                                   Pinger_SEND_PING(sendPing),
                                   uint32_t timeoutMilliseconds,
                                   struct Allocator* allocator,
                                   struct Pinger* pinger)
{
    struct Allocator* alloc = Allocator_child(allocator);

    struct Ping* ping = Allocator_clone(alloc, (&(struct Ping) {
        .pub = {
            .pingAlloc = alloc,
        },
        .sendPing = sendPing,
        .pinger = pinger,
        .timeSent = Time_currentTimeMilliseconds(pinger->eventBase),
        .onResponse = onResponse
    }));
    Identity_set(ping);

    int pingIndex = Map_OutstandingPings_put(&ping, &pinger->outstandingPings);
    ping->pub.handle = pinger->outstandingPings.handles[pingIndex] + pinger->baseHandle;

    ping->cookie = Random_uint64(pinger->rand);

    // Prefix the data with the handle and cookie
    String* toSend = String_newBinary(NULL, ((data) ? data->len : 0) + 12, alloc);
    Bits_memcpy(toSend->bytes, &ping->pub.handle, 4);
    Bits_memcpy(&toSend->bytes[4], &ping->cookie, 8);
    if (data) {
        Bits_memcpy(toSend->bytes + 12, data->bytes, data->len);
    }
    ping->data = toSend;

    Allocator_onFree(alloc, freePing, ping);

    ping->timeout =
        Timeout_setTimeout(timeoutCallback, ping, timeoutMilliseconds, pinger->eventBase, alloc);

    Timeout_setTimeout(asyncSendPing, ping, 0, pinger->eventBase, alloc);

    return &ping->pub;
}
```
主要完成的事情就是构造了一个Ping* ping，发送ping的函数是sendPing,接受response的函数是onResponse,这两个函数的实现都在SwitchPinger.c中。
发送ping的动作在
```
Timeout_setTimeout(asyncSendPing, ping, 0, pinger->eventBase, alloc);
```
的asyncSendPing中，这是一个函数，

### asyncSendPing
```
static void asyncSendPing(void* vping)
{
    struct Ping* p = Identity_check((struct Ping*) vping);
    //Log_debug(p->pinger->logger, "Sending ping [%u]", p->pub.handle);
    p->sendPing(p->data, p->pub.context);
}
```
可以看到它是调用了p->sendPing(p->data, p->pub.context);方法，也就是SwitchPinger.c中的sendPing方法。

#### Allocator_clone(pp->pingAlloc, (&(struct Ping).....
接下来回到SwitchPinger_newPing函数中，查看
```
struct Ping* ping = Allocator_clone(pp->pingAlloc, (&(struct Ping) {
        .pub = {
            .pingAlloc = pp->pingAlloc
        },
        .label = label,
        .data = String_clone(data, pp->pingAlloc),
        .context = ctx,
        .onResponse = onResponse,
        .pingerPing = pp
    }));
```
这个函数主要构造了一个Ping* ping。它的pingering是上一步获得的Pinger_Ping* pp；它的onResponse是InterfaceController.c中的函数onPingResponse。

### SwitchPinger.c中的sendPing函数
上面提到sendPing会最终调用到SwitchPinger.c中的sendPing函数。
```
static void sendPing(String* data, void* sendPingContext)
{
    struct Ping* p = Identity_check((struct Ping*) sendPingContext);

    struct Message* msg = Message_new(0, data->len + 512, p->pub.pingAlloc);

    while (((uintptr_t)msg->bytes - data->len) % 4) {
        Message_push8(msg, 0, NULL);
    }
    msg->length = 0;

    Message_push(msg, data->bytes, data->len, NULL);
    Assert_true(!((uintptr_t)msg->bytes % 4) && "alignment fault");

    if (p->pub.keyPing) {
        Message_shift(msg, Control_KeyPing_HEADER_SIZE, NULL);
        struct Control_KeyPing* keyPingHeader = (struct Control_KeyPing*) msg->bytes;
        keyPingHeader->magic = Control_KeyPing_MAGIC;
        keyPingHeader->version_be = Endian_hostToBigEndian32(Version_CURRENT_PROTOCOL);
        Bits_memcpy(keyPingHeader->key, p->context->myAddr->key, 32);
    } else {
        Message_shift(msg, Control_Ping_HEADER_SIZE, NULL);
        struct Control_Ping* pingHeader = (struct Control_Ping*) msg->bytes;
        pingHeader->magic = Control_Ping_MAGIC;
        pingHeader->version_be = Endian_hostToBigEndian32(Version_CURRENT_PROTOCOL);
    }

    Message_shift(msg, Control_Header_SIZE, NULL);
    struct Control* ctrl = (struct Control*) msg->bytes;
    ctrl->header.checksum_be = 0;
    ctrl->header.type_be = (p->pub.keyPing) ? Control_KEYPING_be : Control_PING_be;
    ctrl->header.checksum_be = Checksum_engine(msg->bytes, msg->length);

    struct RouteHeader rh;
    Bits_memset(&rh, 0, RouteHeader_SIZE);
    rh.flags |= RouteHeader_flags_CTRLMSG;
    rh.sh.label_be = Endian_hostToBigEndian64(p->label);
    SwitchHeader_setVersion(&rh.sh, SwitchHeader_CURRENT_VERSION);

    Message_push(msg, &rh, RouteHeader_SIZE, NULL);

    Iface_send(&p->context->pub.controlHandlerIf, msg);
}
```
构造message，把ping需要的内容放入其中。最后调用Iface_send(&p->context->pub.controlHandlerIf, msg);发送这个message。

### incomingFromSwitchPinger
这个函数的实现，在ControlHandler.c中。
```
static Iface_DEFUN incomingFromSwitchPinger(struct Message* msg, struct Iface* switchPingerIf)
{
    struct ControlHandler_pvt* ch =
        Identity_containerOf(switchPingerIf, struct ControlHandler_pvt, pub.switchPingerIf);
    return Iface_next(&ch->pub.coreIf, msg);
}
```


## 收到ping,发回pong
按照log来看，收到ping和pong的处理，都是在ControlHandler.c的中incomingFromCore当中。
### 收到ping和pong
```
static Iface_DEFUN incomingFromCore(struct Message* msg, struct Iface* coreIf)
{
    struct ControlHandler_pvt* ch = Identity_check((struct ControlHandler_pvt*) coreIf);

    struct RouteHeader routeHdr;
    Message_pop(msg, &routeHdr, RouteHeader_SIZE, NULL);
    uint8_t labelStr[20];
    uint64_t label = Endian_bigEndianToHost64(routeHdr.sh.label_be);
    AddrTools_printPath(labelStr, label);
    Log_debug(ch->log, "ctrl packet from [%s]", labelStr);

    if (msg->length < 4 + Control_Header_SIZE) {
        Log_info(ch->log, "DROP runt ctrl packet from [%s]", labelStr);
        return NULL;
    }

    Assert_true(routeHdr.flags & RouteHeader_flags_CTRLMSG);

    if (Checksum_engine(msg->bytes, msg->length)) {
        Log_info(ch->log, "DROP ctrl packet from [%s] with invalid checksum", labelStr);
        return NULL;
    }

    struct Control* ctrl = (struct Control*) msg->bytes;

    if (ctrl->header.type_be == Control_ERROR_be) {
        return handleError(msg, ch, label, labelStr);

    } else if (ctrl->header.type_be == Control_KEYPING_be
            || ctrl->header.type_be == Control_PING_be)
    {
        return handlePing(msg, ch, label, labelStr, ctrl->header.type_be);

    } else if (ctrl->header.type_be == Control_KEYPONG_be
            || ctrl->header.type_be == Control_PONG_be)
    {
        Log_debug(ch->log, "got switch pong from [%s]", labelStr);
        Message_push(msg, &routeHdr, RouteHeader_SIZE, NULL);
        return Iface_next(&ch->pub.switchPingerIf, msg);
    }

    Log_info(ch->log, "DROP control packet of unknown type from [%s], type [%d]",
             labelStr, Endian_bigEndianToHost16(ctrl->header.type_be));

    return NULL;
}

```
处理ping消息调用到
```
 return handlePing(msg, ch, label, labelStr, ctrl->header.type_be);
```
查看这个方法
 
### handlePing
 ```
 static Iface_DEFUN handlePing(struct Message* msg,
                              struct ControlHandler_pvt* ch,
                              uint64_t label,
                              uint8_t* labelStr,
                              uint16_t messageType_be)
{
    if (msg->length < handlePing_MIN_SIZE) {
        Log_info(ch->log, "DROP runt ping");
        return NULL;
    }

    struct Control* ctrl = (struct Control*) msg->bytes;
    Message_shift(msg, -Control_Header_SIZE, NULL);

    // Ping and keyPing share version location
    struct Control_Ping* ping = (struct Control_Ping*) msg->bytes;
    uint32_t herVersion = Endian_bigEndianToHost32(ping->version_be);
    if (!Version_isCompatible(Version_CURRENT_PROTOCOL, herVersion)) {
        Log_debug(ch->log, "DROP ping from incompatible version [%d]", herVersion);
        return NULL;
    }

    if (messageType_be == Control_KEYPING_be) {
        Log_debug(ch->log, "got switch keyPing from [%s]", labelStr);
        if (msg->length < Control_KeyPing_HEADER_SIZE) {
            // min keyPing size is longer
            Log_debug(ch->log, "DROP runt keyPing");
            return NULL;
        }
        if (msg->length > Control_KeyPing_MAX_SIZE) {
            Log_debug(ch->log, "DROP long keyPing");
            return NULL;
        }
        if (ping->magic != Control_KeyPing_MAGIC) {
            Log_debug(ch->log, "DROP keyPing (bad magic)");
            return NULL;
        }

        struct Control_KeyPing* keyPing = (struct Control_KeyPing*) msg->bytes;
        keyPing->magic = Control_KeyPong_MAGIC;
        ctrl->header.type_be = Control_KEYPONG_be;
        Bits_memcpy(keyPing->key, ch->myPublicKey, 32);

    } else if (messageType_be == Control_PING_be) {
        Log_debug(ch->log, "got switch ping from [%s]", labelStr);
        if (ping->magic != Control_Ping_MAGIC) {
            Log_debug(ch->log, "DROP ping (bad magic)");
            return NULL;
        }
        ping->magic = Control_Pong_MAGIC;
        ctrl->header.type_be = Control_PONG_be;

    } else {
        Assert_failure("2+2=5");
    }

    ping->version_be = Endian_hostToBigEndian32(Version_CURRENT_PROTOCOL);

    Message_shift(msg, Control_Header_SIZE, NULL);

    ctrl->header.checksum_be = 0;
    ctrl->header.checksum_be = Checksum_engine(msg->bytes, msg->length);

    Message_shift(msg, RouteHeader_SIZE, NULL);

    struct RouteHeader* routeHeader = (struct RouteHeader*) msg->bytes;
    Bits_memset(routeHeader, 0, RouteHeader_SIZE);
    SwitchHeader_setVersion(&routeHeader->sh, SwitchHeader_CURRENT_VERSION);
    routeHeader->sh.label_be = Endian_hostToBigEndian64(label);
    routeHeader->flags |= RouteHeader_flags_CTRLMSG;

    return Iface_next(&ch->pub.coreIf, msg);
}
```
首先看这一段
```
else if (messageType_be == Control_PING_be) {
        Log_debug(ch->log, "got switch ping from [%s]", labelStr);
        if (ping->magic != Control_Ping_MAGIC) {
            Log_debug(ch->log, "DROP ping (bad magic)");
            return NULL;
        }
        ping->magic = Control_Pong_MAGIC;
        ctrl->header.type_be = Control_PONG_be;

    } 
```
可以看到，在收到ping之后，将ctrl的header.type_be设置为Control_PONG_be，也就是说，接下来，我们会发回一个pong
最后的return Iface_next(&ch->pub.coreIf, msg);是调用到了SwitchPinger.c中的messageFromControlHandler
### messageFromControlHandler
```
static Iface_DEFUN messageFromControlHandler(struct Message* msg, struct Iface* iface)
{
    struct SwitchPinger_pvt* ctx = Identity_check((struct SwitchPinger_pvt*) iface);
    struct RouteHeader rh;
    Message_pop(msg, &rh, RouteHeader_SIZE, NULL);
    ctx->incomingLabel = Endian_bigEndianToHost64(rh.sh.label_be);
    ctx->incomingVersion = 0;

    struct Control* ctrl = (struct Control*) msg->bytes;
    if (ctrl->header.type_be == Control_PONG_be) {
        Message_shift(msg, -Control_Header_SIZE, NULL);
        ctx->error = Error_NONE;
        if (msg->length >= Control_Pong_MIN_SIZE) {
            struct Control_Ping* pongHeader = (struct Control_Ping*) msg->bytes;
            ctx->incomingVersion = Endian_bigEndianToHost32(pongHeader->version_be);
            if (pongHeader->magic != Control_Pong_MAGIC) {
                Log_debug(ctx->logger, "dropped invalid switch pong");
                return NULL;
            }
            Message_shift(msg, -Control_Pong_HEADER_SIZE, NULL);
        } else {
            Log_debug(ctx->logger, "got runt pong message, length: [%d]", msg->length);
            return NULL;
        }

    } else if (ctrl->header.type_be == Control_KEYPONG_be) {
        Message_shift(msg, -Control_Header_SIZE, NULL);
        ctx->error = Error_NONE;
        if (msg->length >= Control_KeyPong_HEADER_SIZE && msg->length <= Control_KeyPong_MAX_SIZE) {
            struct Control_KeyPing* pongHeader = (struct Control_KeyPing*) msg->bytes;
            ctx->incomingVersion = Endian_bigEndianToHost32(pongHeader->version_be);
            if (pongHeader->magic != Control_KeyPong_MAGIC) {
                Log_debug(ctx->logger, "dropped invalid switch key-pong");
                return NULL;
            }
            Bits_memcpy(ctx->incomingKey, pongHeader->key, 32);
            Message_shift(msg, -Control_KeyPong_HEADER_SIZE, NULL);
        } else if (msg->length > Control_KeyPong_MAX_SIZE) {
            Log_debug(ctx->logger, "got overlong key-pong message, length: [%d]", msg->length);
            return NULL;
        } else {
            Log_debug(ctx->logger, "got runt key-pong message, length: [%d]", msg->length);
            return NULL;
        }

    } else if (ctrl->header.type_be == Control_ERROR_be) {
        Message_shift(msg, -Control_Header_SIZE, NULL);
        Assert_true((uint8_t*)&ctrl->content.error.errorType_be == msg->bytes);
        if (msg->length < (Control_Error_HEADER_SIZE + SwitchHeader_SIZE + Control_Header_SIZE)) {
            Log_debug(ctx->logger, "runt error packet");
            return NULL;
        }

        ctx->error = Message_pop32(msg, NULL);
        Message_push32(msg, 0, NULL);

        Message_shift(msg, -(Control_Error_HEADER_SIZE + SwitchHeader_SIZE), NULL);

        struct Control* origCtrl = (struct Control*) msg->bytes;

        Log_debug(ctx->logger, "error [%s] was caused by our [%s]",
                  Error_strerror(ctx->error),
                  Control_typeString(origCtrl->header.type_be));

        int shift;
        if (origCtrl->header.type_be == Control_PING_be) {
            shift = -(Control_Header_SIZE + Control_Ping_HEADER_SIZE);
        } else if (origCtrl->header.type_be == Control_KEYPING_be) {
            shift = -(Control_Header_SIZE + Control_KeyPing_HEADER_SIZE);
        } else {
            Assert_failure("problem in Ducttape.c");
        }
        if (msg->length < -shift) {
            Log_debug(ctx->logger, "runt error packet");
        }
        Message_shift(msg, shift, NULL);

    } else {
        // If it gets here then Ducttape.c is failing.
        Assert_true(false);
    }

    String* msgStr = &(String) { .bytes = (char*) msg->bytes, .len = msg->length };
    Pinger_pongReceived(msgStr, ctx->pinger);
    Bits_memset(ctx->incomingKey, 0, 32);
    return NULL;
}
```
查看这一段
```
if (ctrl->header.type_be == Control_PONG_be) {
        Message_shift(msg, -Control_Header_SIZE, NULL);
        ctx->error = Error_NONE;
        if (msg->length >= Control_Pong_MIN_SIZE) {
            struct Control_Ping* pongHeader = (struct Control_Ping*) msg->bytes;
            ctx->incomingVersion = Endian_bigEndianToHost32(pongHeader->version_be);
            if (pongHeader->magic != Control_Pong_MAGIC) {
                Log_debug(ctx->logger, "dropped invalid switch pong");
                return NULL;
            }
            Message_shift(msg, -Control_Pong_HEADER_SIZE, NULL);
        } else {
            Log_debug(ctx->logger, "got runt pong message, length: [%d]", msg->length);
            return NULL;
        }

    } 
```
这一段主要是针对要发pong包之前的处理
然后，
```
String* msgStr = &(String) { .bytes = (char*) msg->bytes, .len = msg->length };
    Pinger_pongReceived(msgStr, ctx->pinger);
```
首先放入了msgStr，然后调用Pinger_pongReceived(msgStr, ctx->pinger);把pong发出去。

### Pinger_pongReceived
在util/Pinger.c文件中
```
void Pinger_pongReceived(String* data, struct Pinger* pinger)
{
    if (data->len < 12) {
        Log_debug(pinger->logger, "Invalid ping response, too short");
        return;
    }
    uint32_t handle;
    Bits_memcpy(&handle, data->bytes, 4);
    int index = Map_OutstandingPings_indexForHandle(handle - pinger->baseHandle,
                                                    &pinger->outstandingPings);
    if (index < 0) {
        Log_debug(pinger->logger, "Invalid ping response handle [%u].", handle);
    } else {
        data->len -= 4;
        data->bytes += 4;
        uint64_t cookie;
        Bits_memcpy(&cookie, data->bytes, 8);
        struct Ping* p = Identity_check((struct Ping*) pinger->outstandingPings.values[index]);
        if (cookie != p->cookie) {
            Log_debug(pinger->logger, "Ping response with invalid cookie");
            return;
        }
        data->len -= 8;
        data->bytes += 8;
        callback(data, p);
    }
}
```
就看最后一句，
callback(data, p);
两个参数
1. data 发来的ping中携带的数据
2. p，这个p是一个Ping类型的对象，它来自于发送ping的那个peer，携带着那个peer的相关信息。

### callback(data,p)
```
static void callback(String* data, struct Ping* ping)
{
    uint32_t now = Time_currentTimeMilliseconds(ping->pinger->eventBase);
    ping->onResponse(data, now - ping->timeSent, ping->pub.context);

    // Flag the freePing function to tell it that the ping was not terminated by the user...
    ping->timeSent = 0;

    Allocator_free(ping->pub.pingAlloc);
}
```
最重要的一句
ping->onResponse(data, now - ping->timeSent, ping->pub.context);
这是调用了发送方提供给我们的onResponse函数。
查看Ping的结构体定义
```
struct Ping
{
    struct Pinger_Ping pub;
    struct Pinger* pinger;
    struct Timeout* timeout;
    String* data;
    int64_t timeSent;
    uint64_t cookie;
    Pinger_SEND_PING(sendPing);
    Pinger_ON_RESPONSE(onResponse);
    Identity
};
```
onResponse是一个Pinger_ON_RESPONSE(onResponse);再查看他的定义
```
#define Pinger_ON_RESPONSE(x) \
    void (* x)(String* data, uint32_t milliseconds, void* context)
```
上面已经提到过，这是SwitchPinger.c中实现的。同时，这也是发送方收到pong之后，做消息缺人的过程。在这个过程中，会核对一些信息，包括发出的ping中携带的data和收到的pong中携带的data是否相同。
### onPingResponse
```
static void onPingResponse(String* data, uint32_t milliseconds, void* vping)
{
    struct Ping* p = Identity_check((struct Ping*) vping);
    enum SwitchPinger_Result err = SwitchPinger_Result_OK;
    uint64_t label = p->context->incomingLabel;
    if (data) {
        if (label != p->label) {
            err = SwitchPinger_Result_LABEL_MISMATCH;
        } else if ((p->data || data->len > 0) && !String_equals(data, p->data)) {
            err = SwitchPinger_Result_WRONG_DATA;
        } else if (p->context->error == Error_LOOP_ROUTE) {
            err = SwitchPinger_Result_LOOP_ROUTE;
        } else if (p->context->error) {
            err = SwitchPinger_Result_ERROR_RESPONSE;
        }
    } else {
        err = SwitchPinger_Result_TIMEOUT;
    }

    uint32_t version = p->context->incomingVersion;
    struct SwitchPinger_Response* resp =
        Allocator_calloc(p->pub.pingAlloc, sizeof(struct SwitchPinger_Response), 1);
    resp->version = p->context->incomingVersion;
    resp->res = err;
    resp->label = label;
    resp->data = data;
    resp->milliseconds = milliseconds;
    resp->version = version;
    Bits_memcpy(resp->key, p->context->incomingKey, 32);
    resp->ping = &p->pub;
    p->onResponse(resp, p->pub.onResponseContext);
}
```
函数的最后，调用了p->onResponse(resp, p->pub.onResponseContext);
p是一个Ping* 类型的对象，查看Ping的定义
```
struct Ping
{
    struct SwitchPinger_Ping pub;
    uint64_t label;
    String* data;
    struct SwitchPinger_pvt* context;
    SwitchPinger_ResponseCallback onResponse;
    void* onResponseContext;
    struct Pinger_Ping* pingerPing;
    Identity
};
```
可见SwitchPinger_ResponseCallback onResponse;，查看SwitchPinger_ResponseCallback的定义
```
typedef void (* SwitchPinger_ResponseCallback)(struct SwitchPinger_Response* resp, void* userData);
```
这个函数的实现在InterfaceController.c中
### onPingResponse
```
static void onPingResponse(struct SwitchPinger_Response* resp, void* onResponseContext)
{
    if (SwitchPinger_Result_OK != resp->res) {
        return;
    }
    struct Peer* ep = Identity_check((struct Peer*) onResponseContext);
    struct InterfaceController_pvt* ic = Identity_check(ep->ici->ic);

    ep->addr.protocolVersion = resp->version;

    if (Defined(Log_DEBUG)) {
        String* addr = Address_toString(&ep->addr, resp->ping->pingAlloc);
        if (!Version_isCompatible(Version_CURRENT_PROTOCOL, resp->version)) {
            Log_debug(ic->logger, "got switch pong from node [%s] with incompatible version",
                                  addr->bytes);
        } else if (ep->addr.path != resp->label) {
            uint8_t sl[20];
            AddrTools_printPath(sl, resp->label);
            Log_debug(ic->logger, "got switch pong from node [%s] mismatch label [%s]",
                                  addr->bytes, sl);
        } else {
            Log_debug(ic->logger, "got switch pong from node [%s]", addr->bytes);
        }
    }

    if (!Version_isCompatible(Version_CURRENT_PROTOCOL, resp->version)) {
        return;
    }

    if (ep->state == InterfaceController_PeerState_ESTABLISHED) {
        sendPeer(0xffffffff, PFChan_Core_PEER, ep);
    }

    ep->timeOfLastPing = Time_currentTimeMilliseconds(ic->eventBase);

    if (Defined(Log_DEBUG)) {
        String* addr = Address_toString(&ep->addr, resp->ping->pingAlloc);
        Log_debug(ic->logger, "Received [%s] from lazy endpoint [%s]",
                  SwitchPinger_resultString(resp->res)->bytes, addr->bytes);
    }
}
```
至此，一个switch ping操作从InterfaceController.c出发，到对方处响应发回pong，回到发送方的InterfaceController.c中进行处理。