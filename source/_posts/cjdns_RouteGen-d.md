---
title: cjdns源码分析--RouteGen
category:
  - cjdns
  - cjdns源码分析
tags:
  - cjdns
  - cjdns源码分析
  - C/C++
description: "RouteGen主要是跟路由相关的代码。"
date: 2017-08-02 11:42:23
---
# 源码分析 #
## 结构体 ##
```
struct RouteGen
{
    bool hasUncommittedChanges;
};


struct RouteGen_pvt
{
    struct RouteGen pub;
    struct ArrayList_OfPrefix6* prefixes6;
    struct ArrayList_OfPrefix6* localPrefixes6;
    struct ArrayList_OfPrefix6* exceptions6;

    struct ArrayList_OfPrefix4* prefixes4;
    struct ArrayList_OfPrefix4* localPrefixes4;
    struct ArrayList_OfPrefix4* exceptions4;

    struct Allocator* alloc;
    struct Log* log;

    Identity
};
```
可以看到，ipv4和ipv6各有三个ArrayList。这里面放着一些网段信息。

## RouteGen_new ##
这个方法在Core.c的Core_init中调用。
```
struct RouteGen* RouteGen_new(struct Allocator* allocator, struct Log* log)
{
    struct Allocator* alloc = Allocator_child(allocator);
    struct RouteGen_pvt* rp = Allocator_calloc(alloc, sizeof(struct RouteGen_pvt), 1);
    rp->prefixes6 = ArrayList_OfPrefix6_new(alloc);
    rp->localPrefixes6 = ArrayList_OfPrefix6_new(alloc);
    rp->exceptions6 = ArrayList_OfPrefix6_new(alloc);
    rp->prefixes4 = ArrayList_OfPrefix4_new(alloc);
    rp->localPrefixes4 = ArrayList_OfPrefix4_new(alloc);
    rp->exceptions4 = ArrayList_OfPrefix4_new(alloc);
    rp->log = log;
    rp->alloc = alloc;
    Identity_set(rp);
    setupDefaultLocalPrefixes(rp);
    return &rp->pub;
}
```
除了简单的赋值之外，主要做了两件事：
1. new各种ArrayList
2. setupDefaultLocalPrefixes
```
static void setupDefaultLocalPrefixes(struct RouteGen_pvt* rp)
{
    struct Sockaddr_storage ss;
    #define ADD_PREFIX(str) \
        Assert_true(!Sockaddr_parse(str, &ss));       \
        RouteGen_addLocalPrefix(&rp->pub, &ss.addr)

    ADD_PREFIX("fe80::/10");
    ADD_PREFIX("fd00::/8");

    ADD_PREFIX("10.0.0.0/8");
    ADD_PREFIX("172.16.0.0/12");
    ADD_PREFIX("192.168.0.0/16");
    ADD_PREFIX("127.0.0.0/8");

    #undef ADD_PREFIX
}
```
这个函数add了一些网段信息到LocalPrefix表。
这个过程调用到了两个函数:
1.Sockaddr_parse
将一个字符串类型的ip地址转成Sockaddr_storage类型
```
struct Sockaddr
{
    /** the length of this sockaddr, this field is included in the length. */
    uint16_t addrLen;

    #define Sockaddr_flags_BCAST  1
    #define Sockaddr_flags_PREFIX (1<<1)
    uint16_t flags;

    /** Only applies if flags & Sockaddr_flags_PREFIX is true. */
    uint8_t prefix;

    uint8_t pad1;
    uint16_t pad2;
};

struct Sockaddr_storage
{
    struct Sockaddr addr;
    uint64_t nativeAddr[Sockaddr_MAXSIZE / 8];
};

int Sockaddr_parse(const char* input, struct Sockaddr_storage* out)
{
    struct Sockaddr_storage unusedOut;
    if (!out) {
        out = &unusedOut;
    }
    uint8_t buff[64] = {0};
    if (CString_strlen(input) > 63) {
        return -1;
    }
    CString_strncpy(buff, input, 63);

    int64_t port = 0;
    char* lastColon = CString_strrchr(buff, ':');
    char* firstColon = CString_strchr(buff, ':');
    char* bracket = CString_strchr(buff, ']');
    if (!lastColon) {
        // ipv4, no port
    } else if (lastColon != firstColon && (!bracket || lastColon < bracket)) {
        // ipv6, no port
    } else {
        if (bracket && lastColon != &bracket[1]) { return -1; }
        if (Base10_fromString(&lastColon[1], &port)) { return -1; }
        if (port > 65535) { return -1; }
        *lastColon = '\0';
    }
    if (bracket) {
        *bracket = '\0';
        if (buff[0] != '[') { return -1; }
    } else if (buff[0] == '[') { return -1; }

    int64_t prefix = -1;
    char* slash = CString_strchr(buff, '/');
    if (slash) {
        *slash = '\0';
        if (!slash[1]) { return -1; }
        if (Base10_fromString(&slash[1], &prefix)) { return -1; }
    }

    Bits_memset(out, 0, sizeof(struct Sockaddr_storage));
    if (lastColon != firstColon) {
        // ipv6
        struct sockaddr_in6* in6 = (struct sockaddr_in6*) Sockaddr_asNative(&out->addr);
        if (uv_inet_pton(AF_INET6, (char*) ((buff[0] == '[') ? &buff[1] : buff), &in6->sin6_addr)) {
            return -1;
        }
        out->addr.addrLen = sizeof(struct sockaddr_in6) + Sockaddr_OVERHEAD;
        in6->sin6_port = Endian_hostToBigEndian16(port);
        in6->sin6_family = AF_INET6;
    } else {
        struct sockaddr_in* in = ((struct sockaddr_in*) Sockaddr_asNative(&out->addr));
        if (uv_inet_pton(AF_INET, (char*) buff, &in->sin_addr)) {
            return -1;
        }
        out->addr.addrLen = sizeof(struct sockaddr_in) + Sockaddr_OVERHEAD;
        in->sin_port = Endian_hostToBigEndian16(port);
        in->sin_family = AF_INET;
    }
    if (prefix != -1) {
        if (prefix < 0 || prefix > 128) { return -1; }
        if (Sockaddr_getFamily(&out->addr) == Sockaddr_AF_INET && prefix > 32) { return -1; }
        out->addr.prefix = prefix;
        out->addr.flags |= Sockaddr_flags_PREFIX;
    }
    return 0;
}
```
看一下跟prefix相关的代码，基本上可以将prefix理解为掩码的长度。这个变量可以用来判断两个ip地址是否在同一网段，或某个网段是否是另一个网段的子集。

2.RouteGen_addLocalPrefix
```
void RouteGen_addLocalPrefix(struct RouteGen* rg, struct Sockaddr* destination)
{
    struct RouteGen_pvt* rp = Identity_check((struct RouteGen_pvt*) rg);
    addSomething(rp, destination, rp->localPrefixes6, rp->localPrefixes4);
}

static void addSomething(struct RouteGen_pvt* rp,
                         struct Sockaddr* exempt,
                         struct ArrayList_OfPrefix6* list6,
                         struct ArrayList_OfPrefix4* list4)
{
    if (Sockaddr_getFamily(exempt) == Sockaddr_AF_INET) {
        struct Prefix4* p4 = sockaddrToPrefix4(exempt, rp->alloc);
        ArrayList_OfPrefix4_add(list4, p4);
    } else if (Sockaddr_getFamily(exempt) == Sockaddr_AF_INET6) {
        struct Prefix6* p6 = sockaddrToPrefix6(exempt, rp->alloc);
        ArrayList_OfPrefix6_add(list6, p6);
    } else {
        Assert_failure("unexpected addr type");
    }
    rp->pub.hasUncommittedChanges = true;
}
```
将一些路由信息加入到localPrefixes表中。

## RouteGen_commit ##
这个函数有两个地方调用：
1. 在RouteGen_admin中注册，作为一个外部可直接调用的API，需要传入tunName。目前代码自己运行的逻辑中并未使用这种调用。
2. 在IpTunnel.c的incomingAddresses方法中调用。也就是在普通点收到离岸点分配的ip之后，会调用到该方法。

分析一下这个函数：
```
void RouteGen_commit(struct RouteGen* rg,
                     const char* tunName,
                     struct Allocator* tempAlloc,
                     struct Except* eh)
{
    struct RouteGen_pvt* rp = Identity_check((struct RouteGen_pvt*) rg);
    struct Prefix46* p46 = getGeneratedRoutes(rp, tempAlloc);
    struct Sockaddr** prefixSet =
        Allocator_calloc(tempAlloc, sizeof(char*), p46->prefix4->length + p46->prefix6->length);
    int prefixNum = 0;
    for (int i = 0; i < p46->prefix4->length; i++) {
        struct Prefix4* pfx4 = ArrayList_OfPrefix4_get(p46->prefix4, i);
        prefixSet[prefixNum++] = sockaddrForPrefix4(tempAlloc, pfx4);
    }
    for (int i = 0; i < p46->prefix6->length; i++) {
        struct Prefix6* pfx6 = ArrayList_OfPrefix6_get(p46->prefix6, i);
        prefixSet[prefixNum++] = sockaddrForPrefix6(tempAlloc, pfx6);
    }
    Assert_true(prefixNum == p46->prefix4->length + p46->prefix6->length);
    NetDev_setRoutes(tunName, prefixSet, prefixNum, rp->log, tempAlloc, eh);
    rp->pub.hasUncommittedChanges = false;
}
```
依次做了一下操作：
### 1. getGeneratedRoutes ###
```
static struct Prefix46* getGeneratedRoutes(struct RouteGen_pvt* rp, struct Allocator* alloc)
{
    struct Prefix46* out = Allocator_calloc(alloc, sizeof(struct Prefix46), 1);
    if (rp->prefixes4->length > 0) {
        out->prefix4 = genPrefixes4(rp->prefixes4, rp->exceptions4, rp->localPrefixes4, alloc);
    } else {
        out->prefix4 = ArrayList_OfPrefix4_new(alloc);
    }
    if (rp->prefixes6->length > 0) {
        out->prefix6 = genPrefixes6(rp->prefixes6, rp->exceptions6, rp->localPrefixes6, alloc);
    } else {
        out->prefix6 = ArrayList_OfPrefix6_new(alloc);
    }
    return out;
}
```
这个函数，针对ipv4和ipv6做了一样的操作：如果prefixes的length>0，调用genPrefixes方法，否则new一个对应的ArrayList。
查看一下genPrefixes方法，以genPrefixes4为例：
```
static struct ArrayList_OfPrefix4* genPrefixes4(struct ArrayList_OfPrefix4* prefixes,
                                                struct ArrayList_OfPrefix4* exceptions,
                                                struct ArrayList_OfPrefix4* localPrefixes,
                                                struct Allocator* alloc)
{
    struct Allocator* tempAlloc = Allocator_child(alloc);

    struct ArrayList_OfPrefix4* effectiveLocalPrefixes = ArrayList_OfPrefix4_new(tempAlloc);
    for (int i = 0; i < localPrefixes->length; i++) {
        bool add = true;
        struct Prefix4* localPfx = ArrayList_OfPrefix4_get(localPrefixes, i);
        for (int j = 0; j < prefixes->length; j++) {
            struct Prefix4* pfx = ArrayList_OfPrefix4_get(prefixes, j);
            if (isSubsetOf4(pfx, localPfx)) {
                add = false;
                break;
            }
        }
        if (add) {
            ArrayList_OfPrefix4_add(effectiveLocalPrefixes, localPfx);
        }
    }

    struct ArrayList_OfPrefix4* allPrefixes = ArrayList_OfPrefix4_new(tempAlloc);
    for (int i = 0; i < exceptions->length; i++) {
        struct Prefix4* pfxToInvert = ArrayList_OfPrefix4_get(exceptions, i);
        bool add = true;
        for (int j = 0; j < effectiveLocalPrefixes->length; j++) {
            struct Prefix4* localPfx = ArrayList_OfPrefix4_get(effectiveLocalPrefixes, j);
            if (isSubsetOf4(pfxToInvert, localPfx)) {
                add = false;
                break;
            }
        }
        if (add) {
            struct ArrayList_OfPrefix4* prefixes4 = invertPrefix4(pfxToInvert, tempAlloc);
            mergePrefixSets4(allPrefixes, prefixes4);
        }
    }

    for (int i = allPrefixes->length - 2; i >= 0; i--) {
        struct Prefix4* pfx = ArrayList_OfPrefix4_get(allPrefixes, i);
        struct Prefix4* pfx2 = ArrayList_OfPrefix4_get(allPrefixes, i+1);
        if (isSubsetOf4(pfx2, pfx)) {
            ArrayList_OfPrefix4_remove(allPrefixes, i+1);
            if (i < (allPrefixes->length - 2)) { i++; }
        }
    }

    for (int i = 0; i < prefixes->length; i++) {
        struct Prefix4* pfx = ArrayList_OfPrefix4_get(prefixes, i);
        int addPrefix = true;
        for (int j = allPrefixes->length - 1; j >= 0; j--) {
            struct Prefix4* pfx2 = ArrayList_OfPrefix4_get(allPrefixes, j);
            if (isSubsetOf4(pfx2, pfx)) {
                addPrefix = false;
            }
        }
        if (addPrefix) {
            ArrayList_OfPrefix4_add(allPrefixes, pfx);
        }
    }

    ArrayList_OfPrefix4_sort(allPrefixes);

    struct ArrayList_OfPrefix4* out = ArrayList_OfPrefix4_new(alloc);
    for (int i = 0; i < allPrefixes->length; i++) {
        struct Prefix4* pfx = ArrayList_OfPrefix4_get(allPrefixes, i);
        for (int j = 0; j < prefixes->length; j++) {
            struct Prefix4* pfx2 = ArrayList_OfPrefix4_get(prefixes, j);
            if (isSubsetOf4(pfx, pfx2)) {
                ArrayList_OfPrefix4_add(out, clonePrefix4(pfx, alloc));
                break;
            }
        }
    }
    Allocator_free(tempAlloc);
    return out;
}
```
这是一个重要的函数，它将prefixes,exceptions,localPrefixes三个ArrayList中的内容进行merge，共同形成一个新的ArrayList.
#### 0. 各prefixes的内容 ####
按照程序目前的运行流程，当收到离岸点分配的ipv4地址后，首次调用RouteGen_commit，进而调用到genPrefixes4，此时，三个prefixes中的内容如下：
1. rp->prefixes4 这是自己的ipv4地址，由离岸点分配得到
```
1467638405 DEBUG RouteGen.c:137 getGeneratedRoutes rp prefixes4
1467638405 DEBUG Prefix.c:112 printprefix 192.168.254.2/0
```
2. rp->localPrefixes4 这是一些local地址，在RouteGen.c中的RouteGen_new函数中，通过调用setupDefaultLocalPrefixes方法来设置。
```
1467638405 DEBUG RouteGen.c:139 getGeneratedRoutes rp localPrefixes4
1467638405 DEBUG Prefix.c:112 printprefix 10.0.0.0/8
1467638405 DEBUG Prefix.c:112 printprefix 172.16.0.0/12
1467638405 DEBUG Prefix.c:112 printprefix 192.168.0.0/16
1467638405 DEBUG Prefix.c:112 printprefix 127.0.0.0/8
```
3. rp->exceptions4 这是接入点的ipv4地址，在Configurator.c文件中，通过调用接口设置。
```
1467638405 DEBUG RouteGen.c:141 getGeneratedRoutes rp exceptions4
1467638405 DEBUG Prefix.c:112 printprefix 106.75.59.53/32
```

#### 1. prefixes和localPrefixes 进行merge ####
 先看第一块代码
```
    for (int i = 0; i < localPrefixes->length; i++) {
        bool add = true;
        struct Prefix4* localPfx = ArrayList_OfPrefix4_get(localPrefixes, i);
        for (int j = 0; j < prefixes->length; j++) {
            struct Prefix4* pfx = ArrayList_OfPrefix4_get(prefixes, j);
            if (isSubsetOf4(pfx, localPfx)) {
                add = false;
                break;
            } 
        }
        if (add) {
            ArrayList_OfPrefix4_add(effectiveLocalPrefixes, localPfx);
        }
    }
```
这一块代码，将localPrefixes和prefixes中网段依次取出，放入isSubsetOf4函数中进行检测，如果prefixes中的网段不是localPrefixes中的网段的子集，就将这条从localPrefixes中取出的网段放入effectiveLocalPrefixes中。
也就是说，一条localPrefixes中的网段想要被加入effectiveLocalPrefixes中，必须满足条件：在prefixes中没有比它更加精准的网段匹配项，或者说，没有比它范围更小的匹配项。看一下isSubsetOf4的代码
```
struct Prefix4
{
    uint32_t bits;   //addr
    int prefix;
    struct Allocator* alloc;
};
```
```
static bool isSubsetOf4(struct Prefix4* isSubset, struct Prefix4* isSuperset)
{
    if (isSuperset->prefix > isSubset->prefix) { return false; }
    if (isSuperset->prefix >= 32) {
        return isSuperset->bits == isSubset->bits;
    }
    if (!isSuperset->prefix) { return true; }
    uint32_t shift = 32 - isSuperset->prefix;
    return (isSuperset->bits >> shift) == (isSubset->bits >> shift);
}
```
可以看到，主要依靠prefix来判断。

#### 2. 与exceptions进行merge ####
```
    struct ArrayList_OfPrefix4* allPrefixes = ArrayList_OfPrefix4_new(tempAlloc);
    for (int i = 0; i < exceptions->length; i++) {
        struct Prefix4* pfxToInvert = ArrayList_OfPrefix4_get(exceptions, i);
        bool add = true;
        for (int j = 0; j < effectiveLocalPrefixes->length; j++) {
            struct Prefix4* localPfx = ArrayList_OfPrefix4_get(effectiveLocalPrefixes, j);
            if (isSubsetOf4(pfxToInvert, localPfx)) {
                add = false;
                break;
            }
        }
        if (add) {
            struct ArrayList_OfPrefix4* prefixes4 = invertPrefix4(pfxToInvert, tempAlloc);
            mergePrefixSets4(allPrefixes, prefixes4);
        }
    }
    
   for (int i = allPrefixes->length - 2; i >= 0; i--) {
        struct Prefix4* pfx = ArrayList_OfPrefix4_get(allPrefixes, i);
        struct Prefix4* pfx2 = ArrayList_OfPrefix4_get(allPrefixes, i+1);
        if (isSubsetOf4(pfx2, pfx)) {
            ArrayList_OfPrefix4_remove(allPrefixes, i+1);
            if (i < (allPrefixes->length - 2)) { i++; }
        }
    }
```
首先将exceptions和effectiveLocalPrefixes中的路由记录依次取出，放入isSubsetOf4函数中进行检测，查看exceptions中的项是否是effectiveLocalPrefixes中项的子集。
之后，不再是简单的add操作，而是invertPrefix4和mergePrefixSets4。
exception操作，是一个排除型操作，当我们往exception列表中加入一个地址时，其实是往路由表中加入掩码长度小于等于这个exception地址掩码长度的所有非此网段的地址。举个栗子：
当我们使用RouteGen_addException('193.199.1.1/16')时，其实是向路由表中加入了：
```
193.198.0.0/16
193.196.0.0/15
193.192.0.0/14
193.200.0.0/13 
193.208.0.0/12 
193.224.0.0/11 
193.128.0.0/10 
193.0.0.0/9 
192.0.0.0/8
194.0.0.0/7 
196.0.0.0/6
200.0.0.0/5 
208.0.0.0/4 
224.0.0.0/3 
128.0.0.0/2
0.0.0.0/1
```

现在来查看执行这个排除操作的函数invertPrefix4
```
static struct ArrayList_OfPrefix4* invertPrefix4(struct Prefix4* toInvert, struct Allocator* alloc)
{
    struct ArrayList_OfPrefix4* result = ArrayList_OfPrefix4_new(alloc);
    for (int i = 32 - toInvert->prefix; i < 32; i++) {
        struct Prefix4* pfx = Allocator_calloc(alloc, sizeof(struct Prefix4), 1);
        pfx->bits = ( toInvert->bits & ((uint32_t)~0 << i) ) ^ (1 << i);
        pfx->prefix = 32 - i;
        ArrayList_OfPrefix4_add(result, pfx);
    }
    return result;
}
```
可以看到，这个函数是将exceptions之外的网段加result中，result是一个ArrayList。

接下来，将这个result和之前的allPrefixes进行merge。(其实此时，allPrefixes还是空的）
```
static void mergePrefixSets4(struct ArrayList_OfPrefix4* mergeInto,
                             struct ArrayList_OfPrefix4* prefixes)
{
    struct Prefix4* highestPrefix = NULL;
    for (int j = 0; j < prefixes->length; j++) {
        struct Prefix4* result = ArrayList_OfPrefix4_get(prefixes, j);
        Assert_true(result);
        if (!highestPrefix || highestPrefix->prefix < result->prefix) {
            highestPrefix = result;
        }
    }

    struct Prefix4 target;
    Bits_memcpy(&target, highestPrefix, sizeof(struct Prefix4));
    target.bits ^= (target.prefix) ? (1 << (32 - target.prefix)) : 0;
    for (int i = mergeInto->length - 1; i >= 0; i--) {
        struct Prefix4* result = ArrayList_OfPrefix4_get(mergeInto, i);
        Assert_true(result);
        if (isSubsetOf4(&target, result)) {
            ArrayList_OfPrefix4_remove(mergeInto, i);
        }
    }

    for (int i = 0; i < prefixes->length; i++) {
        bool include = true;
        struct Prefix4* toInclude = ArrayList_OfPrefix4_get(prefixes, i);
        for (int j = 0; j < mergeInto->length; j++) {
            struct Prefix4* test = ArrayList_OfPrefix4_get(mergeInto, j);
            if (isSubsetOf4(test, toInclude)) {
                include = false;
                break;
            }
        }
        if (include) {
            ArrayList_OfPrefix4_add(mergeInto, toInclude);
        }
    }
}
```
至此，localPrefixes和exception进行merge之后的网段都加入到了allPrefixes之中。
然后在进行一次排除subset的操作
```
    
    for (int i = allPrefixes->length - 2; i >= 0; i--) {
        struct Prefix4* pfx = ArrayList_OfPrefix4_get(allPrefixes, i);
        struct Prefix4* pfx2 = ArrayList_OfPrefix4_get(allPrefixes, i+1);
        if (isSubsetOf4(pfx2, pfx)) {
            ArrayList_OfPrefix4_remove(allPrefixes, i+1);
            if (i < (allPrefixes->length - 2)) { i++; }
        }
    }
```

#### 3. prefixes与allPrefixes进行merge ####
```
    for (int i = 0; i < prefixes->length; i++) {
        struct Prefix4* pfx = ArrayList_OfPrefix4_get(prefixes, i);
        int addPrefix = true;
        for (int j = allPrefixes->length - 1; j >= 0; j--) {
            struct Prefix4* pfx2 = ArrayList_OfPrefix4_get(allPrefixes, j);
            if (isSubsetOf4(pfx2, pfx)) {
                addPrefix = false;
            }
        }
        if (addPrefix) {
            ArrayList_OfPrefix4_add(allPrefixes, pfx);
        }
    }
    
    ArrayList_OfPrefix4_sort(allPrefixes);
```
将prefixes加入到allPrefixes，之后还进行了排序。此时，allPrefixes中是localPrefixes,prefixes,exception共同merge后的结果。

#### 4. prefixes和allPrefixes共同生成out ####
```
    struct ArrayList_OfPrefix4* out = ArrayList_OfPrefix4_new(alloc);
    for (int i = 0; i < allPrefixes->length; i++) {
        struct Prefix4* pfx = ArrayList_OfPrefix4_get(allPrefixes, i);
        for (int j = 0; j < prefixes->length; j++) {
            struct Prefix4* pfx2 = ArrayList_OfPrefix4_get(prefixes, j);
            if (isSubsetOf4(pfx, pfx2)) {
                ArrayList_OfPrefix4_add(out, clonePrefix4(pfx, alloc));
                break;
            }
        }
    }
```
这里不太一样的地方在于，是比较两者，然后取子集加入到out当中。也就是说，out最后回得到一个条目更加精细的集合，该集合综合了prefixes,localPrefixes,exception的记录。

### 2. 把prefixes加入到prefixSet之中 ###
```
struct Sockaddr** prefixSet =
        Allocator_calloc(tempAlloc, sizeof(char*), p46->prefix4->length + p46->prefix6->length);
    int prefixNum = 0;
    for (int i = 0; i < p46->prefix4->length; i++) {
        struct Prefix4* pfx4 = ArrayList_OfPrefix4_get(p46->prefix4, i);
        prefixSet[prefixNum++] = sockaddrForPrefix4(tempAlloc, pfx4);
    }
```
将上一步得到的路由信息转成Sockaddr，然后加入到prefixSet当中。

### 3. NetDev_setRoutes ###
```
void NetDev_setRoutes(const char* ifName,
                      struct Sockaddr** prefixSet,
                      int prefixCount,
                      struct Log* logger,
                      struct Allocator* tempAlloc,
                      struct Except* eh)
{
    for (int i = 0; i < prefixCount; i++) {
        struct Allocator* alloc = Allocator_child(tempAlloc);
        int addrFam;
        char* printedAddr;
        void* addr;
        checkAddressAndPrefix(prefixSet[i], &addrFam, &printedAddr, &addr, alloc, eh);
        Allocator_free(alloc);
    }

    NetPlatform_setRoutes(ifName, prefixSet, prefixCount, logger, tempAlloc, eh);
}
```
```
void NetPlatform_setRoutes(const char* ifName,
                           struct Sockaddr** prefixSet,
                           int prefixCount,
                           struct Log* logger,
                           struct Allocator* tempAlloc,
                           struct Except* eh)
{
    int ifIndex = ifIndexForName(ifName, eh);
    struct RouteInfo* newRi = riForSockaddrs(prefixSet, prefixCount, ifIndex, tempAlloc);
    int sock = mkSocket(tempAlloc, eh);
    struct RouteInfo* oldRi = getRoutes(sock, ifIndex, tempAlloc, eh);
    logRis(oldRi, logger, "DELETE ROUTE");
    addDeleteRoutes(sock, true, oldRi, tempAlloc, eh);
    logRis(newRi, logger, "ADD ROUTE");
    addDeleteRoutes(sock, false, newRi, tempAlloc, eh);
}
```
实际设置路由的过程，在/util/platform/netdev/文件夹下，不做详细分析。

## RouteGen_admin ##
这个admin文件，写的还是跟别的不太一样的，用到了很多的宏来简化重复代码。
```
void RouteGen_admin_register(struct RouteGen* rg, struct Admin* admin, struct Allocator* alloc)
{
    struct RouteGen_admin_Ctx* ctx = Allocator_calloc(alloc, sizeof(struct RouteGen_admin_Ctx), 1);
    ctx->rg = rg;
    ctx->admin = admin;
    Identity_set(ctx);

    REGISTER_GET_SOMETHING(getPrefixes, ctx, admin);
    REGISTER_GET_SOMETHING(getLocalPrefixes, ctx, admin);
    REGISTER_GET_SOMETHING(getExceptions, ctx, admin);
    REGISTER_GET_SOMETHING(getGeneratedRoutes, ctx, admin);

    REGISTER_ADD_REMOVE_SOMETHING(addException, ctx, admin);
    REGISTER_ADD_REMOVE_SOMETHING(addPrefix, ctx, admin);
    REGISTER_ADD_REMOVE_SOMETHING(addLocalPrefix, ctx, admin);
    REGISTER_ADD_REMOVE_SOMETHING(removePrefix, ctx, admin);
    REGISTER_ADD_REMOVE_SOMETHING(removeLocalPrefix, ctx, admin);
    REGISTER_ADD_REMOVE_SOMETHING(removeException, ctx, admin);

    Admin_registerFunction("RouteGen_commit", commit, ctx, true,
        ((struct Admin_FunctionArg[]) {
            { .name = "tunName", .required = 1, .type = "String" },
        }), admin);
}
```
除了RouteGen_commit直接写了registerfunction方法，其他都使用了宏定义函数。


###  REGISTER_GET_SOMETHING ###
```
#define REGISTER_GET_SOMETHING(_name, ctx, admin) \
    Admin_registerFunction("RouteGen_" #_name, _name, ctx, true,                                \
        ((struct Admin_FunctionArg[]) {                                                         \
            { .name = "page", .required = 0, .type = "Int" },                                   \
            { .name = "ip6", .required = 0, .type = "Int" }                                     \
        }), admin)
```
单井号就是将后面的 宏参数 进行字符串操作，就是将后面的参数用双引号引起来。
假设name为getGeneratedRoutes，则等同于
```
    Admin_registerFunction("RouteGen_" "getGeneratedRoutes", getGeneratedRoutes, ctx, true,                                \
        ((struct Admin_FunctionArg[]) {                                                         \
            { .name = "page", .required = 0, .type = "Int" },                                   \
            { .name = "ip6", .required = 0, .type = "Int" }                                     \
        }), admin)
```
这样，RouteGen_getGeneratedRoutes就被register到getGeneratedRoutes函数上。这一类getxxxx的函数，也使用了宏定义函数，看下这类函数的定义：
```
#define GET_SOMETHING(name) \
    static void name(Dict* args, void* vcontext, String* txid, struct Allocator* requestAlloc)  \
    {                                                                                           \
        struct RouteGen_admin_Ctx* ctx = Identity_check((struct RouteGen_admin_Ctx*) vcontext); \
        Dict* genRoutes = RouteGen_ ## name (ctx->rg, requestAlloc);                            \
        getSomething(args, ctx, txid, requestAlloc, genRoutes);                                 \
    }
GET_SOMETHING(getPrefixes)
GET_SOMETHING(getLocalPrefixes)
GET_SOMETHING(getExceptions)
GET_SOMETHING(getGeneratedRoutes)
```
依然以getGeneratedRoutes为例，得到一个getGeneratedRoutes函数(宏定义函数中的双井号就是用于连接)
```
    static void getGeneratedRoutes(Dict* args, void* vcontext, String* txid, struct Allocator* requestAlloc) 
    {
        struct RouteGen_admin_Ctx* ctx = Identity_check((struct RouteGen_admin_Ctx*) vcontext); 
        Dict* genRoutes = RouteGen_getGeneratedRoutes (ctx->rg, requestAlloc);                            
        getSomething(args, ctx, txid, requestAlloc, genRoutes);                                 
    }
```
这里用到了两个函数：
#### RouteGen_getGeneratedRoutes ####
```
Dict* RouteGen_getGeneratedRoutes(struct RouteGen* rg, struct Allocator* alloc)
{
    struct RouteGen_pvt* rp = Identity_check((struct RouteGen_pvt*) rg);
    struct Prefix46* p46 = getGeneratedRoutes(rp, alloc);
    return getSomething(rp, alloc, p46->prefix6, p46->prefix4);
}
```
1. getGeneratedRoutes(rp, alloc)
```
static struct Prefix46* getGeneratedRoutes(struct RouteGen_pvt* rp, struct Allocator* alloc)
{
    struct Prefix46* out = Allocator_calloc(alloc, sizeof(struct Prefix46), 1);
    if (rp->prefixes4->length > 0) {
        out->prefix4 = genPrefixes4(rp->prefixes4, rp->exceptions4, rp->localPrefixes4, alloc);
    } else {
        out->prefix4 = ArrayList_OfPrefix4_new(alloc);
    }
    if (rp->prefixes6->length > 0) {
        out->prefix6 = genPrefixes6(rp->prefixes6, rp->exceptions6, rp->localPrefixes6, alloc);
    } else {
        out->prefix6 = ArrayList_OfPrefix6_new(alloc);
    }
    return out;
}
```
上面已经做过分析。

2.  getSomething(rp, alloc, p46->prefix6, p46->prefix4）
注意，这个getSomething是RouteGen.c中的私有方法。
```
static Dict* getSomething(struct RouteGen_pvt* rp,
                          struct Allocator* alloc,
                          struct ArrayList_OfPrefix6* list6,
                          struct ArrayList_OfPrefix4* list4)
{
    ArrayList_OfPrefix6_sort(list6);
    ArrayList_OfPrefix4_sort(list4);
    List* prefixes4 = List_new(alloc);
    for (int i = 0; i < list4->length; i++) {
        struct Prefix4* pfx4 = ArrayList_OfPrefix4_get(list4, i);
        List_addString(prefixes4, printPrefix4(alloc, pfx4), alloc);
    }
    List* prefixes6 = List_new(alloc);
    for (int i = 0; i < list6->length; i++) {
        struct Prefix6* pfx6 = ArrayList_OfPrefix6_get(list6, i);
        List_addString(prefixes6, printPrefix6(alloc, pfx6), alloc);
    }
    Dict* out = Dict_new(alloc);
    Dict_putList(out, String_new("ipv4", alloc), prefixes4, alloc);
    Dict_putList(out, String_new("ipv6", alloc), prefixes6, alloc);
    return out;
}
```
out中包括两个Dict，一个记录ipv4，一个记录ipv6

#### getSomething ####
注意，这个getSomething是RouteGen_admin.c中的私有方法。
```
static void getSomething(Dict* args,
                         struct RouteGen_admin_Ctx* ctx,
                         String* txid,
                         struct Allocator* requestAlloc,
                         Dict* genRoutes)
{
    int page = getIntVal(args, String_CONST("page"));
    List* routes;
    if (getIntVal(args, String_CONST("ip6"))) {
        routes = Dict_getListC(genRoutes, "ipv6");
    } else {
        routes = Dict_getListC(genRoutes, "ipv4");
    }
    Assert_true(routes);
    List* outList = List_new(requestAlloc);
    bool more = false;
    for (int i = page * ROUTES_PER_PAGE, j = 0; i < List_size(routes) && j < ROUTES_PER_PAGE; j++) {
        String* route = List_getString(routes, i);
        Assert_true(route);
        List_addString(outList, route, requestAlloc);
        if (++i >= List_size(routes)) {
            more = false;
            break;
        }
        more = true;
    }
    Dict* out = Dict_new(requestAlloc);
    if (more) {
        Dict_putInt(out, String_new("more", requestAlloc), 1, requestAlloc);
    }
    Dict_putList(out, String_new("routes", requestAlloc), outList, requestAlloc);
    Admin_sendMessage(out, txid, ctx->admin);
}
```
这里主要是根据请求中带的是ip6还是ip4来区分需要返回哪一组数据。

### REGISTER_ADD_REMOVE_SOMETHING ###
```
enum addRemoveSomething_What {
    addRemoveSomething_What_ADD_EXCEPTION,
    addRemoveSomething_What_RM_EXCEPTION,
    addRemoveSomething_What_ADD_PREFIX,
    addRemoveSomething_What_RM_PREFIX,
    addRemoveSomething_What_ADD_LOCALPREFIX,
    addRemoveSomething_What_RM_LOCALPREFIX,
};
static void addRemoveSomething(Dict* args,
                               void* vcontext,
                               String* txid,
                               struct Allocator* requestAlloc,
                               enum addRemoveSomething_What what)
{
    struct RouteGen_admin_Ctx* ctx = Identity_check((struct RouteGen_admin_Ctx*) vcontext);
    String* route = Dict_getStringC(args, "route");
    char* error = NULL;

    struct Sockaddr_storage ss;
    if (route->len > 63) {
        error = "parse_failed";
    }
    if (!error) {
        if (Sockaddr_parse(route->bytes, &ss)) {
            error = "parse_failed";
        } else {
            int family = Sockaddr_getFamily(&ss.addr);
            if (family != Sockaddr_AF_INET && family != Sockaddr_AF_INET6) {
                error = "unexpected_af";
            }
        }
    }
    int retVal = -1;
    Dict* out = Dict_new(requestAlloc);
    if (!error) {
        switch (what) {
            case addRemoveSomething_What_ADD_EXCEPTION:
                RouteGen_addException(ctx->rg, &ss.addr); break;
            case addRemoveSomething_What_ADD_PREFIX:
                RouteGen_addPrefix(ctx->rg, &ss.addr); break;
            case addRemoveSomething_What_ADD_LOCALPREFIX:
                RouteGen_addLocalPrefix(ctx->rg, &ss.addr); break;
            case addRemoveSomething_What_RM_EXCEPTION:
                retVal = RouteGen_removeException(ctx->rg, &ss.addr); break;
            case addRemoveSomething_What_RM_PREFIX:
                retVal = RouteGen_removePrefix(ctx->rg, &ss.addr); break;
            case addRemoveSomething_What_RM_LOCALPREFIX:
                retVal = RouteGen_removeLocalPrefix(ctx->rg, &ss.addr); break;
            default: Assert_failure("invalid op");
        }
        if (!retVal) {
            error = "no_such_route";
        } else {
            error = "none";
        }
    }
    Dict_putString(out,
                   String_new("error", requestAlloc),
                   String_new(error, requestAlloc),
                   requestAlloc);
    Admin_sendMessage(out, txid, ctx->admin);
}

#define ADD_REMOVE_SOMETHING(name, op) \
    static void name(Dict* args, void* vcontext, String* txid, struct Allocator* requestAlloc)  \
    {                                                                                           \
        addRemoveSomething(args, vcontext, txid, requestAlloc, op);                             \
    }
ADD_REMOVE_SOMETHING(addException, addRemoveSomething_What_ADD_EXCEPTION)
ADD_REMOVE_SOMETHING(addPrefix, addRemoveSomething_What_ADD_PREFIX)
ADD_REMOVE_SOMETHING(addLocalPrefix, addRemoveSomething_What_ADD_LOCALPREFIX)
ADD_REMOVE_SOMETHING(removePrefix, addRemoveSomething_What_RM_PREFIX)
ADD_REMOVE_SOMETHING(removeLocalPrefix, addRemoveSomething_What_RM_LOCALPREFIX)
ADD_REMOVE_SOMETHING(removeException, addRemoveSomething_What_RM_EXCEPTION)
#define REGISTER_ADD_REMOVE_SOMETHING(_name, ctx, admin) \
    Admin_registerFunction("RouteGen_" #_name, _name, ctx, true,                                \
        ((struct Admin_FunctionArg[]) {                                                         \
            { .name = "route", .required = 1, .type = "String" },                               \
        }), admin)
```
相比GET系列，ADD_REMOVE系列简单很多。以RouteGen_addException为例：
```

static void addSomething(struct RouteGen_pvt* rp,
                         struct Sockaddr* exempt,
                         struct ArrayList_OfPrefix6* list6,
                         struct ArrayList_OfPrefix4* list4)
{
    if (Sockaddr_getFamily(exempt) == Sockaddr_AF_INET) {
        struct Prefix4* p4 = sockaddrToPrefix4(exempt, rp->alloc);
        ArrayList_OfPrefix4_add(list4, p4);
    } else if (Sockaddr_getFamily(exempt) == Sockaddr_AF_INET6) {
        struct Prefix6* p6 = sockaddrToPrefix6(exempt, rp->alloc);
        ArrayList_OfPrefix6_add(list6, p6);
    } else {
        Assert_failure("unexpected addr type");
    }
    rp->pub.hasUncommittedChanges = true;
}

void RouteGen_addException(struct RouteGen* rg, struct Sockaddr* destination)
{
    struct RouteGen_pvt* rp = Identity_check((struct RouteGen_pvt*) rg);
    addSomething(rp, destination, rp->exceptions6, rp->exceptions4);
}
```
几乎不需要分析。

## 源码中对这几个ArrayList的操作 ##
1. IpTunnel.c的addAddress中
```
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
```

2. RouteGen_new中调用setupDefaultLocalPrefixes
```
static void setupDefaultLocalPrefixes(struct RouteGen_pvt* rp)
{
    struct Sockaddr_storage ss;
    #define ADD_PREFIX(str) \
        Assert_true(!Sockaddr_parse(str, &ss));       \
        RouteGen_addLocalPrefix(&rp->pub, &ss.addr)

    ADD_PREFIX("fe80::/10");
    ADD_PREFIX("fd00::/8");

    ADD_PREFIX("10.0.0.0/8");
    ADD_PREFIX("172.16.0.0/12");
    ADD_PREFIX("192.168.0.0/16");
    ADD_PREFIX("127.0.0.0/8");

    #undef ADD_PREFIX
}
```

3. Configurator.c中设置exception
解析conf文件中关于设置inbound的配置。配置中包括inbound的ipv4地址，调用'RouteGen_addException'，将这个地址添加到exception当中。
conf文件
```
"connectTo":{
	"106.75.59.53:50001":{
    	"password":"XRuMWpgvNienefc7ZT8gXTuTCvSWWSA",
    	"publicKey":"nsn1nz93lztkg6zbw4yqnr6zlzxhchppzdkduwh9wh79p88fwx60.k"
    }
},
```
Configurator.c中对conf的解析操作。
```
        Dict* connectTo = Dict_getDictC(udp, "connectTo");
        if (connectTo) {
            struct Dict_Entry* entry = *connectTo;
            struct Allocator* perCallAlloc = Allocator_child(ctx->alloc);
            while (entry != NULL) {
                String* key = (String*) entry->key;
                if (entry->val->type != Object_DICT) {
                    Log_critical(ctx->logger, "interfaces.UDPInterface.connectTo: entry [%s] "
                                               "is not a dictionary type.", key->bytes);
                    exit(-1);
                }
                Dict* value = entry->val->as.dictionary;
                Log_keys(ctx->logger, "Attempting to connect to node [%s].", key->bytes);
                key = String_clone(key, perCallAlloc);
                char* lastColon = CString_strrchr(key->bytes, ':');

                if (lastColon) {
                    if (!Sockaddr_parse(key->bytes, NULL)) {
                        // it's a sockaddr, fall through
                    } else {
                        // try it as a hostname.
                        Log_critical(ctx->logger, "Couldn't add connection [%s], "
                                                    "hostnames aren't supported.", key->bytes);
                        exit(-1);
                    }
                } else {
                    // it doesn't have a port
                    Log_critical(ctx->logger, "Connection [%s] must be $IP:$PORT, or "
                                                "[$IP]:$PORT for IPv6.", key->bytes);
                    exit(-1);
                }
                Dict_putIntC(value, "interfaceNumber", ifNum, perCallAlloc);
                Dict_putStringC(value, "address", key, perCallAlloc);
                rpcCall(String_CONST("UDPInterface_beginConnection"), value, ctx, perCallAlloc);

                // Make a IPTunnel exception for this node
                Dict* aed = Dict_new(perCallAlloc);
                *lastColon = '\0';
                Dict_putStringC(aed, "route", String_new(key->bytes, perCallAlloc),
                    perCallAlloc);
                *lastColon = ':';
                rpcCall(String_CONST("RouteGen_addException"), aed, ctx, perCallAlloc);

                entry = entry->next;
            }
            Allocator_free(perCallAlloc);
        }
```

