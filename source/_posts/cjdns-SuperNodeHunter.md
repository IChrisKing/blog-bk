---
title: cjdns源码分析--初探Supernode
category:
  - cjdns
  - cjdns源码分析
tags:
  - cjdns
  - cjdns源码分析
  - C/C++
description: “分析supernode（简称snode）在cjdns中的应用。包括普通点，snode，inbound各自如何获取supernode信息；如何通过supernode获取前往其他点的路径；涉及到的源码主要包括SupernodeHunter和部分Subnodeathfinder”
date: 2017-08-28 15:24:44
---

## supernode相关API ##
### Snodes系列 ###
这一系列有三个API：
* SupernodeHunter_addSnode
* SupernodeHunter_removeSnode
* SupernodeHunter_listSnodes

他们都对snp->authorizedSnodes进行操作，增加/删除/列出snp->authorizedSnodes中的地址。
目前，这三个API都是没有用到的。
如果在conf文件中配置了supernodes，则会调用到SupernodeHunter_addSnode，增加一个snode

authorizedSnodes的定义：
```
struct SupernodeHunter_pvt
{
    struct SupernodeHunter pub;

    /** Nodes which are authorized to be our supernode. */
    struct AddrSet* authorizedSnodes;

    /** Our peers, DO NOT TOUCH, changed from in SubnodePathfinder. */
    struct AddrSet* peers;

    // Number of the next peer to ping in the peers AddrSet
    int nextPeer;

    // Will be set to the best known supernode possibility
    struct Address snodeCandidate;

    bool snodePathUpdated;

    struct Allocator* alloc;

    struct Log* log;

    struct MsgCore* msgCore;

    struct EventBase* base;

    struct SwitchPinger* sp;

    struct Address* myAddress;
    String* selfAddrStr;

    Identity
};
```
```
struct AddrSet_pvt
{
    struct AddrSet pub;
    struct ArrayList_OfAddrs* addrList;
    struct Allocator* alloc;
    Identity
};
```
可见，authorizedSnodes最核心的内容，是一个ArrayList。

列出三个API的具体代码：
1. SupernodeHunter_addSnode
```
static void addSnode(Dict* args, void* vcontext, String* txid, struct Allocator* requestAlloc)
{
    struct Context* ctx = Identity_check((struct Context*) vcontext);
    struct Address* addr = getAddr(args, requestAlloc);
    if (!addr) {
        sendError(ctx, txid, requestAlloc, "parse_error");
        return;
    }
    int ret = SupernodeHunter_addSnode(ctx->snh, addr);
    char* err;
    switch (ret) {
        case SupernodeHunter_addSnode_EXISTS: {
            err = "SupernodeHunter_addSnode_EXISTS";
            break;
        }
        case 0: {
            err = "none";
            break;
        }
        default: {
            err = "UNKNOWN";
        }
    }
    sendError(ctx, txid, requestAlloc, err);
}
```
```
int SupernodeHunter_addSnode(struct SupernodeHunter* snh, struct Address* snodeAddr)
{
    struct SupernodeHunter_pvt* snp = Identity_check((struct SupernodeHunter_pvt*) snh);
    int length0 = snp->authorizedSnodes->length;
    AddrSet_add(snp->authorizedSnodes, snodeAddr);
    if (snp->authorizedSnodes->length == length0) {
        return SupernodeHunter_addSnode_EXISTS;
    }
    return 0;
}
```

2. SupernodeHunter_removeSnode
```
static void removeSnode(Dict* args, void* vcontext, String* txid, struct Allocator* requestAlloc)
{
    struct Context* ctx = Identity_check((struct Context*) vcontext);
    struct Address* addr = getAddr(args, requestAlloc);
    if (!addr) {
        sendError(ctx, txid, requestAlloc, "parse_error");
        return;
    }

    int ret = SupernodeHunter_removeSnode(ctx->snh, addr);
    char* err;
    switch (ret) {
        case SupernodeHunter_removeSnode_NONEXISTANT: {
            err = "SupernodeHunter_removeSnode_NONEXISTANT";
            break;
        }
        case 0: {
            err = "none";
            break;
        }
        default: {
            err = "UNKNOWN";
        }
    }
    sendError(ctx, txid, requestAlloc, err);
}
```
```
int SupernodeHunter_removeSnode(struct SupernodeHunter* snh, struct Address* toRemove)
{
    struct SupernodeHunter_pvt* snp = Identity_check((struct SupernodeHunter_pvt*) snh);
    int length0 = snp->authorizedSnodes->length;
    AddrSet_remove(snp->authorizedSnodes, toRemove);
    if (snp->authorizedSnodes->length == length0) {
        return SupernodeHunter_removeSnode_NONEXISTANT;
    }
    return 0;
}
```

3. SupernodeHunter_listSnodes
```
static void listSnodes(Dict* args, void* vcontext, String* txid, struct Allocator* requestAlloc)
{
    struct Context* ctx = Identity_check((struct Context*) vcontext);
    int page = 0;
    int64_t* pageP = Dict_getIntC(args, "page");
    if (pageP && *pageP > 0) { page = *pageP; }

    struct Address** snodes;
    int count = SupernodeHunter_listSnodes(ctx->snh, &snodes, requestAlloc);
    List* snodeList = List_new(requestAlloc);
    for (int i = page * NODES_PER_PAGE, j = 0; i < count && j < NODES_PER_PAGE; i++, j++) {
        List_addString(snodeList, Key_stringify(snodes[i]->key, requestAlloc), requestAlloc);
    }
    Dict* out = Dict_new(requestAlloc);
    Dict_putListC(out, "snodes", snodeList, requestAlloc);
    Dict_putStringCC(out, "error", "none", requestAlloc);
    Admin_sendMessage(out, txid, ctx->admin);
}
```
```
int SupernodeHunter_listSnodes(struct SupernodeHunter* snh,
                               struct Address*** outP,
                               struct Allocator* alloc)
{
    struct SupernodeHunter_pvt* snp = Identity_check((struct SupernodeHunter_pvt*) snh);
    struct Address** out = Allocator_calloc(alloc, sizeof(char*), snp->authorizedSnodes->length);
    for (int i = 0; i < snp->authorizedSnodes->length; i++) {
        out[i] = AddrSet_get(snp->authorizedSnodes, i);
    }
    *outP = out;
    return snp->authorizedSnodes->length;
}
```
当运行./cexec  "SupernodeHunter_listSnodes()"时，返回结果如下：
```
{
  "error": "none",
  "snodes": [],
  "txid": "2764470409"
}
```
在当前的系统中，snp->authorizedSnodes其实是空的。也就是说，普通点使用的snode，不是通过addSnode加进来的，也没有通过其他方法加入到snp->authorizedSnodes中。

### snode状态查看 ###
SupernodeHunter_status
```
static void status(Dict* args, void* vcontext, String* txid, struct Allocator* requestAlloc)
{
    struct Context* ctx = Identity_check((struct Context*) vcontext);
    char* activeSnode = "NONE";
    Dict* out = Dict_new(requestAlloc);
    if (ctx->snh->snodeIsReachable) {
        String* as = Address_toString(&ctx->snh->snodeAddr, requestAlloc);
        activeSnode = as->bytes;
    }
    Dict_putStringCC(out, "state",
        (ctx->snh->snodeIsReachable>0)? "REACHABLE":"UNREACHABLE", requestAlloc);
    Dict_putIntC(out, "usingAuthorizedSnode", ctx->snh->snodeIsReachable > 1, requestAlloc);
    Dict_putStringCC(out, "activeSnode", activeSnode, requestAlloc);
    Dict_putStringCC(out, "error", "none", requestAlloc);
    Admin_sendMessage(out, txid, ctx->admin);
}
```
当运行./cexec   "SupernodeHunter_status()"时，返回结果如下：
```
{
  "activeSnode": "v20.0000.0000.0000.0153.r3919swdqt8022xwf3dq8y4uwn8t87f38qsw7fkjzl7x6h358s20.k",
  "error": "none",
  "state": "REACHABLE",
  "txid": "589537837",
  "usingAuthorizedSnode": "0"
}
```
需要特别注意的是state字段和usingAuthorizedSnode字段，他们都是由ctx->snh->snodeIsReachable获得的，代码中有两个地方对他赋值：一个是从peer得到snode路径；一个是从snode处获得最佳路径。这两个得方的赋值方法都一样：
```
snp->pub.snodeIsReachable = (AddrSet_indexOf(snp->authorizedSnodes, src) != -1) ? 2 : 1;
```
snp->pub.snodeIsReachable的值取决于snp->authorizedSnodes中是否有src，src是snode的地址。前面提到过，snp->authorizedSnodes是空的，所以，snp->pub.snodeIsReachable的值是1.也就决定了state是REACHABLE，而usingAuthorizedSnode是0.

## 普通点的SuperNode启用流程 ##
### 1. Core_init ###
admin/angel/Core.c
```
    // The link between the Pathfinder and the core needs to be asynchronous.
    struct SubnodePathfinder* spf = SubnodePathfinder_new(
        alloc, logger, eventBase, rand, nc->myAddress, privateKey, encodingScheme, notification);
    struct ASynchronizer* spfAsync = ASynchronizer_new(alloc, eventBase, logger);
    Iface_plumb(&spfAsync->ifA, &spf->eventIf);
    EventEmitter_regPathfinderIface(nc->ee, &spfAsync->ifB);

    #ifndef SUBNODE
        struct Pathfinder* opf = Pathfinder_register(alloc, logger, eventBase, rand, admin);
        struct ASynchronizer* opfAsync = ASynchronizer_new(alloc, eventBase, logger);
        Iface_plumb(&opfAsync->ifA, &opf->eventIf);
        EventEmitter_regPathfinderIface(nc->ee, &opfAsync->ifB);
    #endif

    SubnodePathfinder_start(spf);
```

### 2. SubnodePathfinder_start() ###
subnode/SubnodePathfinder.c
```
void SubnodePathfinder_start(struct SubnodePathfinder* sp)
{
    struct SubnodePathfinder_pvt* pf = Identity_check((struct SubnodePathfinder_pvt*) sp);
    pf->msgCore = MsgCore_new(pf->base, pf->rand, pf->alloc, pf->log, pf->myScheme);
    Iface_plumb(&pf->msgCoreIf, &pf->msgCore->interRouterIf);

    PingResponder_new(pf->alloc, pf->log, pf->msgCore, pf->br);

    GetPeersResponder_new(
        pf->alloc, pf->log, pf->myPeers, pf->myAddress, pf->msgCore, pf->br, pf->myScheme);

    pf->pub.snh = SupernodeHunter_new(
        pf->alloc, pf->log, pf->base, pf->sp, pf->myPeers, pf->msgCore,
                            pf->myAddress, pf->notification);

    pf->ra = ReachabilityAnnouncer_new(
        pf->alloc, pf->log, pf->base, pf->rand, pf->msgCore, pf->pub.snh, pf->privateKey,
            pf->myScheme);

    pf->pub.rc = ReachabilityCollector_new(
        pf->alloc, pf->msgCore, pf->log, pf->base, pf->br, pf->myAddress);

    pf->pub.rc->userData = pf;
    pf->pub.rc->onChange = rcChange;

    struct PFChan_Pathfinder_Connect conn = {
        .superiority_be = Endian_hostToBigEndian32(1),
        .version_be = Endian_hostToBigEndian32(Version_CURRENT_PROTOCOL)
    };
    CString_strncpy(conn.userAgent, "Anet subnode pathfinder", 64);
    sendEvent(pf, PFChan_Pathfinder_CONNECT, &conn, PFChan_Pathfinder_Connect_SIZE);
}
```
### 3. SupernodeHunter_new ###
```
struct SupernodeHunter* SupernodeHunter_new(struct Allocator* allocator,
                                            struct Log* log,
                                            struct EventBase* base,
                                            struct SwitchPinger* sp,
                                            struct AddrSet* peers,
                                            struct MsgCore* msgCore,
                                            struct Address* myAddress,
                                            struct Notification* notification)
{
    struct Allocator* alloc = Allocator_child(allocator);
    struct SupernodeHunter_pvt* out =
        Allocator_calloc(alloc, sizeof(struct SupernodeHunter_pvt), 1);
    Identity_set(out);
    out->authorizedSnodes = AddrSet_new(alloc);
    out->peers = peers;
    out->base = base;
    out->pub.notification = notification;
    //out->rand = rand;
    //out->nodes = AddrSet_new(alloc);
    //out->timeSnodeCalled = Time_currentTimeMilliseconds(base);
    //out->snodeCandidates = AddrSet_new(alloc);
    out->log = log;
    out->alloc = alloc;
    out->msgCore = msgCore;
    out->myAddress = myAddress;
    out->selfAddrStr = String_newBinary(myAddress->ip6.bytes, 16, alloc);
    out->sp = sp;
    out->snodePathUpdated = false;
    out->pub.onSnodeUnreachable = onSnodeUnreachable;
    Timeout_setInterval(probePeerCycle, out, CYCLE_MS, base, alloc);
    return &out->pub;
}
```

### 4. probePeerCycle ###
```

static void probePeerCycle(void* vsn)
{
    struct SupernodeHunter_pvt* snp = Identity_check((struct SupernodeHunter_pvt*) vsn);

    if (snp->pub.snodeIsReachable && !snp->snodePathUpdated) {
        updateSnodePath(snp);
    }

    if (snp->pub.snodeIsReachable > 1) { return; }
    if (snp->pub.snodeIsReachable && !snp->authorizedSnodes->length) { return; }
    if (!snp->peers->length) { return; }

    //Log_debug(snp->log, "probePeerCycle()");

    if (AddrSet_indexOf(snp->authorizedSnodes, snp->myAddress) != -1) {
        Log_info(snp->log, "Self is specified as supernode, pinging...");
        adoptSupernode(snp, snp->myAddress);
        return;
    }

    struct Address* peer = getPeerByNpn(snp, snp->nextPeer);
    if (!peer) {
        Log_info(snp->log, "No peer found who is version >= 20");
        return;
    }
    snp->nextPeer++;

    struct SwitchPinger_Ping* p =
        SwitchPinger_newPing(peer->path, String_CONST(""), 3000, peerResponse, snp->alloc, snp->sp);
    Assert_true(p);

    p->type = SwitchPinger_Type_GETSNODE;
    if (snp->pub.snodeIsReachable) {
        Bits_memcpy(&p->snode, &snp->pub.snodeAddr, sizeof(struct Address));
    }

    p->onResponseContext = snp;
}
```
当首次调用到这里，snp->pub.snodeIsReachable为false，snp->snodePathUpdate为false，snp->authorizedSnodes为空。
直接到这里开始执行，首先遍历自己的peer，依次问每个peer，找到version大于等于20的peer
```
    struct Address* peer = getPeerByNpn(snp, snp->nextPeer);
    if (!peer) {
        Log_info(snp->log, "No peer found who is version >= 20");
        return;
    }
    snp->nextPeer++;
```
```
static struct Address* getPeerByNpn(struct SupernodeHunter_pvt* snp, int npn)
{
    npn = npn % snp->peers->length;
    int i = npn;
    do {
        struct Address* peer = AddrSet_get(snp->peers, i);
        if (peer && peer->protocolVersion > 19) { return peer; }
        i = (i + 1) % snp->peers->length;
    } while (i != npn);
    return NULL;
}
```
然后问这个peer，snode在哪里
```
    struct SwitchPinger_Ping* p =
        SwitchPinger_newPing(peer->path, String_CONST(""), 3000, peerResponse, snp->alloc, snp->sp);
    Assert_true(p);

    p->type = SwitchPinger_Type_GETSNODE;
    if (snp->pub.snodeIsReachable) {
        Bits_memcpy(&p->snode, &snp->pub.snodeAddr, sizeof(struct Address));
    }

    p->onResponseContext = snp;
```
发送了一个SwitchPinger_Ping，type是SwitchPinger_Type_GETSNODE.回调函数是peerResponse
```
static void peerResponse(struct SwitchPinger_Response* resp, void* userData)
{
    struct SupernodeHunter_pvt* snp = Identity_check((struct SupernodeHunter_pvt*) userData);
    char* err = "";
    switch (resp->res) {
        case SwitchPinger_Result_OK: peerResponseOK(resp, snp); return;
        case SwitchPinger_Result_LABEL_MISMATCH: err = "LABEL_MISMATCH"; break;
        case SwitchPinger_Result_WRONG_DATA: err = "WRONG_DATA"; break;
        case SwitchPinger_Result_ERROR_RESPONSE: err = "ERROR_RESPONSE"; break;
        case SwitchPinger_Result_LOOP_ROUTE: err = "LOOP_ROUTE"; break;
        case SwitchPinger_Result_TIMEOUT: err = "TIMEOUT"; break;
        default: err = "unknown error"; break;
    }
    Log_debug(snp->log, "Error sending snp query to peer [%s]", err);
}
```
正常情况下，应该收到SwitchPinger_Result_OK，调用peerResponseOK
```
static void peerResponseOK(struct SwitchPinger_Response* resp, struct SupernodeHunter_pvt* snp)
{
    struct Address snode;
    Bits_memcpy(&snode, &resp->snode, sizeof(struct Address));
    if (!snode.path) {
        uint8_t label[20];
        AddrTools_printPath(label, resp->label);
        Log_debug(snp->log, "Peer [%s] reports no supernode", label);
        return;
    }

    uint64_t path = LabelSplicer_splice(snode.path, resp->label);
    if (path == UINT64_MAX) {
        Log_debug(snp->log, "Supernode path could not be spliced");
        return;
    }
    snode.path = path;

    struct Address* firstPeer = getPeerByNpn(snp, 0);
    if (!firstPeer) {
        Log_info(snp->log, "All peers have gone away while packet was outstanding");
        return;
    }

    // 1.
    // If we have looped around and queried all of our peers returning to the first and we have
    // still not found an snode in our authorized snodes list, we should simply accept this one.
    if (!snp->pub.snodeIsReachable && snp->nextPeer > 1 && firstPeer->path == resp->label) {
        if (!snp->snodeCandidate.path) {
            Log_info(snp->log, "No snode candidate found");
            snp->nextPeer = 0;
            return;
        }
        if (Bits_memcmp(&snp->snodeCandidate, &snp->pub.snodeAddr, Address_SIZE)) {
            Log_info(snp->log, "SnodeAddr != snodeCandidate");
            snp->nextPeer = 0;
            return;
        }
        adoptSupernode(snp, &snp->snodeCandidate);
        return;
    }

    // 2.
    // If this snode is one of our authorized snodes OR if we have none defined, accept this one.
    if (!snp->authorizedSnodes->length || AddrSet_indexOf(snp->authorizedSnodes, &snode) > -1) {
        Address_getPrefix(&snode);
        adoptSupernode(snp, &snode);
        return;
    }

    if (!snp->snodeCandidate.path) {
        Bits_memcpy(&snp->snodeCandidate, &snode, sizeof(struct Address));
        Address_getPrefix(&snp->snodeCandidate);
    }
}
```
这里分了两种情况来添加snode，对于第一次调用到此处的普通点，它会将peer告诉他的snode地址设置成自己的snode，调用adoptSupernode，试图和snode联系
```
static void adoptSupernode(struct SupernodeHunter_pvt* snp, struct Address* candidate)
{
    struct MsgCore_Promise* qp = MsgCore_createQuery(snp->msgCore, 0, snp->alloc);
    struct Query* q = Allocator_calloc(qp->alloc, sizeof(struct Query), 1);
    Identity_set(q);
    q->snp = snp;
    q->sendTime = Time_currentTimeMilliseconds(snp->base);

    Dict* msg = qp->msg = Dict_new(qp->alloc);
    qp->cb = adoptSupernode2;
    qp->userData = q;
    qp->target = Address_clone(candidate, qp->alloc);

    Log_debug(snp->log, "Pinging snode [%s]", Address_toString(qp->target, qp->alloc)->bytes);
    Dict_putStringCC(msg, "sq", "pn", qp->alloc);

    Assert_true(AddressCalc_validAddress(candidate->ip6.bytes));
    return;
}
```
回调函数是adoptSupernode2
```
static void adoptSupernode2(Dict* msg, struct Address* src, struct MsgCore_Promise* prom)
{
    struct Query* q = Identity_check((struct Query*) prom->userData);
    struct SupernodeHunter_pvt* snp = Identity_check(q->snp);

    if (!src) {
        String* addrStr = Address_toString(prom->target, prom->alloc);
        Log_debug(snp->log, "timeout sending to %s", addrStr->bytes);
        return;
    }
    String* addrStr = Address_toString(src, prom->alloc);
    Log_debug(snp->log, "Reply from %s", addrStr->bytes);

    int64_t* snodeRecvTime = Dict_getIntC(msg, "recvTime");
    if (!snodeRecvTime) {
        Log_info(snp->log, "getRoute reply with no timeStamp, bad snode");
        return;
    }
    Log_debug(snp->log, "\n\nSupernode location confirmed [%s]\n\n",
        Address_toString(src, prom->alloc)->bytes);
    if (snp->pub.snodeIsReachable) {
        // If while we were searching, the outside code declared that indeed the snode
        // is reachable, we will not try to change their snode.
    } else if (snp->pub.onSnodeChange) {
        Bits_memcpy(&snp->pub.snodeAddr, src, Address_SIZE);
        snp->pub.snodeIsReachable = (AddrSet_indexOf(snp->authorizedSnodes, src) != -1) ? 2 : 1;
        snp->pub.onSnodeChange(&snp->pub, q->sendTime, *snodeRecvTime);
        Notification_doNotify(snp->pub.notification, SNODE_REACHABLE,REACHABLE);
    } else {
        Log_warn(snp->log, "onSnodeChange is not set");
    }
}
```
主要操作是：将snode的地址设置到snp->pub.snodeAddr，更新snp->pub.snodeIsReachable的值，调用snp->pub.onSnodeChange
```
static void onSnodeChange(struct SupernodeHunter* sh,
                          int64_t sendTime,
                          int64_t snodeRecvTime)
{
    struct ReachabilityAnnouncer_pvt* rap =
        Identity_check((struct ReachabilityAnnouncer_pvt*) sh->userData);
    int64_t clockSkew = estimateClockSkew(sendTime, snodeRecvTime, ourTime(rap));
    uint64_t clockSkewDiff = (clockSkew - rap->clockSkew) & ~(((uint64_t)1)<<63);
    // If the node is the same and the clock skew difference is less than 10 seconds,
    // just change path and continue.
    if (!Bits_memcmp(rap->snode.key, sh->snodeAddr.key, 32) && clockSkewDiff < 5000) {
        Log_debug(rap->log, "Change Supernode (path only)");
        Bits_memcpy(&rap->snode, &sh->snodeAddr, Address_SIZE);
        return;
    }
    Log_debug(rap->log, "Change Supernode");
    Bits_memcpy(&rap->snode, &sh->snodeAddr, Address_SIZE);
    rap->clockSkew = estimateClockSkew(sendTime, snodeRecvTime, ourTime(rap));
    stateReset(rap);
}
```
做了一些地址设置的操作，暂时不继续往下分析。
至此，普通点向peer询问snode地址，并与snode建立联系的过程分析完成。但这并没有结束，接下来，普通点还会向snode询问最佳路径。
回到`SupernodeHunter_new`中再看一次：
```
struct SupernodeHunter* SupernodeHunter_new(struct Allocator* allocator,
                                            struct Log* log,
                                            struct EventBase* base,
                                            struct SwitchPinger* sp,
                                            struct AddrSet* peers,
                                            struct MsgCore* msgCore,
                                            struct Address* myAddress,
                                            struct Notification* notification)
{
    struct Allocator* alloc = Allocator_child(allocator);
    struct SupernodeHunter_pvt* out =
        Allocator_calloc(alloc, sizeof(struct SupernodeHunter_pvt), 1);
    Identity_set(out);
    out->authorizedSnodes = AddrSet_new(alloc);
    out->peers = peers;
    out->base = base;
    out->pub.notification = notification;
    //out->rand = rand;
    //out->nodes = AddrSet_new(alloc);
    //out->timeSnodeCalled = Time_currentTimeMilliseconds(base);
    //out->snodeCandidates = AddrSet_new(alloc);
    out->log = log;
    out->alloc = alloc;
    out->msgCore = msgCore;
    out->myAddress = myAddress;
    out->selfAddrStr = String_newBinary(myAddress->ip6.bytes, 16, alloc);
    out->sp = sp;
    out->snodePathUpdated = false;
    out->pub.onSnodeUnreachable = onSnodeUnreachable;
    Timeout_setInterval(probePeerCycle, out, CYCLE_MS, base, alloc);
    return &out->pub;
```
可以看到，对probePeerCycle的调用，是一个周期性的行为，每CYCLE_MS时间调用一次，继续进入到probePeerCycle中：
```
static void probePeerCycle(void* vsn)
{
    struct SupernodeHunter_pvt* snp = Identity_check((struct SupernodeHunter_pvt*) vsn);

    if (snp->pub.snodeIsReachable && !snp->snodePathUpdated) {
        updateSnodePath(snp);
    }

    if (snp->pub.snodeIsReachable > 1) { return; }
    if (snp->pub.snodeIsReachable && !snp->authorizedSnodes->length) { return; }
    if (!snp->peers->length) { return; }

    //Log_debug(snp->log, "probePeerCycle()");

    if (AddrSet_indexOf(snp->authorizedSnodes, snp->myAddress) != -1) {
        Log_info(snp->log, "Self is specified as supernode, pinging...");
        adoptSupernode(snp, snp->myAddress);
        return;
    }

    struct Address* peer = getPeerByNpn(snp, snp->nextPeer);
    if (!peer) {
        Log_info(snp->log, "No peer found who is version >= 20");
        return;
    }
    snp->nextPeer++;

    struct SwitchPinger_Ping* p =
        SwitchPinger_newPing(peer->path, String_CONST(""), 3000, peerResponse, snp->alloc, snp->sp);
    Assert_true(p);

    p->type = SwitchPinger_Type_GETSNODE;
    if (snp->pub.snodeIsReachable) {
        Bits_memcpy(&p->snode, &snp->pub.snodeAddr, sizeof(struct Address));
    }

    p->onResponseContext = snp;
}
```
这一次，snp->pub.snodeIsReachable已经为true，调用updateSnodePath
```
static void updateSnodePath(struct SupernodeHunter_pvt* snp)
{
    struct MsgCore_Promise* qp = MsgCore_createQuery(snp->msgCore, 0, snp->alloc);
    struct Query* q = Allocator_calloc(qp->alloc, sizeof(struct Query), 1);
    Identity_set(q);
    q->snp = snp;
    q->sendTime = Time_currentTimeMilliseconds(snp->base);

    Dict* msg = qp->msg = Dict_new(qp->alloc);
    qp->cb = updateSnodePath2;
    qp->userData = q;
    qp->target = Address_clone(&snp->pub.snodeAddr, qp->alloc);;

    Log_debug(snp->log, "Update snode [%s] path", Address_toString(qp->target, qp->alloc)->bytes);
    Dict_putStringCC(msg, "sq", "gr", qp->alloc);
    String* src = String_newBinary(snp->myAddress->ip6.bytes, 16, qp->alloc);
    Dict_putStringC(msg, "src", src, qp->alloc);
    String* target = String_newBinary(snp->pub.snodeAddr.ip6.bytes, 16, qp->alloc);
    Dict_putStringC(msg, "tar", target, qp->alloc);
}
```
回调函数为updateSnodePath2
```
static void updateSnodePath2(Dict* msg, struct Address* src, struct MsgCore_Promise* prom)
{
    struct Query* q = Identity_check((struct Query*) prom->userData);
    struct SupernodeHunter_pvt* snp = Identity_check(q->snp);

    if (!src) {
        String* addrStr = Address_toString(prom->target, prom->alloc);
        Log_debug(snp->log, "timeout sending to %s", addrStr->bytes);
        return;
    }
    int64_t* snodeRecvTime = Dict_getIntC(msg, "recvTime");
    if (!snodeRecvTime) {
        Log_info(snp->log, "getRoute reply with no timeStamp, bad snode");
        return;
    }
    struct Address_List* al = ReplySerializer_parse(src, msg, snp->log, false, prom->alloc);
    if (!al || al->length == 0) { return; }
    Log_debug(snp->log, "Supernode path updated with[%s]",
                        Address_toString(&al->elems[0], prom->alloc)->bytes);

    snp->snodePathUpdated = true;
    if (!Bits_memcmp(&snp->pub.snodeAddr, &al->elems[0], Address_SIZE)) {
        return;
    }
    Bits_memcpy(&snp->pub.snodeAddr, &al->elems[0], Address_SIZE);
    Bits_memcpy(&snp->snodeCandidate, &al->elems[0], Address_SIZE);
    if (snp->pub.onSnodeChange) {
        snp->pub.snodeIsReachable = (AddrSet_indexOf(snp->authorizedSnodes, src) != -1) ? 2 : 1;
        snp->pub.onSnodeChange(&snp->pub, q->sendTime, *snodeRecvTime);
        Notification_doNotify(snp->pub.notification, SNODE_REACHABLE,REACHABLE);
    }
}
```
主要操作是将path设置到snp->pub.snodeAddr和snp->snodeCandidate，将 snp->snodePathUpdated设为true。
这样，下次调用到probePeerCycle时，会进入到
```
    if (snp->pub.snodeIsReachable && !snp->authorizedSnodes->length) { return; }
```
直接返回。这样，只要snodeIsReachable一直为true，就会一直返回。

## snode的SuperNode启动过程 ##
### conf文件分析 ###
snode的conf文件：
```
"interfaces":{
	"UDPInterface":[
    	{
        	"bind":"0.0.0.0:34435",
            "connectTo":{}
        }
    ],
    ...
},
"router":{
	"supernodes":[
    	"r3919swdqt8022xwf3dq8y4uwn8t87f38qsw7fkjzl7x6h358s20.k"
    ],
    ...
    "ipTunnel":{
    	"allowedConnections":[],
        "outgoingConnections":[]
    }
}
```
对比起来，普通点的conf文件：
```
"interfaces":{
	"UDPInterface":[
		{
			"bind":"0.0.0.0:26808",
			"connectTo":{
				"106.75.59.53:50001":{
					"password":"XRuMWpgvNienefc7ZT8gXTuTCvSWWSA",
					"publicKey":"nsn1nz93lztkg6zbw4yqnr6zlzxhchppzdkduwh9wh79p88fwx60.k"
				}
			}
		}
	],
	...
},
"router":{
	...
	"ipTunnel":{
		"allowedConnections":[],
		"outgoingConnections":[
			"wfzyzrc0q4g83y0dgzxx1l862u0lscucj75yw9q1ymbltzwh2fq0.k"
		]
	}
},
```
snode的conf文件的不同主要体现在下面几点：
1. interfaces的UDPInterface的connectTo为空，说明snode并不主动连接到任何点。
2. route的ipTunnel的outgoingConnetctions为空，说明snode并不将任何点设为离岸点。
3. 在route中设置有snode，值为自己的pubkey。
关于上面两条，本文不做具体分析，重点关注与snode相关的内容，即第三条。

当conf文件中配置有snode时，首先的影响就是会调用到增加snode的API接口：`SupernodeHunter_addSnode`。上面分析过，这个接口的作用是增加snode到snp->authorizedSnodes中。仅仅增加地址到snp->authorizedSnodes中，还远未完成整个snode的添加过程。接下来，分析snode是怎么把自己设置为snode的。

### snode的Super Node设置 ###
snode的启动过程与普通点一样，不同点是从probePeerCycle方法中开始的，再次查看这个方法：
```
static void probePeerCycle(void* vsn)
{
    struct SupernodeHunter_pvt* snp = Identity_check((struct SupernodeHunter_pvt*) vsn);

    if (snp->pub.snodeIsReachable && !snp->snodePathUpdated) {
        updateSnodePath(snp);
    }

    if (snp->pub.snodeIsReachable > 1) { return; }
    if (snp->pub.snodeIsReachable && !snp->authorizedSnodes->length) { return; }
    if (!snp->peers->length) { return; }

    //Log_debug(snp->log, "probePeerCycle()");

    if (AddrSet_indexOf(snp->authorizedSnodes, snp->myAddress) != -1) {
        Log_info(snp->log, "Self is specified as supernode, pinging...");
        adoptSupernode(snp, snp->myAddress);
        return;
    }

    struct Address* peer = getPeerByNpn(snp, snp->nextPeer);
    if (!peer) {
        Log_info(snp->log, "No peer found who is version >= 20");
        return;
    }
    snp->nextPeer++;

    struct SwitchPinger_Ping* p =
        SwitchPinger_newPing(peer->path, String_CONST(""), 3000, peerResponse, snp->alloc, snp->sp);
    Assert_true(p);

    p->type = SwitchPinger_Type_GETSNODE;
    if (snp->pub.snodeIsReachable) {
        Bits_memcpy(&p->snode, &snp->pub.snodeAddr, sizeof(struct Address));
    }

    p->onResponseContext = snp;
}
```
当首次调用到这里，snp->pub.snodeIsReachable为false，snp->snodePathUpdate为false，snp->authorizedSnodes中是从conf中添加进来的snode，也就是自己。所以，会执行
```
    if (AddrSet_indexOf(snp->authorizedSnodes, snp->myAddress) != -1) {
        Log_info(snp->log, "Self is specified as supernode, pinging...");
        adoptSupernode(snp, snp->myAddress);
        return;
    }
```
将自己设置为snode，具体设置过程不再分析。
看一下API调用结果：
```
[root:sbu-snode]# ./cexec   "SupernodeHunter_status()"
{
  "activeSnode": "v20.0000.0000.0000.0001.r3919swdqt8022xwf3dq8y4uwn8t87f38qsw7fkjzl7x6h358s20.k",
  "error": "none",
  "state": "REACHABLE",
  "txid": "2038609306",
  "usingAuthorizedSnode": "1"
}
```
不同点在于usingAuthorizedSnode为1

```
[root:sbu-snode]# ./cexec   "SupernodeHunter_listSnodes(0)"
{
  "error": "none",
  "snodes": [
    "r3919swdqt8022xwf3dq8y4uwn8t87f38qsw7fkjzl7x6h358s20.k"
  ],
  "txid": "3762445565"
}
```
有返回，不为空。

## inbound的Super Node启动过程 ##
### conf文件分析 ###
以inbound上的sbu-gate1为例：
```
"interfaces":{
	"UDPInterface":[
    	{
        	"connectTo":{
            	"47.92.135.33:34435":{
					"password":"0Jsal98j1Mbv1WFVVzlzB33of4J910C",
                    "publicKey":"r3919swdqt8022xwf3dq8y4uwn8t87f38qsw7fkjzl7x6h358s20.k"
				},
                "172.17.0.1:50002":{
                	"password":"q3lAM4EvrYsYDGQynkIeD1rbDjBtoc8",
                    "publicKey":"5zl6xndspf1sm2042987rthn8pcgxzf74rfvc5njl2gh6s9g8780.k"
                }
            },
            "bind":"0.0.0.0:50001"
        }
    ],
    ...
},
"router":{
	"supernodes":[
    	"r3919swdqt8022xwf3dq8y4uwn8t87f38qsw7fkjzl7x6h358s20.k"
    ],
    ...
}
```
1. interfaces的UDPInterface的connectTo中，设置了两个点，一个是snode，一个是sbu-gate2，也就是同时将snode和sbu-gate2设为peer。
2. route的supernodes中设置了snode。
conf文件中配置有snode，所以会调用到增加snode的API接口：`SupernodeHunter_addSnode`。增加snode到snp->authorizedSnodes中。

### inbound的Super Node设置 ###
直接从probePeerCycle开始分析
```
static void probePeerCycle(void* vsn)
{
    struct SupernodeHunter_pvt* snp = Identity_check((struct SupernodeHunter_pvt*) vsn);

    if (snp->pub.snodeIsReachable && !snp->snodePathUpdated) {
        updateSnodePath(snp);
    }

    if (snp->pub.snodeIsReachable > 1) { return; }
    if (snp->pub.snodeIsReachable && !snp->authorizedSnodes->length) { return; }
    if (!snp->peers->length) { return; }

    //Log_debug(snp->log, "probePeerCycle()");

    if (AddrSet_indexOf(snp->authorizedSnodes, snp->myAddress) != -1) {
        Log_info(snp->log, "Self is specified as supernode, pinging...");
        adoptSupernode(snp, snp->myAddress);
        return;
    }

    struct Address* peer = getPeerByNpn(snp, snp->nextPeer);
    if (!peer) {
        Log_info(snp->log, "No peer found who is version >= 20");
        return;
    }
    snp->nextPeer++;

    struct SwitchPinger_Ping* p =
        SwitchPinger_newPing(peer->path, String_CONST(""), 3000, peerResponse, snp->alloc, snp->sp);
    Assert_true(p);

    p->type = SwitchPinger_Type_GETSNODE;
    if (snp->pub.snodeIsReachable) {
        Bits_memcpy(&p->snode, &snp->pub.snodeAddr, sizeof(struct Address));
    }

    p->onResponseContext = snp;
}
```
当首次调用到这里，snp->pub.snodeIsReachable为false，snp->snodePathUpdate为false，snp->authorizedSnodes中是从conf中添加进来的snode，也就是snode点。所以，会执行
```
    struct Address* peer = getPeerByNpn(snp, snp->nextPeer);
    if (!peer) {
        Log_info(snp->log, "No peer found who is version >= 20");
        return;
    }
    snp->nextPeer++;

    struct SwitchPinger_Ping* p =
        SwitchPinger_newPing(peer->path, String_CONST(""), 3000, peerResponse, snp->alloc, snp->sp);
    Assert_true(p);

    p->type = SwitchPinger_Type_GETSNODE;
    if (snp->pub.snodeIsReachable) {
        Bits_memcpy(&p->snode, &snp->pub.snodeAddr, sizeof(struct Address));
    }

    p->onResponseContext = snp;
```
和普通点一样，询问自己的peer，进而找到前往snode的路径，与snode建立连接后，更新前往snode的路径。
但注意，inbound的两个peer，其中有一个就是snode。
查看API调用的结果：
```
root@sbu-gate1:/ancode/new# ./cexec   "SupernodeHunter_status()"
{
  "activeSnode": "v20.0000.0000.0000.0015.r3919swdqt8022xwf3dq8y4uwn8t87f38qsw7fkjzl7x6h358s20.k",
  "error": "none",
  "state": "REACHABLE",
  "txid": "1834461195",
  "usingAuthorizedSnode": "1"
}
```
usingAuthorizedSnode值为1
```
root@sbu-gate4:/ancode/new# ./cexec "SupernodeHunter_listSnodes(0)"
{
  "error": "none",
  "snodes": [
    "r3919swdqt8022xwf3dq8y4uwn8t87f38qsw7fkjzl7x6h358s20.k"
  ],
  "txid": "2813207163"
}
```
listSnode中有conf中配置的snode。

## GETSNODE请求的发起和peer的回应 ##
### GETSNODE请求的发送 ###
之前在分析snode的启动流程时讲到，普通点和inbound在寻找snode的路径时，采用的方法是，向自己的peer询问。即向每个peer发送一个p->type为SwitchPinger_Type_GETSNODE的SwitchPinger_Ping。现在来具体看一下这个请求的发送过程：
```
    struct SwitchPinger_Ping* p =
        SwitchPinger_newPing(peer->path, String_CONST(""), 3000, peerResponse, snp->alloc, snp->sp);
    Assert_true(p);

    p->type = SwitchPinger_Type_GETSNODE;
```
调用到SwitchPinger_newPing
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
调用sendPing
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

    if (p->pub.type == SwitchPinger_Type_KEYPING) {
        Message_push(msg, NULL, Control_KeyPing_HEADER_SIZE, NULL);
        struct Control_KeyPing* keyPingHeader = (struct Control_KeyPing*) msg->bytes;
        keyPingHeader->magic = Control_KeyPing_MAGIC;
        keyPingHeader->version_be = Endian_hostToBigEndian32(Version_CURRENT_PROTOCOL);
        Bits_memcpy(keyPingHeader->key, p->context->myAddr->key, 32);
    } else if (p->pub.type == SwitchPinger_Type_PING) {
        Message_push(msg, NULL, Control_Ping_HEADER_SIZE, NULL);
        struct Control_Ping* pingHeader = (struct Control_Ping*) msg->bytes;
        pingHeader->magic = Control_Ping_MAGIC;
        pingHeader->version_be = Endian_hostToBigEndian32(Version_CURRENT_PROTOCOL);
    } else if (p->pub.type == SwitchPinger_Type_GETSNODE) {
        Message_push(msg, NULL, Control_GetSnode_HEADER_SIZE, NULL);
        struct Control_GetSnode* hdr = (struct Control_GetSnode*) msg->bytes;
        hdr->magic = Control_GETSNODE_QUERY_MAGIC;
        hdr->version_be = Endian_hostToBigEndian32(Version_CURRENT_PROTOCOL);
        hdr->kbps_be = Endian_hostToBigEndian32(p->pub.kbpsLimit);
        Bits_memcpy(hdr->snodeKey, p->pub.snode.key, 32);
        uint64_t pathToSnode_be = Endian_hostToBigEndian64(p->pub.snode.path);
        Bits_memcpy(hdr->pathToSnode_be, &pathToSnode_be, 8);
        hdr->snodeVersion_be = Endian_hostToBigEndian32(p->pub.snode.protocolVersion);
    } else {
        Assert_failure("unexpected ping type");
    }

    Message_shift(msg, Control_Header_SIZE, NULL);
    struct Control* ctrl = (struct Control*) msg->bytes;
    ctrl->header.checksum_be = 0;
    if (p->pub.type == SwitchPinger_Type_PING) {
        ctrl->header.type_be = Control_PING_be;
    } else if (p->pub.type == SwitchPinger_Type_KEYPING) {
        ctrl->header.type_be = Control_KEYPING_be;
    } else if (p->pub.type == SwitchPinger_Type_GETSNODE) {
        ctrl->header.type_be = Control_GETSNODE_QUERY_be;
    } else {
        Assert_failure("unexpected type");
    }
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
主要的操作包括两部分：
```
else if (p->pub.type == SwitchPinger_Type_GETSNODE) {
        Message_push(msg, NULL, Control_GetSnode_HEADER_SIZE, NULL);
        struct Control_GetSnode* hdr = (struct Control_GetSnode*) msg->bytes;
        hdr->magic = Control_GETSNODE_QUERY_MAGIC;
        hdr->version_be = Endian_hostToBigEndian32(Version_CURRENT_PROTOCOL);
        hdr->kbps_be = Endian_hostToBigEndian32(p->pub.kbpsLimit);
        Bits_memcpy(hdr->snodeKey, p->pub.snode.key, 32);
        uint64_t pathToSnode_be = Endian_hostToBigEndian64(p->pub.snode.path);
        Bits_memcpy(hdr->pathToSnode_be, &pathToSnode_be, 8);
        hdr->snodeVersion_be = Endian_hostToBigEndian32(p->pub.snode.protocolVersion);
    }
```
```
else if (p->pub.type == SwitchPinger_Type_GETSNODE) {
        ctrl->header.type_be = Control_GETSNODE_QUERY_be;
    }
```
主要是一些字段内容的设置，特别注意到`
ctrl->header.type_be = Control_GETSNODE_QUERY_be`,这一设置对应到peer回应时的处理。
字段设置之后，这个ping被发送出去，具体的发送过程不在这里分析了。接下来看看peer对这个ping的回复过程。

### peer对GETSNODE请求的回应 ###
对于回应过程的分析，直接从ControlHandler.c中的incomingFromCore函数开始。无关内容太多，直接给出核心代码：
```
else if (ctrl->header.type_be == Control_GETSNODE_QUERY_be) {
        return handleGetSnodeQuery(msg, ch, label, labelStr);

    }
```
```
static Iface_DEFUN handleGetSnodeQuery(struct Message* msg,
                                       struct ControlHandler_pvt* ch,
                                       uint64_t label,
                                       uint8_t* labelStr)
{
    Log_debug(ch->log, "incoming getSupernode query");
    if (msg->length < handleGetSnodeQuery_MIN_SIZE) {
        Log_info(ch->log, "DROP runt getSupernode query");
        return NULL;
    }

    struct Control* ctrl = (struct Control*) msg->bytes;
    struct Control_GetSnode* snq = &ctrl->content.getSnode;

    if (snq->magic != Control_GETSNODE_QUERY_MAGIC) {
        Log_debug(ch->log, "DROP getSupernode query (bad magic)");
        return NULL;
    }

    uint32_t herVersion = Endian_bigEndianToHost32(snq->version_be);
    if (!Version_isCompatible(Version_CURRENT_PROTOCOL, herVersion)) {
        Log_debug(ch->log, "DROP getSupernode query from incompatible version [%d]", herVersion);
        return NULL;
    }

    ctrl->header.type_be = Control_GETSNODE_REPLY_be;
    snq->kbps_be = 0xffffffff;
    snq->version_be = Endian_hostToBigEndian32(Version_CURRENT_PROTOCOL);
    snq->magic = Control_GETSNODE_REPLY_MAGIC;
    if (ch->activeSnode.path) {
        uint64_t fixedLabel = NumberCompress_getLabelFor(ch->activeSnode.path, label);
        uint64_t fixedLabel_be = Endian_hostToBigEndian64(fixedLabel);
        Bits_memcpy(snq->pathToSnode_be, &fixedLabel_be, 8);
        Bits_memcpy(&snq->snodeKey, ch->activeSnode.key, 32);
        snq->snodeVersion_be = Endian_hostToBigEndian32(ch->activeSnode.protocolVersion);

    } else {
        snq->snodeVersion_be = 0;
        Bits_memset(snq->pathToSnode_be, 0, 8);
        Bits_memcpy(&snq->snodeKey, ch->activeSnode.key, 32);
    }

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
主要操作是将snode相关的内容放入struct Control_GetSnode* snq当中：
```
    struct Control_GetSnode* snq = &ctrl->content.getSnode;

    if (snq->magic != Control_GETSNODE_QUERY_MAGIC) {
        Log_debug(ch->log, "DROP getSupernode query (bad magic)");
        return NULL;
    }

    uint32_t herVersion = Endian_bigEndianToHost32(snq->version_be);
    if (!Version_isCompatible(Version_CURRENT_PROTOCOL, herVersion)) {
        Log_debug(ch->log, "DROP getSupernode query from incompatible version [%d]", herVersion);
        return NULL;
    }

    ctrl->header.type_be = Control_GETSNODE_REPLY_be;
    snq->kbps_be = 0xffffffff;
    snq->version_be = Endian_hostToBigEndian32(Version_CURRENT_PROTOCOL);
    snq->magic = Control_GETSNODE_REPLY_MAGIC;
    if (ch->activeSnode.path) {
        uint64_t fixedLabel = NumberCompress_getLabelFor(ch->activeSnode.path, label);
        uint64_t fixedLabel_be = Endian_hostToBigEndian64(fixedLabel);
        Bits_memcpy(snq->pathToSnode_be, &fixedLabel_be, 8);
        Bits_memcpy(&snq->snodeKey, ch->activeSnode.key, 32);
        snq->snodeVersion_be = Endian_hostToBigEndian32(ch->activeSnode.protocolVersion);

    } else {
        snq->snodeVersion_be = 0;
        Bits_memset(snq->pathToSnode_be, 0, 8);
        Bits_memcpy(&snq->snodeKey, ch->activeSnode.key, 32);
    }
```

### 收到GETSNODE请求的回复后，peer的处理 ###
在上面发起请求时，我们使用的是
```
    struct SwitchPinger_Ping* p =
        SwitchPinger_newPing(peer->path, String_CONST(""), 3000, peerResponse, snp->alloc, snp->sp);
    Assert_true(p);
```
可以看到，callback函数是peerResponse，查看这个方法
```

static void peerResponse(struct SwitchPinger_Response* resp, void* userData)
{
    struct SupernodeHunter_pvt* snp = Identity_check((struct SupernodeHunter_pvt*) userData);
    char* err = "";
    switch (resp->res) {
        case SwitchPinger_Result_OK: peerResponseOK(resp, snp); return;
        case SwitchPinger_Result_LABEL_MISMATCH: err = "LABEL_MISMATCH"; break;
        case SwitchPinger_Result_WRONG_DATA: err = "WRONG_DATA"; break;
        case SwitchPinger_Result_ERROR_RESPONSE: err = "ERROR_RESPONSE"; break;
        case SwitchPinger_Result_LOOP_ROUTE: err = "LOOP_ROUTE"; break;
        case SwitchPinger_Result_TIMEOUT: err = "TIMEOUT"; break;
        default: err = "unknown error"; break;
    }
    Log_debug(snp->log, "Error sending snp query to peer [%s]", err);
}
```
在正常的情况下，调用到peerResponseOK，查看这个方法
```
static void peerResponseOK(struct SwitchPinger_Response* resp, struct SupernodeHunter_pvt* snp)
{
    struct Address snode;
    Bits_memcpy(&snode, &resp->snode, sizeof(struct Address));
    if (!snode.path) {
        uint8_t label[20];
        AddrTools_printPath(label, resp->label);
        Log_debug(snp->log, "Peer [%s] reports no supernode", label);
        return;
    }
    uint64_t path = LabelSplicer_splice(snode.path, resp->label);
    if (path == UINT64_MAX) {
        Log_debug(snp->log, "Supernode path could not be spliced");
        return;
    }
    snode.path = path;

    struct Address* firstPeer = getPeerByNpn(snp, 0);
    if (!firstPeer) {
        Log_info(snp->log, "All peers have gone away while packet was outstanding");
        return;
    }

    // 1.
    // If we have looped around and queried all of our peers returning to the first and we have
    // still not found an snode in our authorized snodes list, we should simply accept this one.
    if (!snp->pub.snodeIsReachable && snp->nextPeer > 1 && firstPeer->path == resp->label) {
            Log_debug(snp->log,"snodesnode1 &snp->snodeCandidate:%s  pub.snodeAddr:%s",
                    Address_toString(&snp->snodeCandidate, snp->alloc)->bytes,
                    Address_toString(&snp->pub.snodeAddr, snp->alloc)->bytes);
            Log_debug(snp->log,"snodesnode1 snode:%s",
                Address_toString(&snode, snp->alloc)->bytes);
            Log_debug(snp->log,"snodesnode1 myAddress:%s",
                Address_toString(snp->myAddress, snp->alloc)->bytes);
        if (!snp->snodeCandidate.path) {
            Log_info(snp->log, "No snode candidate found");
            snp->nextPeer = 0;
            return;
        }
        adoptSupernode(snp, &snp->snodeCandidate);
        return;
    }

    // 2.
    // If this snode is one of our authorized snodes OR if we have none defined, accept this one.
    if (!snp->authorizedSnodes->length || AddrSet_indexOf(snp->authorizedSnodes, &snode) > -1) {
            Log_debug(snp->log,"snodesnode2 &snp->snodeCandidate:%s  pub.snodeAddr:%s",
                Address_toString(&snp->snodeCandidate, snp->alloc)->bytes,
                Address_toString(&snp->pub.snodeAddr, snp->alloc)->bytes);
            Log_debug(snp->log,"snodesnode2 snode:%s",
                Address_toString(&snode, snp->alloc)->bytes);
        Address_getPrefix(&snode);
        adoptSupernode(snp, &snode);
        return;
    }

    if (!snp->snodeCandidate.path) {
        Log_debug(snp->log,"snodesnode2.5 &snp->snodeCandidate:%s  pub.snodeAddr:%s",
                Address_toString(&snp->snodeCandidate, snp->alloc)->bytes,
                Address_toString(&snp->pub.snodeAddr, snp->alloc)->bytes);
        Log_debug(snp->log,"snodesnode2.5 snode:%s",
                Address_toString(&snode, snp->alloc)->bytes);
        Bits_memcpy(&snp->snodeCandidate, &snode, sizeof(struct Address));
        Address_getPrefix(&snp->snodeCandidate);
    }

    // 3.
    // If this snode is not one of our authorized snodes, query it for all of our authorized snodes.
    queryForAuthorized(snp, &snode);
}
```
这段代码中，首先要注意的是它对peer返回的snode的path做了一个LabelSplicer_splice操作。这个操作的作用是基于peer在自己这里的path和snode在peer那里的path，来推算出snode在自己这里应有的path。
```
uint64_t path = LabelSplicer_splice(snode.path, resp->label);
```
其中，snode.path就是snode在peer那里的path，而resp->laber是peer在自己这里的path。
LabelSplicer_splice方法可以基于peer在自己这里的path和snode在peer那里的path，来推算出snode在自己这里应有的path。

由于这段代码已经被修改过，情况3已经不再处理，这段代码已经不是原始的cjdns代码，所以，只简单分析大致的逻辑。
当前点会询问所有的peer，当收到第一个peer返回的snode之后，将这个snode的地址记录在snodeCandidate中，继续等待其他peer的回应，在这个过程中，如果某个peer回应的snode是当前点在conf中配置的snode，那么接受这个在conf中配置的snode。
如果所有的peer都已经回复的自己的snode，而其中并没有当前点在conf中配置的snode，那么接受snodeCandidate，也就是第一个peer回复的snode。
接下来，看一下什么是接受snode。所谓接受snode，就是adoptSupernode方法。
```
static void adoptSupernode(struct SupernodeHunter_pvt* snp, struct Address* candidate)
{
    struct MsgCore_Promise* qp = MsgCore_createQuery(snp->msgCore, 0, snp->alloc);
    struct Query* q = Allocator_calloc(qp->alloc, sizeof(struct Query), 1);
    Identity_set(q);
    q->snp = snp;
    q->sendTime = Time_currentTimeMilliseconds(snp->base);

    Dict* msg = qp->msg = Dict_new(qp->alloc);
    qp->cb = adoptSupernode2;
    qp->userData = q;
    qp->target = Address_clone(candidate, qp->alloc);

    Log_debug(snp->log, "Pinging snode [%s]", Address_toString(qp->target, qp->alloc)->bytes);
    Dict_putStringCC(msg, "sq", "pn", qp->alloc);

    Assert_true(AddressCalc_validAddress(candidate->ip6.bytes));
    return;
}
```
这段代码的作用是向snode发送一个pn请求，试图连接这个snode，如果收到snode的回复，进入到adoptSupernode2
```
static void adoptSupernode2(Dict* msg, struct Address* src, struct MsgCore_Promise* prom)
{
    struct Query* q = Identity_check((struct Query*) prom->userData);
    struct SupernodeHunter_pvt* snp = Identity_check(q->snp);

    if (!src) {
        String* addrStr = Address_toString(prom->target, prom->alloc);
        Log_debug(snp->log, "timeout sending to %s", addrStr->bytes);
        if (!Bits_memcmp(prom->target, snp->myAddress, Address_SIZE)) {
            askPeer(snp);
            //~ snp->pub.snodeIsReachable = 0;
            Log_debug(snp->log, "snode snode my cjdnsnode is not working try to ask peer"
                    " now snodeIsReachable:%d",snp->pub.snodeIsReachable);
        }
        return;
    }
    String* addrStr = Address_toString(src, prom->alloc);
    Log_debug(snp->log, "Reply from %s", addrStr->bytes);

    int64_t* snodeRecvTime = Dict_getIntC(msg, "recvTime");
    if (!snodeRecvTime) {
        Log_info(snp->log, "getRoute reply with no timeStamp, bad snode");
        return;
    }
    Log_debug(snp->log, "\n\nSupernode location confirmed [%s]\n\n",
        Address_toString(src, prom->alloc)->bytes);
    if (snp->pub.snodeIsReachable && Bits_memcmp(src, snp->myAddress, Address_SIZE)) {
        // If while we were searching, the outside code declared that indeed the snode
        // is reachable, we will not try to change their snode.
    } else if (snp->pub.onSnodeChange) {
            Bits_memcpy(&snp->pub.snodeAddr, src, Address_SIZE);
            snp->pub.snodeIsReachable = (AddrSet_indexOf(snp->authorizedSnodes, src) != -1) ? 2 : 1;
            snp->pub.onSnodeChange(&snp->pub, q->sendTime, *snodeRecvTime);
            Notification_snode(snp->pub.notification,REACHABLE);
    } else {
            Log_warn(snp->log, "onSnodeChange is not set");
    }
    SupernodeStore_updateSnodeAddress(snp->pub.snodeStore, src, true);
}
```
在这个方法中，对snp->pub.snodeAddr进行赋值，并且处理了一些在snode发生变化时，应该做的操作。
至此，snodeAddr的赋值完成，snode就算稳定了。

## 向snode点询问去某点的路径 ##
### 向snode发送地址询问请求 ###
从SessionManager.c的triggerSearch方法开始分析。这是询问请求的开始之处。有两个地方会调用到triggerSearch方法，分别是：
1. checkTimedOutSessions
2. needsLookup

下面来看一下triggerSearch方法的源码：
```
static void triggerSearch(struct SessionManager_pvt* sm, uint8_t target[16], uint32_t version)
{
    struct Allocator* eventAlloc = Allocator_child(sm->alloc);
    struct Message* eventMsg = Message_new(0, 512, eventAlloc);
    Message_push32(eventMsg, version, NULL);
    Message_push32(eventMsg, 0, NULL);
    Message_push(eventMsg, target, 16, NULL);
    Message_push32(eventMsg, 0xffffffff, NULL);
    Message_push32(eventMsg, PFChan_Core_SEARCH_REQ, NULL);
    Iface_send(&sm->eventIf, eventMsg);
    Allocator_free(eventAlloc);
}
```
其中，target是对方的ipv6地址，version是对方的版本。
最后调用`Iface_send(&sm->eventIf, eventMsg);`
所以接下来查找sm->eventIf和什么绑定。
在SessionManager_new中:
```
EventEmitter_regCore(ee, &sm->eventIf, PFChan_Pathfinder_NODE);
```
查看这个方法：
```
void EventEmitter_regCore(struct EventEmitter* eventEmitter,
                          struct Iface* iface,
                          enum PFChan_Pathfinder ev)
{
    struct EventEmitter_pvt* ee = Identity_check((struct EventEmitter_pvt*) eventEmitter);
    iface->connectedIf = &ee->trickIf;
    struct ArrayList_Ifaces* l = getHandlers(ee, ev, true);
    if (!l) {
        Assert_true(ev == 0);
        return;
    }
    ArrayList_Ifaces_add(l, iface);
}
```
继续查找ee->trickIf，在EventEmitter_new中：
```
ee->trickIf.send = incomingFromCore;
```
查看incomingFromCore
```
static Iface_DEFUN incomingFromCore(struct Message* msg, struct Iface* trickIf)
{
    struct EventEmitter_pvt* ee = Identity_containerOf(trickIf, struct EventEmitter_pvt, trickIf);
    Assert_true(!((uintptr_t)msg->bytes % 4) && "alignment");
    enum PFChan_Core ev = Message_pop32(msg, NULL);
    Assert_true(PFChan_Core_sizeOk(ev, msg->length+4));
    uint32_t pathfinderNum = Message_pop32(msg, NULL);
    Message_push32(msg, ev, NULL);
    if (pathfinderNum != 0xffffffff) {
        struct Pathfinder* pf = ArrayList_Pathfinders_get(ee->pathfinders, pathfinderNum);
        Assert_true(pf && pf->state == Pathfinder_state_CONNECTED);
        return sendToPathfinder(msg, pf);
    } else {
        for (int i = 0; i < ee->pathfinders->length; i++) {
            struct Pathfinder* pf = ArrayList_Pathfinders_get(ee->pathfinders, i);
            if (!pf || pf->state != Pathfinder_state_CONNECTED) { continue; }
            struct Message* messageClone = Message_clone(msg, msg->alloc);
            Iface_CALL(sendToPathfinder, messageClone, pf);
        }
    }
    return NULL;
}
```
根据message push和pop的顺序关系，可以知道，pathfinderNum等于0xffffffff，执行：
```
        for (int i = 0; i < ee->pathfinders->length; i++) {
            struct Pathfinder* pf = ArrayList_Pathfinders_get(ee->pathfinders, i);
            if (!pf || pf->state != Pathfinder_state_CONNECTED) { continue; }
            struct Message* messageClone = Message_clone(msg, msg->alloc);
            Iface_CALL(sendToPathfinder, messageClone, pf);
        }
```
查看sendToPathfinder
```
static Iface_DEFUN sendToPathfinder(struct Message* msg, struct Pathfinder* pf)
{
    if (!pf || pf->state != Pathfinder_state_CONNECTED) { return NULL; }
    if (pf->bytesSinceLastPing < 8192 && pf->bytesSinceLastPing + msg->length >= 8192) {
        struct Message* ping = Message_new(0, 512, msg->alloc);
        Message_push32(ping, pf->bytesSinceLastPing, NULL);
        Message_push32(ping, PING_MAGIC, NULL);
        Message_push32(ping, PFChan_Core_PING, NULL);
        Iface_send(&pf->iface, ping);
    }
    pf->bytesSinceLastPing += msg->length;
    return Iface_next(&pf->iface, msg);
}
```
调用到`Iface_next(&pf->iface, msg);`
pf->iface在EventEmitter_regPathfinderIface中设置：
```
void EventEmitter_regPathfinderIface(struct EventEmitter* emitter, struct Iface* iface)
{
    struct EventEmitter_pvt* ee = Identity_check((struct EventEmitter_pvt*) emitter);
    struct Allocator* alloc = Allocator_child(ee->alloc);
    struct Pathfinder* pf = Allocator_calloc(alloc, sizeof(struct Pathfinder), 1);
    pf->ee = ee;
    pf->iface.send = incomingFromPathfinder;
    pf->alloc = alloc;
    Iface_plumb(&pf->iface, iface);
    Identity_set(pf);
    int i = 0;
    for (; i < ee->pathfinders->length; i++) {
        struct Pathfinder* xpf = ArrayList_Pathfinders_get(ee->pathfinders, i);
        if (!xpf) { break; }
    }
    pf->pathfinderId = ArrayList_Pathfinders_put(ee->pathfinders, i, pf);
}
```
其中的`Iface_plumb(&pf->iface, iface);`就是绑定pf->iface和iface的语句。
这个函数的调用在Core.c中，分为两种情况：
```
    struct SubnodePathfinder* spf = SubnodePathfinder_new(
        alloc, logger, eventBase, rand, nc->myAddress, privateKey, encodingScheme, notification);
    struct ASynchronizer* spfAsync = ASynchronizer_new(alloc, eventBase, logger);
    Iface_plumb(&spfAsync->ifA, &spf->eventIf);
    EventEmitter_regPathfinderIface(nc->ee, &spfAsync->ifB);

    #ifndef SUBNODE
        struct Pathfinder* opf = Pathfinder_register(alloc, logger, eventBase, rand, admin);
        struct ASynchronizer* opfAsync = ASynchronizer_new(alloc, eventBase, logger);
        Iface_plumb(&opfAsync->ifA, &opf->eventIf);
        EventEmitter_regPathfinderIface(nc->ee, &opfAsync->ifB);
    #endif
```
目前系统中启用了SUBNODE，所以：只分析上面那种。也就是
```
    struct SubnodePathfinder* spf = SubnodePathfinder_new(
        alloc, logger, eventBase, rand, nc->myAddress, privateKey, encodingScheme, notification);
    struct ASynchronizer* spfAsync = ASynchronizer_new(alloc, eventBase, logger);
    Iface_plumb(&spfAsync->ifA, &spf->eventIf);
    EventEmitter_regPathfinderIface(nc->ee, &spfAsync->ifB);
```
它将&spfAsync->ifA 和 &spf->eventIf绑定，并通过函数调用，将pf->iface 和 &spfAsync->ifB绑定。
现在先查看spfAsync->ifB相关代码，在ASynchronizer.c的ASynchronizer_new中有
```
ctx->pub.ifB.send = fromB;
```
查看fromB
```
static Iface_DEFUN fromB(struct Message* msg, struct Iface* ifB)
{
    struct ASynchronizer_pvt* as = Identity_containerOf(ifB, struct ASynchronizer_pvt, pub.ifB);
    if (!as->cycleAlloc) { as->cycleAlloc = Allocator_child(as->alloc); }
    if (!as->msgsToA) { as->msgsToA = ArrayList_Messages_new(as->cycleAlloc); }
    Allocator_adopt(as->cycleAlloc, msg->alloc);
    ArrayList_Messages_add(as->msgsToA, msg);
    checkTimeout(as);
    return NULL;
}
```
注意这两句：
```
    ArrayList_Messages_add(as->msgsToA, msg);
    checkTimeout(as);
```
这个消息被放入到了as->msgsToA当中，然后调用了checkTimeout(as)
```
static void checkTimeout(struct ASynchronizer_pvt* as)
{
    if (as->timeoutAlloc) { return; }
    as->timeoutAlloc = Allocator_child(as->alloc);
    Timeout_setInterval(timeoutTrigger, as, 1, as->base, as->timeoutAlloc);
}
```
调用timeoutTrigger
```
static void timeoutTrigger(void* vASynchronizer)
{
    struct ASynchronizer_pvt* as = Identity_check((struct ASynchronizer_pvt*) vASynchronizer);

    if (!as->cycleAlloc) {
        if (as->dryCycles++ < MAX_DRY_CYCLES || !as->timeoutAlloc) { return; }
        Allocator_free(as->timeoutAlloc);
        as->timeoutAlloc = NULL;
        as->dryCycles = 0;
        return;
    }

    struct ArrayList_Messages* msgsToA = as->msgsToA;
    struct ArrayList_Messages* msgsToB = as->msgsToB;
    struct Allocator* cycleAlloc = as->cycleAlloc;
    as->msgsToA = NULL;
    as->msgsToB = NULL;
    as->cycleAlloc = NULL;

    if (msgsToA) {
        for (int i = 0; i < msgsToA->length; i++) {
            struct Message* msg = ArrayList_Messages_get(msgsToA, i);
            Iface_send(&as->pub.ifA, msg);
        }
    }
    if (msgsToB) {
        for (int i = 0; i < msgsToB->length; i++) {
            struct Message* msg = ArrayList_Messages_get(msgsToB, i);
            Iface_send(&as->pub.ifB, msg);
        }
    }
    Allocator_free(cycleAlloc);
}
```
之前的代码中，将msg加入到了as->msgsToA中，所以，这里会执行到：
```
    if (msgsToA) {
        for (int i = 0; i < msgsToA->length; i++) {
            struct Message* msg = ArrayList_Messages_get(msgsToA, i);
            Iface_send(&as->pub.ifA, msg);
        }
    }
```
调用到`Iface_send(&as->pub.ifA, msg);`
前面在Core.c中讲到过，`Iface_plumb(&spfAsync->ifA, &spf->eventIf);`将spfAsync->ifA 和 &spf->eventIf绑定，所以，现在要寻找`spf->eventIf`，在subnode/SubnodePathfinder.c的SubnodePathfinder_new方法中：
```
pf->pub.eventIf.send = incomingFromEventIf;
```
查看incomingFromEventIf方法，直接给出核心代码：
```
case PFChan_Core_SEARCH_REQ: return searchReq(msg, pf);
```
查看serchReq方法：
```
static Iface_DEFUN searchReq(struct Message* msg, struct SubnodePathfinder_pvt* pf)
{
    uint8_t addr[16];
    Message_pop(msg, addr, 16, NULL);
    Message_pop32(msg, NULL);
    uint32_t version = Message_pop32(msg, NULL);
    if (version && version < 20) { return NULL; }
    Assert_true(!msg->length);
    uint8_t printedAddr[40];
    AddrTools_printIp(printedAddr, addr);
    Log_debug(pf->log, "Search req [%s]", printedAddr);

    if (!pf->pub.snh || !pf->pub.snh->snodeAddr.path) { return NULL; }

    if (!Bits_memcmp(pf->pub.snh->snodeAddr.ip6.bytes, addr, 16)) {
        return sendNode(msg, &pf->pub.snh->snodeAddr, 0xfff00000, PFChan_Pathfinder_NODE, pf);
    }

    struct MsgCore_Promise* qp = MsgCore_createQuery(pf->msgCore, 0, pf->alloc);

    Dict* dict = qp->msg = Dict_new(qp->alloc);
    qp->cb = getRouteReply;
    qp->userData = pf;

    Assert_true(AddressCalc_validAddress(pf->pub.snh->snodeAddr.ip6.bytes));
    qp->target = &pf->pub.snh->snodeAddr;

    Log_debug(pf->log, "Sending getRoute to snode %s",
        Address_toString(qp->target, qp->alloc)->bytes);
    Dict_putStringCC(dict, "sq", "gr", qp->alloc);
    String* src = String_newBinary(pf->myAddress->ip6.bytes, 16, qp->alloc);
    Dict_putStringC(dict, "src", src, qp->alloc);
    String* target = String_newBinary(addr, 16, qp->alloc);
    Dict_putStringC(dict, "tar", target, qp->alloc);

    return NULL;
}
```
发往snode的信息包括：
* "sq":"gr"
* "src":自己的ipv6地址
* "tar":要寻找的点的ipv6地址
这些信息都包含在MsgCore_Promise* qp的msg当中。

此外，qp的target为snode的地址，也就是发往snode
qp的cb是getRouteReply,这是回调函数。

再回头看这一句：
```
    struct MsgCore_Promise* qp = MsgCore_createQuery(pf->msgCore, 0, pf->alloc);
```
查看MsgCore_createQuery方法，在subnode/MsgCore.c中
```
struct MsgCore_Promise* MsgCore_createQuery(struct MsgCore* core,
                                            uint32_t timeoutMilliseconds,
                                            struct Allocator* allocator)
{
    struct MsgCore_pvt* mcp = Identity_check((struct MsgCore_pvt*) core);
    if (!timeoutMilliseconds) {
        timeoutMilliseconds = DEFAULT_TIMEOUT_MILLISECONDS;
    }
    struct Pinger_Ping* p = Pinger_newPing(
        NULL, pingerOnResponse, pingerSendPing, timeoutMilliseconds, allocator, mcp->pinger);
    struct MsgCore_Promise_pvt* out =
        Allocator_calloc(p->pingAlloc, sizeof(struct MsgCore_Promise_pvt), 1);
    Identity_set(out);
    p->context = out;
    out->pub.alloc = p->pingAlloc;
    out->mcp = mcp;
    out->ping = p;
    return &out->pub;
}
```
这是一个Pinger_Ping。pingerSendPing负责发送ping
```
static void pingerSendPing(String* data, void* context)
{
    struct MsgCore_Promise_pvt* pp = Identity_check((struct MsgCore_Promise_pvt*) context);
    Assert_true(pp->pub.target);
    Assert_true(pp->pub.msg);
    Dict_putStringC(pp->pub.msg, "txid", data, pp->pub.alloc);
    sendMsg(pp->mcp, pp->pub.msg, pp->pub.target, pp->pub.alloc);
}
```
```
static void sendMsg(struct MsgCore_pvt* mcp,
                    Dict* msgDict,
                    struct Address* addr,
                    struct Allocator* allocator)
{
    struct Allocator* alloc = Allocator_child(allocator);

    // Send the encoding scheme definition
    Dict_putString(msgDict, CJDHTConstants_ENC_SCHEME, mcp->schemeDefinition, allocator);

    // And tell the asker which interface the message came from
    int encIdx = EncodingScheme_getFormNum(mcp->scheme, addr->path);
    Assert_true(encIdx != EncodingScheme_getFormNum_INVALID);
    Dict_putInt(msgDict, CJDHTConstants_ENC_INDEX, encIdx, allocator);

    // send the protocol version
    Dict_putInt(msgDict, CJDHTConstants_PROTOCOL, Version_CURRENT_PROTOCOL, allocator);

    if (!Defined(SUBNODE)) {
        ......
    }

    struct Message* msg = Message_new(0, 2048, alloc);
    BencMessageWriter_write(msgDict, msg, NULL);

    // Sanity check (make sure the addr was actually calculated)
    Assert_true(AddressCalc_validAddress(addr->ip6.bytes));

    struct DataHeader data;
    Bits_memset(&data, 0, sizeof(struct DataHeader));
    DataHeader_setVersion(&data, DataHeader_CURRENT_VERSION);
    DataHeader_setContentType(&data, ContentType_CJDHT);
    Message_push(msg, &data, sizeof(struct DataHeader), NULL);

    struct RouteHeader route;
    Bits_memset(&route, 0, sizeof(struct RouteHeader));
    Bits_memcpy(route.ip6, addr->ip6.bytes, 16);
    route.version_be = Endian_hostToBigEndian32(addr->protocolVersion);
    route.sh.label_be = Endian_hostToBigEndian64(addr->path);
    Bits_memcpy(route.publicKey, addr->key, 32);
    Message_push(msg, &route, sizeof(struct RouteHeader), NULL);

    Iface_send(&mcp->pub.interRouterIf, msg);
}
```
注意这一句：
```
    DataHeader_setContentType(&data, ContentType_CJDHT);
```
其中ContentType_CJDHT的值为256.
之后的ping操作不再具体分析。

## snode收到了地址询问请求 ##
snode通过server.js提供寻址服务。源码在cjdnsnode项目的server.js中
首先是main
```
const main = (ctx) => {
    const config = getConfig();
    if (!ctx) {
        ......
    }

    nThen((waitFor) => {
        if (Object.keys(ctx.nodesByIp).length == 0)
            loadDb(ctx, waitFor());
        else
            waitFor();
    }).nThen((waitFor) => {
        //keepTableClean(ctx);
        if (config.connectCjdns) { service(ctx); }
        testSrv(ctx);
        ctx.peer.onAnnounce((peer, msg) => { handleAnnounce(ctx, msg, false, false); });
        (ctx.config.peers || []).forEach(ctx.peer.connectTo);
    });
};
```
调用`service(ctx)`
```
const service = (ctx) => {
    let cjdns;
    nThen((waitFor) => {
        Cjdnsadmin.connectWithAdminInfo(waitFor((c) => { cjdns = c; }));
    }).nThen((waitFor) => {
        ......
    }).nThen((waitFor) => {
        Cjdnsniff.sniffTraffic(cjdns, 'CJDHT', waitFor((err, cjdnslink) => {
            console.log("Connected to cjdns engine");
            if (err) { throw err; }
            cjdnslink.on('error', (e) => {
                console.error('sniffTraffic error');
                console.error(e.stack);
            });
            cjdnslink.on('message', (msg) => {
                /*::msg = (msg:Cjdnsniff_BencMsg_t);*/
                onSubnodeMessage(ctx, msg, cjdnslink);
            });
        }));
    }).nThen((waitFor) => {
        ......
    });
};
```
调用`onSubnodeMessage(ctx, msg, cjdnslink);`
```
const onSubnodeMessage = (ctx, msg, cjdnslink) => {
    if (!msg.contentBenc.sq) { return; }
    if (!msg.routeHeader.version) {
        if (msg.contentBenc.p) {
            msg.routeHeader.version = msg.contentBenc.p;
        }
    }
    if (!msg.routeHeader.version || !msg.routeHeader.publicKey) {
        //if (msg.routeHeader.ip) {
        console.log("message from " + msg.routeHeader.ip + `with missing key(${msg.routeHeader.version}) or version(${msg.routeHeader.publicKey})`);
        //}
        return;
    }
    if (msg.contentBenc.sq.toString('utf8') === 'gr') {
        const srcIp = Cjdnskeys.ip6BytesToString(msg.contentBenc.src);
        const tarIp = Cjdnskeys.ip6BytesToString(msg.contentBenc.tar);
        const src = ctx.nodesByIp[srcIp];
        const tar = ctx.nodesByIp[tarIp];
        const logMsg = "getRoute req " + srcIp + " " + tarIp + "  ";
        const r = getRoute(ctx, src, tar);

        if (r) {
            console.log(logMsg + r.label);
            msg.contentBenc.n = Buffer.concat([
                Cjdnskeys.keyStringToBytes(tar.key),
                new Buffer(r.label.replace(/\./g, ''), 'hex')
            ]);
            msg.contentBenc.np = new Buffer([1, tar.version]);
        } else {
            console.log(logMsg + "not found");
        }
        msg.contentBenc.recvTime = now();
        msg.routeHeader.switchHeader.labelShift = 0;

        delete msg.contentBenc.sq;
        delete msg.contentBenc.src;
        delete msg.contentBenc.tar;
        cjdnslink.send(msg);
    } else if (msg.contentBenc.sq.toString('utf8') === 'ann') {
        ......
    } else if (msg.contentBenc.sq.toString('utf8') === 'pn') {
        ......
    } else {
        console.log(msg.contentBenc);
    }
};
```
调用getRoute(ctx, src, tar);
```
const getRoute = (ctx, src, dst) => {
    if (!src || !dst) { return null; }

    if (src === dst) {
        return { label: '0000.0000.0000.0001', hops: [] };
    }

    //if (!ctx.mut.dijkstra) { 
    ctx.mut.routeCache = {};
    const dijkstra = ctx.mut.dijkstra = new Dijkstra();
    for (const nip in ctx.nodesByIp) {
        const links = ctx.nodesByIp[nip].inwardLinksByIp;
        const l = {};
        for (const pip in links) { l[pip] = linkValue(links[pip]); }
        ctx.mut.dijkstra.addNode(nip, l);
    }
    //}

    // const cachedEntry = ctx.mut.routeCache[dst.ipv6 + '|' + src.ipv6];
    // if (typeof (cachedEntry) !== 'undefined') {
    //     return cachedEntry;
    // }

    // we ask for the path in reverse because we build the graph in reverse.
    // because nodes announce own their reachability instead of announcing reachability of others.
    const path = ctx.mut.dijkstra.path(dst.ipv6, src.ipv6);
    if (!path) {
        return (ctx.mut.routeCache[dst.ipv6 + '|' + src.ipv6] = null);
    }
    path.reverse();
    let last;
    let lastLink;
    const hops = [];
    const labels = [];
    let formNum;

    path.forEach((nip) => {
        const node = ctx.nodesByIp[nip];
        if (last) {
            const link = node.inwardLinksByIp[last.ipv6];
            let label = link.label;
            const curFormNum = Cjdnsplice.getEncodingForm(label, last.encodingScheme);
            if (curFormNum < formNum) {
                label = Cjdnsplice.reEncode(label, last.encodingScheme, formNum);
            }
            labels.unshift(label);
            hops.push({
                label: label,
                origLabel: link.label,
                scheme: last.encodingScheme,
                inverseFormNum: formNum
            });
            formNum = link.encodingFormNum;
        }
        last = node;
    });
    labels.unshift('0000.0000.0000.0001');
    const spliced = Cjdnsplice.splice.apply(null, labels);
    return (ctx.mut.routeCache[dst.ipv6 + '|' + src.ipv6] = { label: spliced, hops: hops, path: path });
};
```

## 收到snode的地址请求回复后的处理 ##
上面提到，地址请求的回调函数是subnode/SubnodePathfinder.c中的getRouteReply方法
```
static void getRouteReply(Dict* msg, struct Address* src, struct MsgCore_Promise* prom)
{
    struct SubnodePathfinder_pvt* pf =
        Identity_check((struct SubnodePathfinder_pvt*) prom->userData);
    if (!src) {
        Log_debug(pf->log, "GetRoute timeout");
        return;
    }
    Log_debug(pf->log, "Search reply!");
    struct Address_List* al = ReplySerializer_parse(src, msg, pf->log, false, prom->alloc);
    if (!al || al->length == 0) { return; }
    Log_debug(pf->log, "reply with[%s]", Address_toString(&al->elems[0], prom->alloc)->bytes);

    if (al->elems[0].protocolVersion < 20) {
        Log_debug(pf->log, "not sending [%s] because version is old",
            Address_toString(&al->elems[0], prom->alloc)->bytes);
        return;
    }

    //NodeCache_discoverNode(pf->nc, &al->elems[0]);
    struct Message* msgToCore = Message_new(0, 512, prom->alloc);
    Iface_CALL(sendNode, msgToCore, &al->elems[0], 0xfff00033, PFChan_Pathfinder_NODE, pf);
}
```
调用sendNode
```
static Iface_DEFUN sendNode(struct Message* msg,
                            struct Address* addr,
                            uint32_t metric,
                            enum PFChan_Pathfinder msgType,
                            struct SubnodePathfinder_pvt* pf)
{
    Message_reset(msg);
    Message_shift(msg, PFChan_Node_SIZE, NULL);
    nodeForAddress((struct PFChan_Node*) msg->bytes, addr, metric);
    if (addr->path == UINT64_MAX) {
        ((struct PFChan_Node*) msg->bytes)->path_be = 0;
    }
    Message_push32(msg, msgType, NULL);
    return Iface_next(&pf->pub.eventIf, msg);
}
```
最后是一个Iface_next,上面讲到过，在Core.c中将(&spfAsync->ifA 和 &spf->eventIf绑定，所以，接下来看ASynchronizer的ifA，代码在interface/ASyncronizer.c中，因为ifA和ifB代码完全对称，所以不再做详细分析，调用会因为在Core.c中的设置将将pf->iface 和 &spfAsync->ifB绑定，从而调用到net/EventEmitter.c中的incomingFromPathfinder方法
```
static Iface_DEFUN incomingFromPathfinder(struct Message* msg, struct Iface* iface)
{
    struct Pathfinder* pf = Identity_containerOf(iface, struct Pathfinder, iface);
    struct EventEmitter_pvt* ee = Identity_check((struct EventEmitter_pvt*) pf->ee);
    if (msg->length < 4) {
        Log_debug(ee->log, "DROPPF runt");
        return NULL;
    }
    enum PFChan_Pathfinder ev = Message_pop32(msg, NULL);
    Message_push32(msg, pf->pathfinderId, NULL);
    Message_push32(msg, ev, NULL);
    if (ev <= PFChan_Pathfinder__TOO_LOW || ev >= PFChan_Pathfinder__TOO_HIGH) {
        Log_debug(ee->log, "DROPPF invalid type [%d]", ev);
        return NULL;
    }
    if (!PFChan_Pathfinder_sizeOk(ev, msg->length)) {
        Log_debug(ee->log, "DROPPF incorrect length[%d] for type [%d]", msg->length, ev);
        return NULL;
    }

    if (pf->state == Pathfinder_state_DISCONNECTED) {
        if (ev != PFChan_Pathfinder_CONNECT) {
            Log_debug(ee->log, "DROPPF disconnected and event != CONNECT event:[%d]", ev);
            return NULL;
        }
    } else if (pf->state != Pathfinder_state_CONNECTED) {
        Log_debug(ee->log, "DROPPF error state");
        return NULL;
    }

    if (handleFromPathfinder(ev, msg, ee, pf)) { return NULL; }

    struct ArrayList_Ifaces* handlers = getHandlers(ee, ev, false);
    if (!handlers) { return NULL; }
    for (int i = 0; i < handlers->length; i++) {
        struct Message* messageClone = Message_clone(msg, msg->alloc);
        struct Iface* iface = ArrayList_Ifaces_get(handlers, i);
        // We have to call this manually because we don't have an interface handy which is
        // actually plumbed with this one.
        Assert_true(iface);
        Assert_true(iface->send);
        Iface_CALL(iface->send, messageClone, iface);
    }
    return NULL;
}
```