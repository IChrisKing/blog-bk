---
title: 握手过程中，非对称密钥的应用
category:
  - cjdns
  - cjdns源码分析
tags:
  - cjdns
  - cjdns源码分析
  - C/C++
description: “在peer握手过程中，两对非对称密钥的应用。从握手的第一个包开始，分析普通点和接入点之间如何使用各自的稳定公私钥和临时公私钥，逐步完成整个握手过程。”
date: 2018-01-29 16:25:40
---

## 简介 ##
在cjdns中，使用到了两组非对称密钥对。
* 一组稳定的公私钥，主要用于握手阶段的身份验证等操作。
	这一组密钥也就是我们在conf中所配置的privateKey和publicKey。同时，publicKey也会出现在那些将我们设置为接入点的节点的conf文件中，作为connectTo中的publicKey字段。
* 一组临时的公私钥，用于握手完成后，正式通讯时加密数据。
	每一次会话都会生成临时的公私钥，这对公私钥才是真正用于加密数据的。

本文主要分析，在cjdns源码中，是如何使用稳定的公私钥进行身份验证的，又是如何在节点握手过程中完成临时公私钥生成，和临时公钥交换的。
文章将从普通节点向接入点连接开始分析，遵循节点之间建立连接的过程，逐步分析两对非对称密钥在连接建立过程当中的产生和作用。

## 普通点和接入点 ##
当我们要和接入点建立连接时，我们是知道接入点的publicKey的。这个值会和password一起写在conf文件中。
```
"connectTo":{
                    "xx.xxx.xx.xxx:30199": {
                        "password": "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
                        "publicKey":"xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx.k"
                    }
				}
```
### 发出第一个包：普通点向接入点发起连接 ###
本文不会分析配置从conf文件中读出的过程，直接进入到InterfaceConntroller.c中，有关和接入点建立链接的过程。
直接进入InterfaceController_bootstrapPeer方法，只分析相关代码
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
    struct Peer* ep = Allocator_calloc(epAlloc, sizeof(struct Peer), 1);
    ......
    ep->caSession = CryptoAuth_newSession(ic->ca, epAlloc, herPublicKey, false, "outer");
    ......
	sendPing(ep);

    return 0;
}
```
先看一下CryptoAuth_newSession方法，在CryptoAuth.c中

#### CryptoAuth_newSession ####
```
struct CryptoAuth_Session* CryptoAuth_newSession(struct CryptoAuth* ca,
                                                 struct Allocator* alloc,
                                                 const uint8_t herPublicKey[32],
                                                 const bool requireAuth,
                                                 char* displayName)
{
    ......
    struct CryptoAuth_Session_pvt* session =
        Allocator_calloc(alloc, sizeof(struct CryptoAuth_Session_pvt), 1);
    ......
    Assert_true(herPublicKey);
    Bits_memcpy(session->pub.herPublicKey, herPublicKey, 32);
    uint8_t calculatedIp6[16];
    AddressCalc_addressForPublicKey(calculatedIp6, herPublicKey);
    Bits_memcpy(session->pub.herIp6, calculatedIp6, 16);

    return &session->pub;
}
```
只分析和公私钥对有关的部分。
主要操作是将conf中，接入点的publicKey，通过CryptoAuth_newSession方法，设置到代表接入点的ep的caSession的pub.herPublicKey,并且计算出对应的ip6，设置到session->pub.herIp6中。

#### sendPing ####
sendPing的具体调用流程不在这里详细分析，只截取和公私钥有关的部分。
在sendPing的过程中，会经历两次加密。事实上，每次发包过程，都会经历两次加密。

1. 首先是针对目标节点的加密，使用自己和目标节点之间的公私钥对进行加密。调用点在SessionManager.c中的readyToSend中,`Assert_true(!CryptoAuth_encrypt(sess->pub.caSession, msg));`.
2. 然后是针对peer节点的加密，使用自己和peer节点之间的公私钥对进行加密。调用点在InterfaceController.c的sendFromSwitch中,`Assert_true(!CryptoAuth_encrypt(ep->caSession, msg));`。

这两次加密调用的都是同一个加密函数，但是传入的第一个参数不同。第一次调用时，会传入目标点的caSession。第二次调用时，会传入peer的caSession。
在连接接入点的过程中，两次加密都使用了自己和接入点之间的公私钥对，但两个caSession是不同的。

#### CryptoAuth_encrypt ####
直接看一下加密函数，CryptoAuth.c中的CryptoAuth_encrypt。两次加密使用的都是这个函数。
```
    struct CryptoAuth_Session_pvt* session =
        Identity_check((struct CryptoAuth_Session_pvt*) sessionPub);

    ......

    if (session->nextNonce <= CryptoAuth_State_RECEIVED_KEY) {
        if (session->nextNonce < CryptoAuth_State_RECEIVED_KEY) {
            encryptHandshake(msg, session, 0);
            return 0;
        }
        ......
    }

    ......
    return 0;
}
```
这是我们和接入点之间的第一个包，两个caSession的session->nextNonce = CryptoAuth_State_INIT = 0.所以，无论是第一次加密还是第二次加密，都会进入到encryptHandshake中

#### encryptHandshake ####
```
static void encryptHandshake(struct Message* message,
                             struct CryptoAuth_Session_pvt* session,
                             int setupMessage)
{
	......
    struct CryptoHeader* header = (struct CryptoHeader*) message->bytes;
    ......

    // set the permanent key
    Bits_memcpy(header->publicKey, session->context->pub.publicKey, 32);

    ......
    // Set the session state
    header->nonce = Endian_hostToBigEndian32(session->nextNonce);

    if (session->nextNonce == CryptoAuth_State_INIT ||
        session->nextNonce == CryptoAuth_State_RECEIVED_HELLO)
    {
        // If we're sending a hello or a key
        // Here we make up a temp keypair
        Random_bytes(session->context->rand, session->ourTempPrivKey, 32);
        crypto_scalarmult_curve25519_base(session->ourTempPubKey, session->ourTempPrivKey);
    }

    Bits_memcpy(header->encryptedTempKey, session->ourTempPubKey, 32);

    ......

    uint8_t sharedSecret[32];
    if (session->nextNonce < CryptoAuth_State_RECEIVED_HELLO) {
        getSharedSecret(sharedSecret,
                        session->context->privateKey,
                        session->pub.herPublicKey,
                        passwordHash,
                        session->context->logger);

        session->isInitiator = true;

        Assert_true(session->nextNonce <= CryptoAuth_State_SENT_HELLO);
        session->nextNonce = CryptoAuth_State_SENT_HELLO;
    }
    ......

    encryptRndNonce(header->handshakeNonce, message, sharedSecret);

    ......
}
```
主要做了一下几个操作：
##### 1. 把自己的稳定的publicKey放入握手message中 #####
```
	......
    struct CryptoHeader* header = (struct CryptoHeader*) message->bytes;
    ......

    // set the permanent key
    Bits_memcpy(header->publicKey, session->context->pub.publicKey, 32);

```

##### 2. 把自己的nextNonce放入握手message中的nonce字段 #####
此时的session->nextNonce为CryptoAuth_State_INIT，也就是0
```
    // Set the session state
    header->nonce = Endian_hostToBigEndian32(session->nextNonce);
```

##### 3. 生成握手过程中使用的临时公私钥对，并将临时公钥放入握手message中 #####
```
    if (session->nextNonce == CryptoAuth_State_INIT ||
        session->nextNonce == CryptoAuth_State_RECEIVED_HELLO)
    {
        // If we're sending a hello or a key
        // Here we make up a temp keypair
        Random_bytes(session->context->rand, session->ourTempPrivKey, 32);
        crypto_scalarmult_curve25519_base(session->ourTempPubKey, session->ourTempPrivKey);
    }

    Bits_memcpy(header->encryptedTempKey, session->ourTempPubKey, 32);
```

##### 4. 生成握手过程中使用的加密密钥sharedSecret，修改nextNonce为SENT_HELLO = 1 #####
```
uint8_t sharedSecret[32];
    if (session->nextNonce < CryptoAuth_State_RECEIVED_HELLO) {
        getSharedSecret(sharedSecret,
                        session->context->privateKey,
                        session->pub.herPublicKey,
                        passwordHash,
                        session->context->logger);

        session->isInitiator = true;

        Assert_true(session->nextNonce <= CryptoAuth_State_SENT_HELLO);
        session->nextNonce = CryptoAuth_State_SENT_HELLO;
    } else {
        ......
    }
```
注意看这里的getSharedSecret的参数，是自己的稳定私钥session->context->privateKey，对方的稳定公钥session->pub.herPublicKey,passwordHash。
```
static inline void getSharedSecret(uint8_t outputSecret[32],
                                   uint8_t myPrivateKey[32],
                                   uint8_t herPublicKey[32],
                                   uint8_t passwordHash[32],
                                   struct Log* logger)
{
    if (passwordHash == NULL) {
        crypto_box_curve25519xsalsa20poly1305_beforenm(outputSecret, herPublicKey, myPrivateKey);
    } else {
        union {
            struct {
                uint8_t key[32];
                uint8_t passwd[32];
            } components;
            uint8_t bytes[64];
        } buff;

        crypto_scalarmult_curve25519(buff.components.key, myPrivateKey, herPublicKey);
        Bits_memcpy(buff.components.passwd, passwordHash, 32);
        crypto_hash_sha256(outputSecret, buff.bytes, 64);
    }
    ......
}
```
根据passwordHash是否存在，也就是conf中是否有password字段，分为两种不同的方法来计算outputSecret。具体计算方法不需要分析。只需要知道，根据非对称加密的原理，在正确的成对使用双方的公私钥对，且password相同的情况下，一方加密的内容是一定会被对方解密的。在本次收发包中，使用的是双方的稳定公私钥对，只要接收方在解密握手包时，也使用双方的稳定公私钥对，message一定会被正确的解开。

##### 5. 使用sharedSecret加密握手message #####
```
    encryptRndNonce(header->handshakeNonce, message, sharedSecret);
```

#### 总结 ####
在普通点向接入点发出第一个包后，普通点上会建立一个接入点作为peer的caSession，和一个接入点作为目标点的caSession。

1. 对于第一个caSession，他是代表邻居节点的struct Peer* ep的成员。这个ep会加入到ici->peerMap中，map名为EndpointsBySockaddr。所有对于它的维护都在InterfaceController.c中进行，代表的是对于邻居节点的状态维护。当第一个包发送完成后，这个caSessio中包括
	* 邻居节点的稳定公钥herPublicKey
	* 自己的稳定公钥publicKey
	* 自己的稳定私钥privateKey
	* 自己的临时公钥ourTempPubKey
	* 自己的临时私钥ourTempPrivKey、
	* 密钥sharedSecret
	* nextNonce值为CryptoAuth_State_SENT_HELLO = 1。

	计算sharedSecret使用到
    * 自己的稳定私钥privateKey
    * 邻居节点的稳定公钥herTempPubKey

	在发出去的握手包中包括
	* 值为CryptoAuth_State_INIT = 0的nonce字段
	* 自己的稳定公钥publicKey
	* 自己的临时公钥ourTempPubKey
	* 使用sharedSecret加密过的message内容

2. 对于第二个caSession，他是代表与其他节点的会话的struct SessionManager_Session_pvt* sess的成员。这个sess会加入到sm->ifaceMap中，map名为OfSessionsByIp6。所有对于它的维护都在SessionManager.c中进行，代表的是对于与其他节点的会话的状态维护。当第一个包发送完成后，这个caSessio中包括
	* 目标节点的稳定公钥herPublicKey
	* 自己的稳定公钥publicKey
	* 自己的稳定私钥privateKey
	* 自己的临时公钥ourTempPubKey
	* 自己的临时私钥ourTempPrivKey
	* 密钥sharedSecret
	* nextNonce值为CryptoAuth_State_SENT_HELLO = 1

	计算sharedSecret使用到
    * 自己的稳定私钥privateKey
    * 目标节点的临时公钥herTempPubKey

	在发出去的握手包中包括
	* 值为CryptoAuth_State_INIT = 0的nonce字段
	* 自己的稳定公钥publicKey
	* 自己的临时公钥ourTempPubKey
	* 使用sharedSecret加密过的message内容

### 收到第一个包：接入点对握手包的解析处理 ###
当接入点收到握手包后，会对这个包进行解密。解密过程同样有两次解密操作。

1. 首先解密针对peer节点的加密，使用自己和peer节点之间的公私钥对进行解密。调用点在InterfaceController.c中,有两处：
	* handleIncomingFromWire函数中`if (CryptoAuth_decrypt(ep->caSession, msg)) `
	* handleUnexpectedIncoming函数中`if (CryptoAuth_decrypt(ep->caSession, msg))`。
	这两处调用都在收到数据包后的处理逻辑线上，根据不同情况，调用到其中的一个。

2. 然后解密针对目标节点的加密，使用自己和目标节点之间的公私钥对进行解密。调用点在SessionManager.c中的incomingFromSwitchIf中,`enum CryptoAuth_DecryptErr ret = CryptoAuth_decrypt(session->pub.caSession, msg);`.
这三个解密调用的都是同一个解密函数，但是传入的第一个参数不同。第一次调用时，会传入peer节点的caSession。第二次调用时，会传入目标节点的caSession。
在接入点处理握手包的过程中，两次解密都使用了向他连接的普通点和它之间的公私钥对，但两个caSession是不同的。

根据收到包后的处理逻辑，这个包首先会进入到InterfaceController.c的handleUnexpectedIncoming函数中

#### handleIncomingFromWire ####
```
static Iface_DEFUN handleIncomingFromWire(struct Message* msg, struct Iface* addrIf)
{
    ......
    int epIndex = Map_EndpointsBySockaddr_indexForKey(&lladdr, &ici->peerMap);
    if (epIndex == -1) {
        return handleUnexpectedIncoming(msg, ici);
    }

    struct Peer* ep = Identity_check((struct Peer*) ici->peerMap.values[epIndex]);
    ......
    if (CryptoAuth_decrypt(ep->caSession, msg)) {
        return NULL;
    }
    ......
    return receivedPostCryptoAuth(msg, ep, ici->ic);
}
```
首先试图在ici->peerMap中查找是否有这个点。
接下来针对查找结果有一个判断逻辑，
	* 如果没有记录，进入handleUnexpectedIncoming；
	* 如果有，取出对应的Peer,并调用CryptoAuth_decrypt。最后调用receivedPostCryptoAuth。

因为这是接入点收到普通点的第一个包，所以，不会有关于普通点的记录。我们直接进入进入handleUnexpectedIncoming

#### handleUnexpectedIncoming ####
```
static Iface_DEFUN handleUnexpectedIncoming(struct Message* msg,
                                            struct InterfaceController_Iface_pvt* ici)
{
    ......
    struct Peer* ep = Allocator_calloc(epAlloc, sizeof(struct Peer), 1);
    ......
    ep->caSession = CryptoAuth_newSession(ic->ca, epAlloc, ch->publicKey, true, "outer");
    if (CryptoAuth_decrypt(ep->caSession, msg)) {
        // If the first message is a dud, drop all state for this peer.
        // probably some random crap that wandered in the socket.
        Allocator_free(epAlloc);
        return NULL;
    }
    ......
    return receivedPostCryptoAuth(msg, ep, ic);
}
```

##### 1. 创建Peer，并调用CryptoAuth_newSession ######
重温一下CryptoAuth_newSession
```
struct CryptoAuth_Session* CryptoAuth_newSession(struct CryptoAuth* ca,
                                                 struct Allocator* alloc,
                                                 const uint8_t herPublicKey[32],
                                                 const bool requireAuth,
                                                 char* displayName)
{
    .......
    Assert_true(herPublicKey);
    Bits_memcpy(session->pub.herPublicKey, herPublicKey, 32);
    uint8_t calculatedIp6[16];
    AddressCalc_addressForPublicKey(calculatedIp6, herPublicKey);
    Bits_memcpy(session->pub.herIp6, calculatedIp6, 16);

    return &session->pub;
}
```
将收到的包中，普通点的publicKey，通过CryptoAuth_newSession方法，设置到代表普通点的ep的caSession的pub.herPublicKey,并且计算出对应的ip6，设置到session->pub.herIp6中。

##### 2. 调用CryptoAuth_decrypt，解密握手message #####
```
enum CryptoAuth_DecryptErr CryptoAuth_decrypt(struct CryptoAuth_Session* sessionPub,
                                              struct Message* msg)
{
    ......
    if (!session->established) {
        if (nonce >= Nonce_FIRST_TRAFFIC_PACKET) {
            ......
        }
        ......
        return decryptHandshake(session, nonce, msg, header);

    }
    ......
}
```

##### 3. 调用decryptHandshake #####
先简单说明下几个变量：
* nonce:从收到的包里取出，是发包者那边的nextNonce，值为CryptoAuth_State_INIT = 0
* nextNonce:一个局部变量，随情况变化，最后会被设置到session->nextNonce
* session->nextNonce:session中的nextNonce，真正标识这个session的状态,目前为CryptoAuth_State_INIT = 0

对照一下Nonce和CryptoAuth_State
```
enum Nonce {
    Nonce_HELLO = 0,
    Nonce_REPEAT_HELLO = 1,
    Nonce_KEY = 2,
    Nonce_REPEAT_KEY = 3,
    Nonce_FIRST_TRAFFIC_PACKET = 4
};
```
```
enum CryptoAuth_State {
    // New CryptoAuth session, has not sent or received anything
    CryptoAuth_State_INIT = 0,

    // Sent a hello message, waiting for reply
    CryptoAuth_State_SENT_HELLO = 1,

    // Received a hello message, have not yet sent a reply
    CryptoAuth_State_RECEIVED_HELLO = 2,

    // Received a hello message, sent a key message, waiting for the session to complete
    CryptoAuth_State_SENT_KEY = 3,

    // Sent a hello message, received a key message, may or may not have sent some data traffic
    // but no data traffic has yet been received
    CryptoAuth_State_RECEIVED_KEY = 4,

    // Received data traffic, session is in run state
    CryptoAuth_State_ESTABLISHED = 100
};
```

```
static enum CryptoAuth_DecryptErr decryptHandshake(struct CryptoAuth_Session_pvt* session,
                                                   const uint32_t nonce,
                                                   struct Message* message,
                                                   struct CryptoHeader* header)
{
    ......
    if (nonce < Nonce_KEY) { // HELLO or REPEAT_HELLO
        ......
        getSharedSecret(sharedSecret,
                        session->context->privateKey,
                        session->pub.herPublicKey,
                        passwordHash,
                        session->context->logger);
        nextNonce = CryptoAuth_State_RECEIVED_HELLO;
    }
    ......
    // Decrypt her temp public key and the message.
    if (decryptRndNonce(header->handshakeNonce, message, sharedSecret)) {
        // just in case
        Bits_memset(header, 0, CryptoHeader_SIZE);
        cryptoAuthDebug(session, "DROP message with nonce [%d], decryption failed", nonce);
        return CryptoAuth_DecryptErr_HANDSHAKE_DECRYPT_FAILED;
    }

    if (Bits_isZero(header->encryptedTempKey, 32)) {
        // we need to reject 0 public keys outright because they will be confused with "unknown"
        cryptoAuthDebug0(session, "DROP message with zero as temp public key");
        return CryptoAuth_DecryptErr_WISEGUY;
    }

    // Post-decryption checking
    if (nonce == Nonce_HELLO) {
        // A new hello packet
        if (!Bits_memcmp(session->herTempPubKey, header->encryptedTempKey, 32)) {
            // possible replay attack or duped packet
            cryptoAuthDebug0(session, "DROP dupe hello packet with same temp key");
            return CryptoAuth_DecryptErr_INVALID_PACKET;
        }
    }
    ......

    if (nextNonce == CryptoAuth_State_RECEIVED_KEY) {
       ......
    } else if (nextNonce == CryptoAuth_State_RECEIVED_HELLO) {
        Assert_true(nonce == Nonce_HELLO || nonce == Nonce_REPEAT_HELLO);
        if (Bits_memcmp(session->herTempPubKey, header->encryptedTempKey, 32)) {
            // fresh new hello packet, we should reset the session.
            switch (session->nextNonce) {
                case CryptoAuth_State_SENT_HELLO: {
                   	......
                }
                case CryptoAuth_State_INIT: {
                    Bits_memcpy(session->herTempPubKey, header->encryptedTempKey, 32);
                    break;
                }
                default: {
                    ......
                }
            }
        } else {
        	......
        }
    } else {
        ......
    }
    ......
    session->nextNonce = nextNonce;
    ......
}
```
这个函数主要做了一下操作：
###### 3.1. 计算SharedSecret ######
```
        getSharedSecret(sharedSecret,
                        session->context->privateKey,
                        session->pub.herPublicKey,
                        passwordHash,
                        session->context->logger);
        nextNonce = CryptoAuth_State_RECEIVED_HELLO;
```
关于这个函数的具体实现，在加密部分已经分析过，得出结论，因为双方使用的都是稳定的公私钥对，所以，计算出的SharedSecret是可以解密握手包的。

###### 3.2. 解密message和一些字段的合法性检查 ######
```
    // Decrypt her temp public key and the message.
    if (decryptRndNonce(header->handshakeNonce, message, sharedSecret)) {
        // just in case
        Bits_memset(header, 0, CryptoHeader_SIZE);
        cryptoAuthDebug(session, "DROP message with nonce [%d], decryption failed", nonce);
        return CryptoAuth_DecryptErr_HANDSHAKE_DECRYPT_FAILED;
    }

    if (Bits_isZero(header->encryptedTempKey, 32)) {
        // we need to reject 0 public keys outright because they will be confused with "unknown"
        cryptoAuthDebug0(session, "DROP message with zero as temp public key");
        return CryptoAuth_DecryptErr_WISEGUY;
    }

    // Post-decryption checking
    if (nonce == Nonce_HELLO) {
        // A new hello packet
        if (!Bits_memcmp(session->herTempPubKey, header->encryptedTempKey, 32)) {
            // possible replay attack or duped packet
            cryptoAuthDebug0(session, "DROP dupe hello packet with same temp key");
            return CryptoAuth_DecryptErr_INVALID_PACKET;
        }
    }
```
不做具体分析

###### 3.3. 将message中对方的临时公钥保存到herTempPubKey中 ######
```
case CryptoAuth_State_INIT: {
    Bits_memcpy(session->herTempPubKey, header->encryptedTempKey, 32);
    break;
}
```

###### 3.4 设置session->nextNonce ######
设为CryptoAuth_State_RECEIVED_HELLO = 2
```
session->nextNonce = nextNonce;
```

##### 4. 调用receivedPostCryptoAuth #####
```
static Iface_DEFUN receivedPostCryptoAuth(struct Message* msg,
                                          struct Peer* ep,
                                          struct InterfaceController_pvt* ic)
{
    ......
    if (ep->state < InterfaceController_PeerState_ESTABLISHED) {
        ......
        if (caState == CryptoAuth_State_ESTABLISHED) {
            ......
        } else {
            ......
            if (msg->length < 8 || msg->bytes[7] != 1) {
                ......
            } else {
                ......
                if ((ep->pingCount + 1) % 7) {
                    sendPing(ep);
                }
            }
        }
    }
    ......
    return Iface_next(&ep->switchIf, msg);
}
```
###### 4.1 sendPing ######
这是针对收到握手包之后发回的回应包，具体操作放在后面分析。

###### 4.2 Iface_next(&ep->switchIf, msg) ######
代码执行到这里，接入点作为peer的解密已经完成了。接下来将会去寻找这个包的下一跳，而这个包的目标节点也是接入点，所以这个包最终会交给SessionManager来处理。由incomingFromSwitchIf函数来解密，并维护caSession状态，很相似，不再具体分析了。

#### 总结 ####
当接入点收到普通点发出第一个包后，接入点上会建立一个普通点点作为peer的caSession，和一个普通点作为目标点的caSession。

1. 对于第一个caSession，他是代表邻居节点的struct Peer* ep的成员。这个ep会加入到ici->peerMap中，map名为EndpointsBySockaddr。所有对于它的维护都在InterfaceController.c中进行，代表的是对于邻居节点的状态维护。当这个包接收过程完成后，这个caSessio中包括
	* 邻居节点的稳定公钥herPublicKey
	* 邻居节点的临时公钥herTempPubKey
	* 自己的稳定公钥publicKey
	* 自己的稳定私钥privateKey
	* nextNonce值为CryptoAuth_State_RECEIVED_HELLO = 2

	计算sharedSecret使用到
    * 自己的稳定私钥privateKey
    * 邻居节点的临时公钥herTempPubKey

2. 对于第二个caSession，他是代表与其他节点的会话的struct SessionManager_Session_pvt* sess的成员。这个sess会加入到sm->ifaceMap中，map名为OfSessionsByIp6。所有对于它的维护都在SessionManager.c中进行，代表的是对于与其他节点的会话的状态维护。当第一个包的接收过程处理完成后，这个caSessio中包括
	* 目标节点的稳定公钥herPublicKey
	* 目标节点的临时公钥herTempPublicKey
	* 自己的稳定公钥publicKey
	* 自己的稳定私钥privateKey
	* nextNonce值为CryptoAuth_State_RECEIVED_HELLO = 2

	计算sharedSecret使用到
    * 自己的稳定私钥privateKey
    * 目标节点的临时公钥herTempPubKey

### 发出第二个包：接入点的回包 ###
接入点的回包操作，就是上面的sendPing。依然不详细分析具体调用流程，只关注对于加密部分的函数的调用，直接进入到CryptoAuth_encrypt方法。
#### CryptoAuth_encrypt ####
```
int CryptoAuth_encrypt(struct CryptoAuth_Session* sessionPub, struct Message* msg)
{
    ......
    if (session->nextNonce <= CryptoAuth_State_RECEIVED_KEY) {
        if (session->nextNonce < CryptoAuth_State_RECEIVED_KEY) {
            encryptHandshake(msg, session, 0);
            return 0;
        } else {
            ......
        }
    }
}
```
#### encryptHandshake ####
```
static void encryptHandshake(struct Message* message,
                             struct CryptoAuth_Session_pvt* session,
                             int setupMessage)
{
    ......
    // set the permanent key
    Bits_memcpy(header->publicKey, session->context->pub.publicKey, 32);

    // Set the session state
    header->nonce = Endian_hostToBigEndian32(session->nextNonce);

    if (session->nextNonce == CryptoAuth_State_INIT ||
        session->nextNonce == CryptoAuth_State_RECEIVED_HELLO)
    {
        // If we're sending a hello or a key
        // Here we make up a temp keypair
        Random_bytes(session->context->rand, session->ourTempPrivKey, 32);
        crypto_scalarmult_curve25519_base(session->ourTempPubKey, session->ourTempPrivKey);
		......
    }

    Bits_memcpy(header->encryptedTempKey, session->ourTempPubKey, 32);
    ......
    uint8_t sharedSecret[32];
    if (session->nextNonce < CryptoAuth_State_RECEIVED_HELLO) {
        ......
    } else {
        // Handshake2
        // herTempPubKey was set by decryptHandshake()
        Assert_ifParanoid(!Bits_isZero(session->herTempPubKey, 32));
        getSharedSecret(sharedSecret,
                        session->context->privateKey,
                        session->herTempPubKey,
                        passwordHash,
                        session->context->logger);

        Assert_true(session->nextNonce <= CryptoAuth_State_SENT_KEY);
        session->nextNonce = CryptoAuth_State_SENT_KEY;
        ......
    }

    ......
    encryptRndNonce(header->handshakeNonce, message, sharedSecret);
    ......
}
```
这个函数的主要操作：
##### 1. 将自己的稳定公钥放到message中 #####
```
	// set the permanent key
    Bits_memcpy(header->publicKey, session->context->pub.publicKey, 32);
```

##### 2. 将自己的nextNonce放到message中 #####
```
	// Set the session state
    header->nonce = Endian_hostToBigEndian32(session->nextNonce);
```
此时，nextNonce值为CryptoAuth_State_RECEIVED_HELLO = 2

##### 3. 生成握手过程中使用的临时公私钥对，并将临时公钥放入握手message中 #####
```
    if (session->nextNonce == CryptoAuth_State_INIT ||
        session->nextNonce == CryptoAuth_State_RECEIVED_HELLO)
    {
        // If we're sending a hello or a key
        // Here we make up a temp keypair
        Random_bytes(session->context->rand, session->ourTempPrivKey, 32);
        crypto_scalarmult_curve25519_base(session->ourTempPubKey, session->ourTempPrivKey);
    }

    Bits_memcpy(header->encryptedTempKey, session->ourTempPubKey, 32);
```

##### 4. 生成握手过程中使用的加密密钥sharedSecret，修改nextNonce为SENT_KEY = 3 #####
```
if (session->nextNonce < CryptoAuth_State_RECEIVED_HELLO) {
        ......
    } else {
        // Handshake2
        // herTempPubKey was set by decryptHandshake()
        Assert_ifParanoid(!Bits_isZero(session->herTempPubKey, 32));
        getSharedSecret(sharedSecret,
                        session->context->privateKey,
                        session->herTempPubKey,
                        passwordHash,
                        session->context->logger);

        Assert_true(session->nextNonce <= CryptoAuth_State_SENT_KEY);
        session->nextNonce = CryptoAuth_State_SENT_KEY;
        ......
    }
```
要注意，这里getSharedSecret使用的是自己的稳定私钥和对方的临时公钥。我们要关注解密时使用的公私钥对是否匹配。

##### 5. 使用sharedSecret加密握手message #####
```
    encryptRndNonce(header->handshakeNonce, message, sharedSecret);
```

#### 总结 ####
这是握手过程中的第二个包，由接入点发出。这个包发出后，接入点上关于普通点的两个caSession都发生了变化。

1. 对于代表邻居节点的caSession，目前的状态如下，其中黑体为本次发包过程中的改变
	* 邻居节点的稳定公钥herPublicKey
	* 邻居节点的临时公钥herTempPubKey
	* 自己的稳定公钥publicKey
	* 自己的稳定私钥privateKey
	* **自己的临时公钥ourTempPubKey**
	* **自己的临时私钥ourTempPrivKey**
	* **密钥sharedSecret**
	* **nextNonce值为CryptoAuth_State_SENT_KEY = 3**

	计算sharedSecret使用到
    * 自己的稳定私钥privateKey
    * 邻居节点的临时公钥herTempPubKey

	在发出去的握手包中包括
	* 值为CryptoAuth_State_RECEIVED_HELLO = 2的nonce字段
	* 自己的稳定公钥publicKey
	* 自己的临时公钥ourTempPubKey
	* 使用sharedSecret加密过的message内容

2. 对于代表目标节点的caSession，目前的状态如下，其中黑体为本次发包过程中的改变
	* 目标节点的稳定公钥herPublicKey
	* 目标节点的临时公钥herTempPubKey
	* 自己的稳定公钥publicKey
	* 自己的稳定私钥privateKey
	* **自己的临时公钥ourTempPubKey**
	* **自己的临时私钥ourTempPrivKey**
	* **密钥sharedSecret**
	* **nextNonce值为CryptoAuth_State_SENT_KEY = 3**

	计算sharedSecret使用到
    * 自己的稳定私钥privateKey
    * 目标节点的临时公钥herTempPubKey

	在发出去的握手包中包括
	* 值为CryptoAuth_State_RECEIVED_HELLO = 2的nonce字段
	* 自己的稳定公钥publicKey
	* 自己的临时公钥ourTempPubKey
	* 使用sharedSecret加密过的message内容

### 收到第二个包：普通点收到接入点的回包 ###
当普通点收到接入点的回包后，会对这个包进行解密。解密过程同样有两次解密操作。

1. 首先解密针对peer节点的加密，使用自己和peer节点之间的公私钥对进行解密。调用点在InterfaceController.c中,有两处：
	* handleIncomingFromWire函数中`if (CryptoAuth_decrypt(ep->caSession, msg)) `
	* handleUnexpectedIncoming函数中`if (CryptoAuth_decrypt(ep->caSession, msg))`。
	这两处调用都在收到数据包后的处理逻辑线上，根据不同情况，调用到其中的一个。

2. 然后解密针对目标节点的加密，使用自己和目标节点之间的公私钥对进行解密。调用点在SessionManager.c中的incomingFromSwitchIf中,`enum CryptoAuth_DecryptErr ret = CryptoAuth_decrypt(session->pub.caSession, msg);`.
这三个解密调用的都是同一个解密函数，但是传入的第一个参数不同。第一次调用时，会传入peer节点的caSession。第二次调用时，会传入目标节点的caSession。
在接入点处理握手包的过程中，两次解密都使用了向他连接的普通点和它之间的公私钥对，但两个caSession是不同的。

根据收到包后的处理逻辑，这个包首先会进入到InterfaceController.c的handleUnexpectedIncoming函数中

#### handleIncomingFromWire ####
```
static Iface_DEFUN handleIncomingFromWire(struct Message* msg, struct Iface* addrIf)
{
    ......
    int epIndex = Map_EndpointsBySockaddr_indexForKey(&lladdr, &ici->peerMap);
    ......

    struct Peer* ep = Identity_check((struct Peer*) ici->peerMap.values[epIndex]);
    ......
    if (CryptoAuth_decrypt(ep->caSession, msg)) {
        return NULL;
    }
    ......
    return receivedPostCryptoAuth(msg, ep, ici->ic);
}
```

##### 1. 从ici->peerMap中找到ep #####
```
	int epIndex = Map_EndpointsBySockaddr_indexForKey(&lladdr, &ici->peerMap);
    ......

    struct Peer* ep = Identity_check((struct Peer*) ici->peerMap.values[epIndex]);
```

##### 2. 调用CryptoAuth_decrypt进行解密 #####
```
enum CryptoAuth_DecryptErr CryptoAuth_decrypt(struct CryptoAuth_Session* sessionPub,
                                              struct Message* msg)
{
    ......
    uint32_t nonce = Endian_bigEndianToHost32(header->nonce);

    if (!session->established) {
        if (nonce >= Nonce_FIRST_TRAFFIC_PACKET) {
            ......
        }
		......
        return decryptHandshake(session, nonce, msg, header);

    }
    ......
}
```

##### 3. decryptHandshake #####
先给出几个关键参数的值：
* nonce:从收到的包里取出，是发包者那边的nextNonce，值为CryptoAuth_State_RECEIVED_HELLO = 2
* nextNonce:一个局部变量，随情况变化，最后会被设置到session->nextNonce
* session->nextNonce:session中的nextNonce，真正标识这个session的状态。当前值为CryptoAuth_State_SENT_HELLO = 1

对照一下Nonce和CryptoAuth_State
```
enum Nonce {
    Nonce_HELLO = 0,
    Nonce_REPEAT_HELLO = 1,
    Nonce_KEY = 2,
    Nonce_REPEAT_KEY = 3,
    Nonce_FIRST_TRAFFIC_PACKET = 4
};
```
```
enum CryptoAuth_State {
    // New CryptoAuth session, has not sent or received anything
    CryptoAuth_State_INIT = 0,

    // Sent a hello message, waiting for reply
    CryptoAuth_State_SENT_HELLO = 1,

    // Received a hello message, have not yet sent a reply
    CryptoAuth_State_RECEIVED_HELLO = 2,

    // Received a hello message, sent a key message, waiting for the session to complete
    CryptoAuth_State_SENT_KEY = 3,

    // Sent a hello message, received a key message, may or may not have sent some data traffic
    // but no data traffic has yet been received
    CryptoAuth_State_RECEIVED_KEY = 4,

    // Received data traffic, session is in run state
    CryptoAuth_State_ESTABLISHED = 100
};
```

```
static enum CryptoAuth_DecryptErr decryptHandshake(struct CryptoAuth_Session_pvt* session,
                                                   const uint32_t nonce,
                                                   struct Message* message,
                                                   struct CryptoHeader* header)
{
    ......
    if (nonce < Nonce_KEY) { // HELLO or REPEAT_HELLO
        ......
    } else {
        ......
        // We sent the hello, this is a key
        getSharedSecret(sharedSecret,
                        session->ourTempPrivKey,
                        session->pub.herPublicKey,
                        passwordHash,
                        session->context->logger);
        nextNonce = CryptoAuth_State_RECEIVED_KEY;
    }

    ......
    // Decrypt her temp public key and the message.
    if (decryptRndNonce(header->handshakeNonce, message, sharedSecret)) {
        // just in case
        Bits_memset(header, 0, CryptoHeader_SIZE);
        cryptoAuthDebug(session, "DROP message with nonce [%d], decryption failed", nonce);
        return CryptoAuth_DecryptErr_HANDSHAKE_DECRYPT_FAILED;
    }

    ......
    if (nextNonce == CryptoAuth_State_RECEIVED_KEY) {
        Assert_true(nonce == Nonce_KEY || nonce == Nonce_REPEAT_KEY);
        switch (session->nextNonce) {
            ......
            case CryptoAuth_State_SENT_HELLO: {
                Bits_memcpy(session->herTempPubKey, header->encryptedTempKey, 32);
                break;
            }
            ......
        }
    }
    ......
    session->nextNonce = nextNonce;
    ......
}
```

###### 3.1 计算SharedSecret ######
```
        getSharedSecret(sharedSecret,
                        session->ourTempPrivKey,
                        session->pub.herPublicKey,
                        passwordHash,
                        session->context->logger);
        nextNonce = CryptoAuth_State_RECEIVED_KEY;
```
关于这个函数的具体实现，在加密部分已经分析过。在接入点发包时，使用的是接入点的稳定私钥和普通点的临时公钥。而在这里，使用的是自己的临时私钥和接入点稳定公钥。公私钥对匹配，所以，计算出的SharedSecret是可以解密握手包的。

###### 3.2. 解密message ######
```
    // Decrypt her temp public key and the message.
    if (decryptRndNonce(header->handshakeNonce, message, sharedSecret)) {
        // just in case
        Bits_memset(header, 0, CryptoHeader_SIZE);
        cryptoAuthDebug(session, "DROP message with nonce [%d], decryption failed", nonce);
        return CryptoAuth_DecryptErr_HANDSHAKE_DECRYPT_FAILED;
    }
```
不做具体分析

###### 3.3. 将message中对方的临时公钥保存到herTempPubKey中 ######
```
	if (nextNonce == CryptoAuth_State_RECEIVED_KEY) {
        Assert_true(nonce == Nonce_KEY || nonce == Nonce_REPEAT_KEY);
        switch (session->nextNonce) {
            ......
            case CryptoAuth_State_SENT_HELLO: {
                Bits_memcpy(session->herTempPubKey, header->encryptedTempKey, 32);
                break;
            }
            ......
        }
    }
```

###### 3.4 设置session->nextNonce ######
设为CryptoAuth_State_RECEIVED_KEY = 4
```
	session->nextNonce = nextNonce;
```

##### 4. 调用receivedPostCryptoAuth #####
```
static Iface_DEFUN receivedPostCryptoAuth(struct Message* msg,
                                          struct Peer* ep,
                                          struct InterfaceController_pvt* ic)
{
    ......
    if (ep->state < InterfaceController_PeerState_ESTABLISHED) {
   		......
        if (caState == CryptoAuth_State_ESTABLISHED) {
            ......
        } else {
            ......
            if (msg->length < 8 || msg->bytes[7] != 1) {
                ......
            } else {
                ......
                if ((ep->pingCount + 1) % 7) {
                    sendPing(ep);
                }
            }
        }
    }
    ......
    return Iface_next(&ep->switchIf, msg);
}
```
Iface_next(&ep->switchIf, msg);会将message送去SessionManager.c，进行针对目标节点的解密操作，不做详细分析。
后面将直接进入回包分析，sendPing。

#### 总结 ####
网络中的第二个包，是普通点收到接入点发出的回包，普通点会根据回包的内容，维护两个caSession的状态。

1. 对于代表邻居节点的caSession，目前的状态如下，其中黑体为本次收包过程中的改变
	* 邻居节点的稳定公钥herPublicKey
	* **邻居节点的临时公钥herTempPubKey**
	* 自己的稳定公钥publicKey
	* 自己的稳定私钥privateKey
	* 自己的临时公钥ourTempPubKey
	* 自己的临时私钥ourTempPrivKey
	* **nextNonce值为CryptoAuth_State_RECEIVED_KEY = 4。**

	计算sharedSecret使用到
    * 自己的临时私钥ourTempPrivKey
    * 邻居节点的稳定公钥herPublicKey

2. 对于代表目标节点的caSession，目前的状态如下，其中黑体为本次收包过程中的改变
	* 目标节点的稳定公钥herPublicKey
	* **目标节点的临时公钥herTempPubKey**
	* 自己的稳定公钥publicKey
	* 自己的稳定私钥privateKey
	* 自己的临时公钥ourTempPubKey
	* 自己的临时私钥ourTempPrivKey
	* **nextNonce值为CryptoAuth_State_RECEIVED_KEY = 4 **

	计算sharedSecret使用到
    * 自己的临时私钥ourTempPrivKey
    * 目标节点的稳定公钥herPublicKey

### 发出第三个包：普通点向接入点回包 ###
普通点的回包操作，就是上面的sendPing。依然不详细分析具体调用流程，只关注对于加密部分的函数的调用，直接进入到CryptoAuth_encrypt方法。
#### CryptoAuth_encrypt ####
目前session->nextNonce为CryptoAuth_State_RECEIVED_KEY = 4
```
int CryptoAuth_encrypt(struct CryptoAuth_Session* sessionPub, struct Message* msg)
{
    ......
    if (session->nextNonce <= CryptoAuth_State_RECEIVED_KEY) {
        if (session->nextNonce < CryptoAuth_State_RECEIVED_KEY) {
            ......
        } else {
            ......
            getSharedSecret(session->sharedSecret,
                            session->ourTempPrivKey,
                            session->herTempPubKey,
                            NULL,
                            session->context->logger);
        }
    }

    encrypt(session->nextNonce, msg, session->sharedSecret, session->isInitiator);

    Message_push32(msg, session->nextNonce, NULL);
    session->nextNonce++;
    return 0;
}
```
#### getSharedSecret ####
```
            getSharedSecret(session->sharedSecret,
                            session->ourTempPrivKey,
                            session->herTempPubKey,
                            NULL,
                            session->context->logger);
```
这里使用的是自己的临时私钥和接入点的临时公钥来计算SharedSecret

#### encrypt加密message ####
```
static inline void encrypt(uint32_t nonce,
                           struct Message* msg,
                           uint8_t secret[32],
                           bool isInitiator)
{
    union {
        uint32_t ints[2];
        uint8_t bytes[24];
    } nonceAs = { .ints = {0, 0} };
    nonceAs.ints[isInitiator] = Endian_hostToLittleEndian32(nonce);

    encryptRndNonce(nonceAs.bytes, msg, secret);
}
```
```
static inline void encryptRndNonce(uint8_t nonce[24],
                                   struct Message* msg,
                                   uint8_t secret[32])
{
    Assert_true(msg->padding >= 32);
    uint8_t* startAt = msg->bytes - 32;
    // This function trashes 16 bytes of the padding so we will put it back
    uint8_t paddingSpace[16];
    Bits_memcpy(paddingSpace, startAt, 16);
    Bits_memset(startAt, 0, 32);
    if (!Defined(NSA_APPROVED)) {
        crypto_box_curve25519xsalsa20poly1305_afternm(
            startAt, startAt, msg->length + 32, nonce, secret);
    }

    Bits_memcpy(startAt, paddingSpace, 16);
    Message_shift(msg, 16, NULL);
}
```

#### 将session的next->nextNonce放入message中 ####
此时session->nextNonce为CryptoAuth_State_RECEIVED_KEY = 4
```
Message_push32(msg, session->nextNonce, NULL);
```

#### 维护session->nextNonce ####
```
    session->nextNonce++;
```
此时session->nextNonce值为5

#### 总结 ####
网络中的第三个包，是普通点向接入点发出的包，此时两个caSession的状态。

1. 对于代表邻居节点的caSession，目前的状态如下，其中黑体为本次发包后的改变
	* 邻居节点的稳定公钥herPublicKey
	* 邻居节点的临时公钥herTempPubKey
	* 自己的稳定公钥publicKey
	* 自己的稳定私钥privateKey
	* 自己的临时公钥ourTempPubKey
	* 自己的临时私钥ourTempPrivKey
	* **nextNonce值为5**

	计算sharedSecret使用到
    * 自己的临时私钥ourTempPrivKey
    * 邻居节点的临时公钥herTempPubKey

	在发出去的握手包中包括
	* 值为CryptoAuth_State_RECEIVED_KEY = 4的nonce字段
	* 使用sharedSecret加密过的message内容

2. 对于代表目标节点的caSession，目前的状态如下，其中黑体为本次发包后的改变
	* 目标节点的稳定公钥herPublicKey
	* 目标节点的临时公钥herTempPubKey
	* 自己的稳定公钥publicKey
	* 自己的稳定私钥privateKey
	* 自己的临时公钥ourTempPubKey
	* 自己的临时私钥ourTempPrivKey
	* **nextNonce值为5**

	计算sharedSecret使用到
    * 自己的临时私钥ourTempPrivKey
    * 目标节点的临时公钥herTempPubKey

	在发出去的握手包中包括
	* 值为CryptoAuth_State_RECEIVED_KEY = 4的nonce字段
	* 使用sharedSecret加密过的message内容

### 收到第三个包：接入点收到普通点发来的包 ###
直接进入handleIncomingFromWire
#### handleIncomingFromWire ####
```
static Iface_DEFUN handleIncomingFromWire(struct Message* msg, struct Iface* addrIf)
{
    ......
    int epIndex = Map_EndpointsBySockaddr_indexForKey(&lladdr, &ici->peerMap);
    ......
    struct Peer* ep = Identity_check((struct Peer*) ici->peerMap.values[epIndex]);
    ......
    if (CryptoAuth_decrypt(ep->caSession, msg)) {
        return NULL;
    }
    ......
    return receivedPostCryptoAuth(msg, ep, ici->ic);
}
```
##### 1. CryptoAuth_decrypt #####
```
enum CryptoAuth_DecryptErr CryptoAuth_decrypt(struct CryptoAuth_Session* sessionPub,
                                              struct Message* msg)
{
    ......
    uint32_t nonce = Endian_bigEndianToHost32(header->nonce);//nonce值为4

    if (!session->established) {
        if (nonce >= Nonce_FIRST_TRAFFIC_PACKET) {//Nonce_FIRST_TRAFFIC_PACKET = 4
            ......
            getSharedSecret(secret,
                            session->ourTempPrivKey,
                            session->herTempPubKey,
                            NULL,
                            session->context->logger);

            enum CryptoAuth_DecryptErr ret = decryptMessage(session, nonce, msg, secret);
            if (!ret) {
                ......
                Bits_memcpy(session->sharedSecret, secret, 32);

                // Now we're in run mode, no more handshake packets will be accepted
                session->established = true;
                session->nextNonce += 3;
                ......
                return 0;
            }
            ......
        }
		......
    }
    ......
}
```
###### 1.1 getSharedSecret ######
```
            getSharedSecret(secret,
                            session->ourTempPrivKey,
                            session->herTempPubKey,
                            NULL,
                            session->context->logger);

```
这里，使用了接入点的临时私钥和普通点的临时公钥来计算SharedSecret

####### 1.2 调用decryptMessage解密message ######
```
static inline enum CryptoAuth_DecryptErr decryptMessage(struct CryptoAuth_Session_pvt* session,
                                                        uint32_t nonce,
                                                        struct Message* content,
                                                        uint8_t secret[32])
{
    // Decrypt with authentication and replay prevention.
    if (decrypt(nonce, content, secret, session->isInitiator)) {
        cryptoAuthDebug0(session, "DROP authenticated decryption failed");
        return CryptoAuth_DecryptErr_DECRYPT;
    }
    ......
    return 0;
}
```

###### 1.3 保存会话密钥 ######
```
Bits_memcpy(session->sharedSecret, secret, 32);
```
将使用接入点临时私钥和普通点临时公钥计算出的SharedSecret保存到session->sharedSecret中。
当握手过程完成后，节点间的交互都会使用这个SharedSecret来加密message。将这个值保存起来，方便后期的加密操作。

###### 1.4 维护session状态 ######
这段代码执行之前，session->nextNonce的值为CryptoAuth_State_SENT_KEY = 3
```
session->established = true;
session->nextNonce += 3;
```
此时session->nextNonce的值为6。session状态为established

##### 2. 调用receivedPostCryptoAuth #####
```
static Iface_DEFUN receivedPostCryptoAuth(struct Message* msg,
                                          struct Peer* ep,
                                          struct InterfaceController_pvt* ic)
{
    ......
    int caState = CryptoAuth_getState(ep->caSession);

    if (ep->state < InterfaceController_PeerState_ESTABLISHED) {
        // EP states track CryptoAuth states...
        ep->state = caState;
        ......
        if (caState == CryptoAuth_State_ESTABLISHED) {
            moveEndpointIfNeeded(ep);
            //sendPeer(0xffffffff, PFChan_Core_PEER, ep);// version is not known at this point.
        }
        ......
    }
	......
	return Iface_next(&ep->switchIf, msg);
}
```

###### 2.1 计算caState ######
```
enum CryptoAuth_State CryptoAuth_getState(struct CryptoAuth_Session* caSession)
{
    struct CryptoAuth_Session_pvt* session =
        Identity_check((struct CryptoAuth_Session_pvt*)caSession);

    if (session->nextNonce <= CryptoAuth_State_RECEIVED_KEY) {
        return session->nextNonce;
    }
    return (session->established) ? CryptoAuth_State_ESTABLISHED : CryptoAuth_State_RECEIVED_KEY;
}
```
之前都没有分析这个函数，原因是，在这个包之前，caState都和session->nextNonce一样。
而这一次，由于前面将session->established设为true，所以，这次，caState将会返回CryptoAuth_State_ESTABLISHED

###### 2.2 一些状态维护操作 ######
```
    if (ep->state < InterfaceController_PeerState_ESTABLISHED) {
        // EP states track CryptoAuth states...
        ep->state = caState;
        ......
        if (caState == CryptoAuth_State_ESTABLISHED) {
            moveEndpointIfNeeded(ep);
            //sendPeer(0xffffffff, PFChan_Core_PEER, ep);// version is not known at this point.
        }
        ......
    }
```
主要是修改了ep->state

###### 2.3 将message送去下一站处理 ######
```
	return Iface_next(&ep->switchIf, msg);
```
Iface_next(&ep->switchIf, msg);会将message送去SessionManager.c，进行针对目标节点的解密操作，不做详细分析。

#### 总结 ####
网络中的第三个包，是接入点收到普通点发出的回包，接入点会根据回包的内容，维护两个caSession的状态。

1. 对于代表邻居节点的caSession，目前的状态如下，其中黑体为本次收包过程中的改变
	* 邻居节点的稳定公钥herPublicKey
	* 邻居节点的临时公钥herTempPubKey
	* 自己的稳定公钥publicKey
	* 自己的稳定私钥privateKey
	* 自己的临时公钥ourTempPubKey
	* 自己的临时私钥ourTempPrivKey
	* **nextNonce值为6**
	* **caState为established**
	* **ep->state为established**

	计算sharedSecret使用到
    * 自己的临时私钥ourTempPrivKey
    * 邻居节点的临时公钥herTempPubKey

2. 对于代表目标节点的caSession，目前的状态如下，其中黑体为本次收包过程中的改变
	* 目标节点的稳定公钥herPublicKey
	* 目标节点的临时公钥herTempPubKey
	* 自己的稳定公钥publicKey
	* 自己的稳定私钥privateKey
	* 自己的临时公钥ourTempPubKey
	* 自己的临时私钥ourTempPrivKey
	* **nextNonce值为6**
	* **caState为established**
	* **ep->state为established**

	计算sharedSecret使用到
    * 自己的临时私钥ourTempPrivKey
    * 目标节点的临时公钥herTempPubKey

到这里，接入点上关于普通点的两个caSession的状态都已经变成了established。握手过程结束，进入正常的会话过程。
然而，普通点怎么办？普通点上关于接入点的两个caSession的状态都还没有established，nextNonce值都是5.如何触发普通点上的caSession也变成established？这就需要用到cjdns中的一个关于节点保活的机制。
当接入点上表示普通点的那个ep的state变为established之后，接入点将会每隔固定时间，向普通点发送sendPing来维护普通点的状态。于是，在这个维护机制下，接入点向普通点发送了sendPing。

### 第四个包：接入点发送ping包，触发普通点状态变化 ###
关于sendPing。依然不详细分析具体调用流程，只关注对于加密部分的函数的调用，直接进入到CryptoAuth_encrypt方法。
#### CryptoAuth_encrypt ####
目前session->nextNonce为6
```
int CryptoAuth_encrypt(struct CryptoAuth_Session* sessionPub, struct Message* msg)
{
    ......
    encrypt(session->nextNonce, msg, session->sharedSecret, session->isInitiator);

    Message_push32(msg, session->nextNonce, NULL);
    session->nextNonce++;
    return 0;
}
```
这里主要做了三个操作：
1. 调用encrypt加密message,使用的密钥是session->sharedSecret，这是使用接入点的临时私钥和普通点的临时公钥计算出来的
2. 将nextNonce=6放入message中
3. 修改nextNonce为7

### 收到第四个包：普通点收到接入点发来的ping包 ###
普通点将在处理这个包的过程中完成握手阶段，进入稳定会话阶段。
#### handleIncomingFromWire ####
```static Iface_DEFUN handleIncomingFromWire(struct Message* msg, struct Iface* addrIf)
{
    ......
    int epIndex = Map_EndpointsBySockaddr_indexForKey(&lladdr, &ici->peerMap);
    ......
    struct Peer* ep = Identity_check((struct Peer*) ici->peerMap.values[epIndex]);
    ......
    if (CryptoAuth_decrypt(ep->caSession, msg)) {
        return NULL;
    }
    ......
    if (ep->state == InterfaceController_PeerState_ESTABLISHED &&
        CryptoAuth_getState(ep->caSession) != CryptoAuth_State_ESTABLISHED) {
        sendPeer(0xffffffff, PFChan_Core_PEER_GONE, ep);
    }
    return receivedPostCryptoAuth(msg, ep, ici->ic);
}
```
##### 1. 从ici->peerMap中找到ep #####
```
	int epIndex = Map_EndpointsBySockaddr_indexForKey(&lladdr, &ici->peerMap);
    ......

    struct Peer* ep = Identity_check((struct Peer*) ici->peerMap.values[epIndex]);
```

##### 2. 调用CryptoAuth_decrypt进行解密 #####
nonce:6
session->nextNonce = 5
```
enum CryptoAuth_DecryptErr CryptoAuth_decrypt(struct CryptoAuth_Session* sessionPub,
                                              struct Message* msg)
{
    ......
    uint32_t nonce = Endian_bigEndianToHost32(header->nonce);

    if (!session->established) {
        if (nonce >= Nonce_FIRST_TRAFFIC_PACKET) {
            ......
            getSharedSecret(secret,
                            session->ourTempPrivKey,
                            session->herTempPubKey,
                            NULL,
                            session->context->logger);

            enum CryptoAuth_DecryptErr ret = decryptMessage(session, nonce, msg, secret);
            if (!ret) {
                cryptoAuthDebug0(session, "Final handshake step succeeded");
                Bits_memcpy(session->sharedSecret, secret, 32);

                // Now we're in run mode, no more handshake packets will be accepted
                session->established = true;
                session->nextNonce += 3;
                ......
                return 0;
            }
            ......
        }

        ......
    }
    ......
}
```
###### 2.1 计算SharedSecret ######
```
            getSharedSecret(secret,
                            session->ourTempPrivKey,
                            session->herTempPubKey,
                            NULL,
                            session->context->logger);
```
使用了普通点的临时私钥和接入点的临时公钥来计算SharedSecret

###### 2.2 解密message ######
```
	enum CryptoAuth_DecryptErr ret = decryptMessage(session, nonce, msg, secret);
```

###### 2.3 保存会话密钥 ######
```
Bits_memcpy(session->sharedSecret, secret, 32);
```
将使用普通点临时私钥和接入点临时公钥计算出的SharedSecret保存到session->sharedSecret中。
当握手过程完成后，节点间的交互都会使用这个SharedSecret来加密message。将这个值保存起来，方便后期的加密操作。

###### 2.4 维护session状态 ######
这段代码执行之前，session->nextNonce的值为5
```
session->established = true;
session->nextNonce += 3;
```
此时session->nextNonce的值为8。session状态为established

##### 3. receivedPostCryptoAuth #####
```
static Iface_DEFUN receivedPostCryptoAuth(struct Message* msg,
                                          struct Peer* ep,
                                          struct InterfaceController_pvt* ic)
{
    ......
    int caState = CryptoAuth_getState(ep->caSession);
    if (ep->state < InterfaceController_PeerState_ESTABLISHED) {
        // EP states track CryptoAuth states...
        ep->state = caState;
        ......
        if (caState == CryptoAuth_State_ESTABLISHED) {
            moveEndpointIfNeeded(ep);
            //sendPeer(0xffffffff, PFChan_Core_PEER, ep);// version is not known at this point.
        } else {
            ......
        }
    }
    ......
    return Iface_next(&ep->switchIf, msg);
}
```
和前面接入点收到包后的处理一样，不再详细分析。
caSession和ep->state都是CryptoAuth_State_ESTABLISHED

#### 总结 ####
网络中的第四个包，是接入点执行保活操作时，发往普通点的包。普通点根据包的内容，维护两个caSession的状态。

1. 对于代表邻居节点的caSession，目前的状态如下，其中黑体为本次收包过程中的改变
	* 邻居节点的稳定公钥herPublicKey
	* 邻居节点的临时公钥herTempPubKey
	* 自己的稳定公钥publicKey
	* 自己的稳定私钥privateKey
	* 自己的临时公钥ourTempPubKey
	* 自己的临时私钥ourTempPrivKey
	* **nextNonce值为8**
	* **caState为established**
	* **ep->state为established**

2. 对于代表目标节点的caSession，目前的状态如下，其中黑体为本次收包过程中的改变
	* 目标节点的稳定公钥herPublicKey
	* 目标节点的临时公钥herTempPubKey
	* 自己的稳定公钥publicKey
	* 自己的稳定私钥privateKey
	* 自己的临时公钥ourTempPubKey
	* 自己的临时私钥ourTempPrivKey
	* **nextNonce值为8**
	* **caState为established**
	* **ep->state为established**

到这里，普通点上关于接入点的两个caSession的状态都已经变成了established。握手过程结束，进入正常的会话过程。

### 流程图 ###
![image](/assets/img/cjdns-asymmetric-cryptography/img.jpg)