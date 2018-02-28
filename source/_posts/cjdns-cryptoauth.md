---
title: cjdns源码分析--CryptoAuth
category:
  - cjdns
  - cjdns源码分析
tags:
  - cjdns
  - cjdns源码分析
  - C/C++
description: "介绍CryptoAuth相关源码。包括conf中的配置，相关API，建立CryptoAuth_Session时的ca验证"
date: 2017-09-05 12:48:21
---
## 接入点conf分析 ##
```
	"authorizedPasswords":[
		{
			"user":"a",
			"password":"R9q7eEn3i9YTOB0zITKLMBpwEghxPH6",
            "ipv6":"fbdb:b56d:fdcf:c2ae:5f0b:0cdd:67b4:8f8b"
		},
		{
			"user":"a",
			"password":"R9q7eEn3i9YTOB0zITKLMBpwEghxPH7"
		},
		{
			"user":"b",
			"password":"R9q7eEn3i9YTOB0zITKLMBpwEghxPH7"
		},
		{
			"user":"c",
			"password":"R9q7eEn3i9YTOB0zITKLMBpwEghxPH8"
		},
		{
			"password":"R9q7eEn3i9YTOB0zITKLMBpwEghxPH9"
		},
		{
			"user":"a",
			"password":"R9q7eEn3i9YTOB0zITKLMBpwEghxPHa"
		},
	],
```
接入点的conf中可以多个authorizedPasswords，每个authorizedPasswords必须有password,可以有user和ipv6.其中，password和user都是其他点连接过来时，用来验证身份的。ipv6是允许来连接我的点的ipv6.
从这个conf中可以看出，user，password，ipv6都是可以重复的。

## 对conf的解析 ##
```
static void authorizedPasswords(List* list, struct Context* ctx)
{
    uint32_t count = List_size(list);
    for (uint32_t i = 0; i < count; i++) {
        Dict* d = List_getDict(list, i);
        Log_info(ctx->logger, "Checking authorized password %d.", i);
        if (!d) {
            Log_critical(ctx->logger, "Not a dictionary type %d.", i);
            exit(-1);
        }
        String* passwd = Dict_getStringC(d, "password");
        if (!passwd) {
            Log_critical(ctx->logger, "Must specify a password %d.", i);
            exit(-1);
        }
    }

    for (uint32_t i = 0; i < count; i++) {
        struct Allocator* child = Allocator_child(ctx->alloc);
        Dict* d = List_getDict(list, i);
        String* passwd = Dict_getStringC(d, "password");
        String* user = Dict_getStringC(d, "user");
        String* displayName = user;
        if (!displayName) {
            displayName = String_printf(child, "password [%d]", i);
        }
        //String* publicKey = Dict_getStringC(d, "publicKey");
        String* ipv6 = Dict_getStringC(d, "ipv6");
        Log_info(ctx->logger, "Adding authorized password #[%d] for user [%s].",
            i, displayName->bytes);
        Dict *args = Dict_new(child);
        uint32_t i = 1;
        Dict_putIntC(args, "authType", i, child);
        Dict_putStringC(args, "password", passwd, child);
        if (user) {
            Dict_putStringC(args, "user", user, child);
        }
        Dict_putStringC(args, "displayName", displayName, child);
        if (ipv6) {
            Log_info(ctx->logger,
                "  This connection password restricted to [%s] only.", ipv6->bytes);
            Dict_putStringC(args, "ipv6", ipv6, child);
        }
        rpcCall(String_CONST("add_pass"), args, ctx, child);
        Allocator_free(child);
    }
}
```
* 第一个for循环中，检查了一下格式，以及必须有password。
* 第二个for循环中，获取各项参数，然后put到Dict中，并通过rpcCall调用add_pass。
	可以看到，put的参数除了从conf中取到的password,user,ipv6之外，还有authType,displayName,不过没什么用，因为add_pass中并没有处理这些值。

## add_pass方法的分析 ##
admin/AuthorizedPasswords.c
```
void AuthorizedPasswords_init(struct Admin* admin,
                              struct CryptoAuth* ca,
                              struct Allocator* allocator)
{
...
    Admin_registerFunction("add_pass", add, context, true,
        ((struct Admin_FunctionArg[]){
            { .name = "password", .required = 1, .type = "String" },
            { .name = "ipv6", .required = 0, .type = "String" },
            { .name = "user", .required = 0, .type = "String" }
        }), admin);
...
}
```
从requited的值可以看出，只有password是必须的
```
static void add(Dict* args, void* vcontext, String* txid, struct Allocator* alloc)
{
    struct Context* context = Identity_check((struct Context*) vcontext);

    String* passwd = Dict_getStringC(args, "password");
    String* user = Dict_getStringC(args, "user");
    String* ipv6 = Dict_getStringC(args, "ipv6");

    uint8_t ipv6Bytes[16];
    uint8_t* ipv6Arg;
    if (!ipv6) {
        ipv6Arg = NULL;
    } else if (AddrTools_parseIp(ipv6Bytes, ipv6->bytes)) {
        sendResponse(String_CONST("Invalid IPv6 Address"), context->admin, txid, alloc);
        return;
    } else {
        ipv6Arg = ipv6Bytes;
    }

    int32_t ret = CryptoAuth_addUser_ipv6(passwd, user, ipv6Arg, context->ca);

    switch (ret) {
        case 0:
            sendResponse(String_CONST("none"), context->admin, txid, alloc);
            break;
        case CryptoAuth_addUser_DUPLICATE:
            sendResponse(String_CONST("Password already added."), context->admin, txid, alloc);
            break;
        default:
            sendResponse(String_CONST("Unknown error."), context->admin, txid, alloc);
    }
}
```
调用CryptoAuth_addUser_ipv6,在CryptoAuth.c中
```
int CryptoAuth_addUser_ipv6(String* password,
                            String* login,
                            uint8_t ipv6[16],
                            struct CryptoAuth* cryptoAuth)
{
    struct CryptoAuth_pvt* ca = Identity_check((struct CryptoAuth_pvt*) cryptoAuth);

    struct Allocator* alloc = Allocator_child(ca->allocator);
    struct CryptoAuth_User* user = Allocator_calloc(alloc, sizeof(struct CryptoAuth_User), 1);
    user->alloc = alloc;
    Identity_set(user);

    if (!login) {
        int i = 0;
        for (struct CryptoAuth_User* u = ca->users; u; u = u->next) { i++; }
        user->login = login = String_printf(alloc, "Anon #%d", i);
    } else {
        user->login = String_clone(login, alloc);
    }

    struct CryptoHeader_Challenge ac;
    // Users specified with a login field might want to use authType 1 still.
    hashPassword(user->secret, &ac, login, password, 2);
    Bits_memcpy(user->userNameHash, &ac, CryptoHeader_Challenge_KEYSIZE);
    hashPassword(user->secret, &ac, NULL, password, 1);
    Bits_memcpy(user->passwordHash, &ac, CryptoHeader_Challenge_KEYSIZE);

    for (struct CryptoAuth_User* u = ca->users; u; u = u->next) {
        if (Bits_memcmp(user->secret, u->secret, 32)) {
        } else if (!login) {
        } else if (String_equals(login, u->login)) {
            Allocator_free(alloc);
            return CryptoAuth_addUser_DUPLICATE;
        }
    }

    if (ipv6) {
        Bits_memcpy(user->restrictedToip6, ipv6, 16);
    }

    // Add the user to the *end* of the list
    for (struct CryptoAuth_User** up = &ca->users; ; up = &(*up)->next) {
        if (!*up) {
            *up = user;
            break;
        }
    }

    return 0;
}
```
按照空行分成六块，逐块分析：

1. 结构体CryptoAuth_User的初始化
```
    struct CryptoAuth_pvt* ca = Identity_check((struct CryptoAuth_pvt*) cryptoAuth);

    struct Allocator* alloc = Allocator_child(ca->allocator);
    struct CryptoAuth_User* user = Allocator_calloc(alloc, sizeof(struct CryptoAuth_User), 1);
    user->alloc = alloc;
    Identity_set(user);
```

2. 设置login
```
    if (!login) {
        int i = 0;
        for (struct CryptoAuth_User* u = ca->users; u; u = u->next) { i++; }
        user->login = login = String_printf(alloc, "Anon #%d", i);
    } else {
        user->login = String_clone(login, alloc);
    }
```
如果conf中有user，使用它当作login，否则自动生成一个“Anon”开头的login

3. 计算secret,userNameHash,passwordHash,并赋值
```
    struct CryptoHeader_Challenge ac;
    // Users specified with a login field might want to use authType 1 still.
    hashPassword(user->secret, &ac, login, password, 2);
    Bits_memcpy(user->userNameHash, &ac, CryptoHeader_Challenge_KEYSIZE);
    hashPassword(user->secret, &ac, NULL, password, 1);
    Bits_memcpy(user->passwordHash, &ac, CryptoHeader_Challenge_KEYSIZE);
```

4. 检查合法性
```
	for (struct CryptoAuth_User* u = ca->users; u; u = u->next) {
        if (Bits_memcmp(user->secret, u->secret, 32)) {
        } else if (!login) {
        } else if (String_equals(login, u->login)) {
            Allocator_free(alloc);
            return CryptoAuth_addUser_DUPLICATE;
        }
    }
```
当user和password同时与已有的某个user相同时，会报错：CryptoAuth_addUser_DUPLICATE

5. 如果有ipv6，设置到restrictedToip6中
```
    if (ipv6) {
        Bits_memcpy(user->restrictedToip6, ipv6, 16);
    }
```
这个设置可以限制只有某个ipv6才能使用这个user进行验证。也就是说，这个ipv6是连接过来的普通点的ipv6。

6. 把这个CryptoAuth_User加到列表的最后
```
    // Add the user to the *end* of the list
    for (struct CryptoAuth_User** up = &ca->users; ; up = &(*up)->next) {
        if (!*up) {
            *up = user;
            break;
        }
    }
```

## 普通点conf分析 ##
```
"connectTo":{
	"192.168.2.43:26808":{
    	"login":"c",
        "password":"R9q7eEn3i9YTOB0zITKLMBpwEghxPH6",
        "publicKey":"cy4cmzzry3yykwblnh402vmyt328pu9nm58rx4cbgnqllv6hvv70.k"
    }
}
```
* "192.168.2.43:26808":要连接的点的ipv4地址和端口
* "publicKey":要连接的点的publickey
* "password":要连接的点配置的password
* "login"：要连接的点配置的login，在连接点的conf中，是user字段

## 普通点配置连接点的分析 ##
从Configurator.c中通过rpcCall调用udp_conn。
直接从udp_conn开始分析
interface/UDPInterface_admin.c中
```
    Admin_registerFunction("udp_conn", beginConnection, ctx, true,
        ((struct Admin_FunctionArg[]) {
            { .name = "interfaceNumber", .required = 0, .type = "Int" },
            { .name = "password", .required = 0, .type = "String" },
            { .name = "publicKey", .required = 1, .type = "String" },
            { .name = "address", .required = 1, .type = "String" },
            { .name = "login", .required = 0, .type = "String" }
        }), admin);
```
interfaceNumber并未在conf中出现
password是conf中的password
publicKey是conf中的publicKey
address是conf中的"192.168.2.43:26808"
login是conf中的login
可以看出，只有publicKey和address是必须的。
```
static void beginConnection(Dict* args,
                            void* vcontext,
                            String* txid,
                            struct Allocator* requestAlloc)
{
    struct Context* ctx = vcontext;

    String* password = Dict_getStringC(args, "password");
    String* login = Dict_getStringC(args, "login");
    String* publicKey = Dict_getStringC(args, "publicKey");
    String* address = Dict_getStringC(args, "address");
    int64_t* interfaceNumber = Dict_getIntC(args, "interfaceNumber");
    uint32_t ifNum = (interfaceNumber) ? ((uint32_t) *interfaceNumber) : 0;
    String* peerName = Dict_getStringC(args, "peerName");
    String* error = NULL;

    Log_debug(ctx->logger, "Peering with [%s]", publicKey->bytes);

    struct Sockaddr_storage ss;
    uint8_t pkBytes[32];
    int ret;
    if (interfaceNumber && *interfaceNumber < 0) {
        error = String_CONST("negative interfaceNumber");

    } else if ((ret = Key_parse(publicKey, pkBytes, NULL))) {
        error = String_CONST(Key_parse_strerror(ret));

    } else if (Sockaddr_parse(address->bytes, &ss)) {
        error = String_CONST("unable to parse ip address and port.");

    } else if (Sockaddr_getFamily(&ss.addr) != Sockaddr_getFamily(ctx->udpIf->addr)) {
        error = String_CONST("different address type than this socket is bound to.");

    } else {

        struct Sockaddr* addr = &ss.addr;
        char* addrPtr = NULL;
        int addrLen = Sockaddr_getAddress(&ss.addr, &addrPtr);
        Assert_true(addrLen > 0);
        struct Allocator* tempAlloc = Allocator_child(ctx->alloc);
        if (Bits_isZero(addrPtr, addrLen)) {
            // unspec'd address, convert to loopback
            if (Sockaddr_getFamily(addr) == Sockaddr_AF_INET) {
                addr = Sockaddr_clone(Sockaddr_LOOPBACK, tempAlloc);
            } else if (Sockaddr_getFamily(addr) == Sockaddr_AF_INET6) {
                addr = Sockaddr_clone(Sockaddr_LOOPBACK6, tempAlloc);
            } else {
                Assert_failure("Sockaddr which is not AF_INET nor AF_INET6");
            }
            Sockaddr_setPort(addr, Sockaddr_getPort(&ss.addr));
        }

        int ret = InterfaceController_bootstrapPeer(
            ctx->ic, ifNum, pkBytes, addr, password, login, peerName, ctx->alloc);

        Allocator_free(tempAlloc);

        if (ret) {
            switch(ret) {
                case InterfaceController_bootstrapPeer_BAD_IFNUM:
                    error = String_CONST("no such interface for interfaceNumber");
                    break;

                case InterfaceController_bootstrapPeer_BAD_KEY:
                    error = String_CONST("invalid public key.");
                    break;

                case InterfaceController_bootstrapPeer_OUT_OF_SPACE:
                    error = String_CONST("no more space to register with the switch.");
                    break;

                default:
                    error = String_CONST("unknown error");
                    break;
            }
        } else {
            error = String_CONST("none");
        }
    }

    Dict out = Dict_CONST(String_CONST("error"), String_OBJ(error), NULL);
    Admin_sendMessage(&out, txid, ctx->admin);
}
```
首先看从dict中取出的值：
```
    String* password = Dict_getStringC(args, "password");
    String* login = Dict_getStringC(args, "login");
    String* publicKey = Dict_getStringC(args, "publicKey");
    String* address = Dict_getStringC(args, "address");
    int64_t* interfaceNumber = Dict_getIntC(args, "interfaceNumber");
    uint32_t ifNum = (interfaceNumber) ? ((uint32_t) *interfaceNumber) : 0;
    String* peerName = Dict_getStringC(args, "peerName");
```
password,login,publicKey,address是conf里面写了的
interfaceNumber,peerName根本没有，也就是为空。

之后用if else做了一些参数合法性检查。
最有一个else开始配置连接点的操作。
```
        struct Sockaddr* addr = &ss.addr;
        char* addrPtr = NULL;
        int addrLen = Sockaddr_getAddress(&ss.addr, &addrPtr);
        Assert_true(addrLen > 0);
        struct Allocator* tempAlloc = Allocator_child(ctx->alloc);
        if (Bits_isZero(addrPtr, addrLen)) {
            // unspec'd address, convert to loopback
            if (Sockaddr_getFamily(addr) == Sockaddr_AF_INET) {
                addr = Sockaddr_clone(Sockaddr_LOOPBACK, tempAlloc);
            } else if (Sockaddr_getFamily(addr) == Sockaddr_AF_INET6) {
                addr = Sockaddr_clone(Sockaddr_LOOPBACK6, tempAlloc);
            } else {
                Assert_failure("Sockaddr which is not AF_INET nor AF_INET6");
            }
            Sockaddr_setPort(addr, Sockaddr_getPort(&ss.addr));
        }

        int ret = InterfaceController_bootstrapPeer(
            ctx->ic, ifNum, pkBytes, addr, password, login, peerName, ctx->alloc);

        Allocator_free(tempAlloc);

        if (ret) {
            switch(ret) {
                case InterfaceController_bootstrapPeer_BAD_IFNUM:
                    error = String_CONST("no such interface for interfaceNumber");
                    break;

                case InterfaceController_bootstrapPeer_BAD_KEY:
                    error = String_CONST("invalid public key.");
                    break;

                case InterfaceController_bootstrapPeer_OUT_OF_SPACE:
                    error = String_CONST("no more space to register with the switch.");
                    break;

                default:
                    error = String_CONST("unknown error");
                    break;
            }
        } else {
            error = String_CONST("none");
        }
    }
```
核心代码：
```
        int ret = InterfaceController_bootstrapPeer(
            ctx->ic, ifNum, pkBytes, addr, password, login, peerName, ctx->alloc);
```
查看这个方法,在net/InterfaceController.c,只看和CryptoAuth相关的部分：
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
    ep->caSession = CryptoAuth_newSession(ic->ca, epAlloc, herPublicKey, false, "outer");
    CryptoAuth_setAuth(password, login, ep->caSession);
    if (user) {
        ep->caSession->displayName = String_clone(user, epAlloc);
    }
......

    // We can't just add the node directly to the routing table because we do not know
    // the version. We'll send it a switch ping and when it responds, we will know it's
    // key (if we don't already) and version number.
    sendPing(ep);

    return 0;
}
```
1. CryptoAuth_newSession并没有设置password
2. 调用了CryptoAuth_setAuth
3. user为空，所以没有设置ep->caSession->displayName
4. 最后sendPing连接点。
主要看一下CryptoAuth_setAuth,在crypto/CryptoAuth.c
```
void CryptoAuth_setAuth(const String* password,
                        const String* login,
                        struct CryptoAuth_Session* caSession)
{
    struct CryptoAuth_Session_pvt* session =
        Identity_check((struct CryptoAuth_Session_pvt*)caSession);

    if (!password && (session->password || session->authType)) {
        session->password = NULL;
        session->authType = 0;
    } else if (!session->password || !String_equals(session->password, password)) {
        session->password = String_clone(password, session->alloc);
        session->authType = 1;
        if (login) {
            session->authType = 2;
            session->login = String_clone(login, session->alloc);
        }
    } else {
        return;
    }
    reset(session);
}
```
会进入到
```
	else if (!session->password || !String_equals(session->password, password)) {
        session->password = String_clone(password, session->alloc);
        session->authType = 1;
        if (login) {
            session->authType = 2;
            session->login = String_clone(login, session->alloc);
        }
    }
```
1. 将password设置到session->password
2. 如果有login，authType为2，否则为1.也就是说，如果conf中有login（对应着连接点conf中的user字段），使用验证login的方法，否则使用验证password的方法。这个区分，在后面会具体分析。
3. 如果有login，login设置到session->login

## 连接点验证一个新来的点的身份是否合格 ##
验证的核心方法是crypto/AryptoAuth.c中的一个内部方法getAuth.
调用链为：net/InterfaceController.c的handleUnexpectedIncoming
--> net/InterfaceController.c的CADecryptAndNotify
--> crypto/AryptoAuth.c的CryptoAuth_getAuth
--> crypto/AryptoAuth.c的getAuth
```
static inline struct CryptoAuth_User* getAuth(struct CryptoHeader_Challenge* auth,
                                              struct CryptoAuth_pvt* ca)
{
    if (auth->type == 0) {
        return NULL;
    }
    int count = 0;

    for (struct CryptoAuth_User* u = ca->users; u; u = u->next) {
        count++;
        if (auth->type == 1 &&
            !Bits_memcmp(auth, u->passwordHash, CryptoHeader_Challenge_KEYSIZE))
        {
            return u;
        } else if (auth->type == 2 &&
            !Bits_memcmp(auth, u->userNameHash, CryptoHeader_Challenge_KEYSIZE))
        {
            return u;
        }
    }
    Log_debug(ca->logger, "Got unrecognized auth, password count = [%d]", count);
    return NULL;
}
```
根据auth->type来决定验证username（login）还是验证password

## API分析 ##
admin/AuthorizedPasswords.c，四个API，因branch不同，API名称可能有不同。
### AuthorizedPasswords_add 或 add_pass ###
已经在上面分析过

### AuthorizedPasswords_remove 或 rm_pass ###
```
    Admin_registerFunction("AuthorizedPasswords_remove", remove, context, true,
        ((struct Admin_FunctionArg[]){
            { .name = "user", .required = 1, .type = "String" }
        }), admin);
```
```
static void remove(Dict* args, void* vcontext, String* txid, struct Allocator* requestAlloc)
{
    struct Context* context = Identity_check((struct Context*) vcontext);
    String* user = Dict_getStringC(args, "user");

    int32_t ret = CryptoAuth_removeUsers(context->ca, user);
    if (ret) {
        sendResponse(String_CONST("none"), context->admin, txid, requestAlloc);
    } else {
        sendResponse(String_CONST("Unknown error."), context->admin, txid, requestAlloc);
    }
}
```
CryptoAuth.c
```
int CryptoAuth_removeUsers(struct CryptoAuth* context, String* login)
{
    struct CryptoAuth_pvt* ca = Identity_check((struct CryptoAuth_pvt*) context);

    int count = 0;
    struct CryptoAuth_User** up = &ca->users;
    struct CryptoAuth_User* u = *up;
    while ((u = *up)) {
        if (!login || String_equals(login, u->login)) {
            *up = u->next;
            Allocator_free(u->alloc);
            count++;
        } else {
            up = &u->next;
        }
    }

    if (!login) {
        Log_debug(ca->logger, "Flushing [%d] users", count);
    } else {
        Log_debug(ca->logger, "Removing [%d] user(s) identified by [%s]", count, login->bytes);
    }
    return count;
}
```
注意两点：
1. while循环很好的在删除的同时维护好了list链表的结构
2. 当login为空时，也就意味着将所有user都删除了

### AuthorizedPasswords_remove 或 rm_pass ###
```
    Admin_registerFunction("AuthorizedPasswords_remove_by_pwd", removeByPwd, context, true,
        ((struct Admin_FunctionArg[]){
            { .name = "password", .required = 1, .type = "String" }
        }), admin);
```
```
static void removeByPwd(Dict* args, void* vcontext, String* txid, struct Allocator* requestAlloc)
{
    struct Context* context = Identity_check((struct Context*) vcontext);
    String* password = Dict_getStringC(args, "password");

    int32_t ret = CryptoAuth_removeUsers_pwd(context->ca, password);
    if (ret) {
        sendResponse(String_CONST("none"), context->admin, txid, requestAlloc);
    } else {
        sendResponse(String_CONST("Unknown error."), context->admin, txid, requestAlloc);
    }
}
```
CryptoAuth.c
```
static bool secretEquals(String* password, uint8_t userSecret[32])
{
    uint8_t pwdSecret[32];
    crypto_hash_sha256(pwdSecret, (uint8_t*) password->bytes, password->len);
    return !(Bits_memcmp(pwdSecret, userSecret, 32));
}

int CryptoAuth_removeUsers_pwd(struct CryptoAuth* context, String* password)
{
    struct CryptoAuth_pvt* ca = Identity_check((struct CryptoAuth_pvt*) context);

    int count = 0;
    struct CryptoAuth_User** up = &ca->users;
    struct CryptoAuth_User* u = *up;
    while ((u = *up)) {
        if (!password || secretEquals(password, u->secret)) {
            *up = u->next;
            Allocator_free(u->alloc);
            count++;
        } else {
            up = &u->next;
        }
    }

    if (!password) {
        Log_debug(ca->logger, "Flushing [%d] users", count);
    } else {
        Log_debug(ca->logger, "Removing [%d] user(s) identified by [%s]", count, password->bytes);
    }
    return count;
}
```
比上一个多了一个根据password计算secret的过程，其他并无差别。

## AuthorizedPasswords_list 或 ls_pass ##
```
    Admin_registerFunction("AuthorizedPasswords_list", list, context, true, NULL, admin);
```
```
static void list(Dict* args, void* vcontext, String* txid, struct Allocator* requestAlloc)
{
    struct Context* context = Identity_check((struct Context*) vcontext);
    struct Allocator* child = Allocator_child(context->allocator);

    List* users = CryptoAuth_getUsers(context->ca, child);
    uint32_t count = List_size(users);

    Dict response = Dict_CONST(
        String_CONST("total"), Int_OBJ(count), Dict_CONST(
        String_CONST("users"), List_OBJ(users), NULL
    ));

    Admin_sendMessage(&response, txid, context->admin);

    Allocator_free(child);
}
```
CryptoAuth.c
```
List* CryptoAuth_getUsers(struct CryptoAuth* context, struct Allocator* alloc)
{
    struct CryptoAuth_pvt* ca = Identity_check((struct CryptoAuth_pvt*) context);

    List* users = List_new(alloc);
    for (struct CryptoAuth_User* u = ca->users; u; u = u->next) {
        List_addString(users, String_clone(u->login, alloc), alloc);
    }

    return users;
}
```
返回结果仅列出users的name和users的数量，下面是一个执行结果示例：
```
{
  "total": "5",
  "txid": "3333180639",
  "users": [
    "a",
    "d",
    "c",
    "a",
    "Local Peers"
  ]
}
```
