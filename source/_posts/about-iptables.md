---
title: cjdns源码分析--iptables在cjdns项目中的运用
category:
  - net
  - iptables
tags:
  - iptables
  - linux
description: "结合cjdns项目中，policy执行模块的实现，来分析一下简单的iptables命令"
date: 2017-04-27 11:20:05
---

## 关于iptables
###  简介
iptables是linux平台的网络访问控制管理系统，是一个包过滤防火墙。
它会检查数据包的相关信息，如源地址，目的地址，源端口，目的端口，协议类型等，结合规则的设置，来决定数据包的处理方法。

### 规则表（table）与规则链（chains）
#### 规则链
数据包会在如下五个位置经受检查：
1. 内核空间中：从一个网络接口进来，到另一个网络接口去
2. 数据包从内核流向用户空间
3. 数据包从用户空间流出
4. 进入/离开本机的外网接口
5. 进入/离开本机的内网接口
这五个位置对应着iptables的五个规则链
1. PREROUTING (路由前)
2. INPUT (数据包流入口)
3. FORWARD (转发管卡)
4. OUTPUT(数据包出口)
5. POSTROUTING（路由后）

从数据流向的角度来看，任何一个数据包，只要经过本机，一定会经过这五个链中的一个。
![image](/assets/img/about-iptables/links.png) 

- 第一种情况：入站数据流向

       从外界到达防火墙的数据包，先被PREROUTING规则链处理（是否修改数据包地址等），之后会进行路由选择（判断该数据包应该发往何处），如果数据包 的目标主机是防火墙本机（比如说Internet用户访问防火墙主机中的web服务器的数据包），那么内核将其传给INPUT链进行处理（决定是否允许通 过等），通过以后再交给系统上层的应用程序（比如Apache服务器）进行响应。

- 第二冲情况：转发数据流向

       来自外界的数据包到达防火墙后，首先被PREROUTING规则链处理，之后会进行路由选择，如果数据包的目标地址是其它外部地址（比如局域网用户通过网 关访问QQ站点的数据包），则内核将其传递给FORWARD链进行处理（是否转发或拦截），然后再交给POSTROUTING规则链（是否修改数据包的地 址等）进行处理。

- 第三种情况：出站数据流向
       防火墙本机向外部地址发送的数据包（比如在防火墙主机中测试公网DNS服务器时），首先被OUTPUT规则链处理，之后进行路由选择，然后传递给POSTROUTING规则链（是否修改数据包的地址等）进行处理。
比如，一个进入本机的数据包，首先进入PREROUTING链，然后进入INPUT链；离开本机的数据包，会经过OUTPUT链，然后进入POSTROUTING链；如果本机作为网络的网关，连接内外网，则经过本机的数据会进入FORWARD链。

#### 规则表tables
![ ](/assets/img/about-iptables/tables.png  "image")

iptables内置了4个表，即filter表、nat表、mangle表和raw表，分别用于实现包过滤，网络地址转换、包重构(修改)和数据跟踪处理。
 Iptables采用“表”和“链”的分层结构。在REHL4中是三张表五个链。REHL5中是四张表五个链。

#####  规则表：
1. filter表——三个链：INPUT、FORWARD、OUTPUT
作用：过滤数据包  内核模块：iptables_filter.
2. Nat表——三个链：PREROUTING、POSTROUTING、OUTPUT
作用：用于网络地址转换（IP、端口） 内核模块：iptable_nat
3. Mangle表——五个链：PREROUTING、POSTROUTING、INPUT、OUTPUT、FORWARD
作用：修改数据包的服务类型、TTL、并且可以配置路由实现QOS内核模块：iptable_mangle(别看这个表这么麻烦，咱们设置策略时几乎都不会用到它)
4. Raw表——两个链：OUTPUT、PREROUTING
作用：决定数据包是否被状态跟踪机制处理  内核模块：iptable_raw
(这个是REHL4没有的，不过用的不多)

** 规则表之间的优先顺序：**
Raw——mangle——nat——filter

### 基本规则语法
```
iptables [-t filter] [-AI INPUT,OUTPUT,FORWARD] [-io interface]
      [-p tcp,udp.icmp,all] [-s ip/nerwork] [--sport ports]
      [-d ip/netword] [--dport ports] [-j ACCEPT DROP]
```
常用命令列表：
命令 -A, --append
范例 iptables -A INPUT ...
说明 新增规则到某个规则链中，该规则将会成为规则链中的最后一条规则。
命令 -D, --delete
范例 iptables -D INPUT --dport 80 -j DROP
iptables -D INPUT 1
说明 从某个规则链中删除一条规则，可以输入完整规则，或直接指定规则编号加以删除。
命令 -R, --replace
范例 iptables -R INPUT 1 -s 192.168.0.1 -j DROP
说明 取代现行规则，规则被取代后并不会改变顺序。
命令 -I, --insert
范例 iptables -I INPUT 1 --dport 80 -j ACCEPT
说明 插入一条规则，原本该位置上的规则将会往后移动一个顺位。
命令 -L, --list
范例 iptables -L INPUT
说明 列出某规则链中的所有规则。
命令 -F, --flush
范例 iptables -F INPUT
说明 删除某规则链中的所有规则。
命令 -Z, --zero
范例 iptables -Z INPUT
说明 将封包计数器归零。封包计数器是用来计算同一封包出现次数，是过滤阻断式攻击不可或缺的工具。
命令 -N, --new-chain
范例 iptables -N allowed
说明 定义新的规则链。
命令 -X, --delete-chain
范例 iptables -X allowed
说明 删除某个规则链。
命令 -P, --policy
范例 iptables -P INPUT DROP
说明 定义过滤政策。 也就是未符合过滤条件之封包，预设的处理方式。
命令 -E, --rename-chain
范例 iptables -E allowed disallowed
说明 修改某自订规则链的名称。
常用封包比对参数：
参数 -p, --protocol
范例 iptables -A INPUT -p tcp
说明 比对通讯协议类型是否相符，可以使用 ! 运算子进行反向比对，例如：-p ! tcp ，意思是指除 tcp 以外的其它类型，包含udp、icmp ...等。如果要比对所有类型，则可以使用 all 关键词，例如：-p all。
参数 -s, --src, --source
范例 iptables -A INPUT -s 192.168.1.1
说明 用来比对封包的来源 IP，可以比对单机或网络，比对网络时请用数字来表示屏蔽，例如：-s 192.168.0.0/24，比对 IP 时可以使用 ! 运算子进行反向比对，例如：-s ! 192.168.0.0/24。
参数 -d, --dst, --destination
范例 iptables -A INPUT -d 192.168.1.1
说明 用来比对封包的目的地 IP，设定方式同上。
参数 -i, --in-interface
范例 iptables -A INPUT -i eth0
说明 用来比对封包是从哪片网卡进入，可以使用通配字符 + 来做大范围比对，例如：-i eth+ 表示所有的 ethernet 网卡，也以使用 ! 运算子进行反向比对，例如：-i ! eth0。
参数 -o, --out-interface
范例 iptables -A FORWARD -o eth0
说明 用来比对封包要从哪片网卡送出，设定方式同上。
参数 --sport, --source-port
范例 iptables -A INPUT -p tcp --sport 22
说明 用来比对封包的来源埠号，可以比对单一埠，或是一个范围，例如：--sport 22:80，表示从 22 到 80 埠之间都算是符合件，如果要比对不连续的多个埠，则必须使用 --multiport 参数，详见后文。比对埠号时，可以使用 ! 运算子进行反向比对。
参数 --dport, --destination-port
范例 iptables -A INPUT -p tcp --dport 22
说明 用来比对封包的目的地埠号，设定方式同上。
参数 --tcp-flags
范例 iptables -p tcp --tcp-flags SYN,FIN,ACK SYN
说明 比对 TCP 封包的状态旗号，参数分为两个部分，第一个部分列举出想比对的旗号，第二部分则列举前述旗号中哪些有被设，未被列举的旗号必须是空的。TCP 状态旗号包括：SYN（同步）、ACK（应答）、FIN（结束）、RST（重设）、URG（紧急）
PSH（强迫推送） 等均可使用于参数中，除此之外还可以使用关键词 ALL 和 NONE 进行比对。比对旗号时，可以使用 ! 运算子行反向比对。
参数 --syn
范例 iptables -p tcp --syn
说明 用来比对是否为要求联机之 TCP 封包，与 iptables -p tcp --tcp-flags SYN,FIN,ACK SYN 的作用完全相同，如果使用 !运算子，可用来比对非要求联机封包。
参数 -m multiport --source-port
范例 iptables -A INPUT -p tcp -m multiport --source-port 22,53,80,110
说明 用来比对不连续的多个来源埠号，一次最多可以比对 15 个埠，可以使用 ! 运算子进行反向比对。
参数 -m multiport --destination-port
范例 iptables -A INPUT -p tcp -m multiport --destination-port 22,53,80,110
说明 用来比对不连续的多个目的地埠号，设定方式同上。
参数 -m multiport --port
范例 iptables -A INPUT -p tcp -m multiport --port 22,53,80,110
说明 这个参数比较特殊，用来比对来源埠号和目的埠号相同的封包，设定方式同上。注意：在本范例中，如果来源端口号为 80 目的地埠号为 110，这种封包并不算符合条件。
参数 --icmp-type
范例 iptables -A INPUT -p icmp --icmp-type 8
说明 用来比对 ICMP 的类型编号，可以使用代码或数字编号来进行比对。请打 iptables -p icmp --help 来查看有哪些代码可用。
参数 -m limit --limit
范例 iptables -A INPUT -m limit --limit 3/hour
说明 用来比对某段时间内封包的平均流量，上面的例子是用来比对：每小时平均流量是否超过一次 3 个封包。 除了每小时平均次外，也可以每秒钟、每分钟或每天平均一次，默认值为每小时平均一次，参数如后： /second、 /minute、/day。 除了进行封
数量的比对外，设定这个参数也会在条件达成时，暂停封包的比对动作，以避免因骇客使用洪水攻击法，导致服务被阻断。
参数 --limit-burst
范例 iptables -A INPUT -m limit --limit-burst 5
说明 用来比对瞬间大量封包的数量，上面的例子是用来比对一次同时涌入的封包是否超过 5 个（这是默认值），超过此上限的封将被直接丢弃。使用效果同上。
参数 -m mac --mac-source
范例 iptables -A INPUT -m mac --mac-source 00:00:00:00:00:01
说明 用来比对封包来源网络接口的硬件地址，这个参数不能用在 OUTPUT 和 Postrouting 规则炼上，这是因为封包要送出到网后，才能由网卡驱动程序透过 ARP 通讯协议查出目的地的 MAC 地址，所以 iptables 在进行封包比对时，并不知道封包会送到个网络接口去。
参数 --mark
范例 iptables -t mangle -A INPUT -m mark --mark 1
说明 用来比对封包是否被表示某个号码，当封包被比对成功时，我们可以透过 MARK 处理动作，将该封包标示一个号码，号码最不可以超过 4294967296。
参数 -m owner --uid-owner
范例 iptables -A OUTPUT -m owner --uid-owner 500
说明 用来比对来自本机的封包，是否为某特定使用者所产生的，这样可以避免服务器使用 root 或其它身分将敏感数据传送出，可以降低系统被骇的损失。可惜这个功能无法比对出来自其它主机的封包。
参数 -m owner --gid-owner
范例 iptables -A OUTPUT -m owner --gid-owner 0
说明 用来比对来自本机的封包，是否为某特定使用者群组所产生的，使用时机同上。
参数 -m owner --pid-owner
范例 iptables -A OUTPUT -m owner --pid-owner 78
说明 用来比对来自本机的封包，是否为某特定行程所产生的，使用时机同上。
参数 -m owner --sid-owner
范例 iptables -A OUTPUT -m owner --sid-owner 100
说明 用来比对来自本机的封包，是否为某特定联机（Session ID）的响应封包，使用时机同上。
参数 -m state --state
范例 iptables -A INPUT -m state --state RELATED,ESTABLISHED
说明 用来比对联机状态，联机状态共有四种：INVALID、ESTABLISHED、NEW 和 RELATED。

INVALID 表示该封包的联机编号（Session ID）无法辨识或编号不正确。
ESTABLISHED 表示该封包属于某个已经建立的联机。
NEW 表示该封包想要起始一个联机（重设联机或将联机重导向）。
RELATED 表示该封包是属于某个已经建立的联机，所建立的新联机。例如：FTP-DATA 联机必定是源自某个 FTP 联机。

常用的处理动作：
-j 参数用来指定要进行的处理动作，常用的处理动作包括：ACCEPT、REJECT、DROP、REDIRECT、MASQUERADE、LOG、DNAT、

SNAT、MIRROR、QUEUE、RETURN、MARK，分别说明如下：
ACCEPT 将封包放行，进行完此处理动作后，将不再比对其它规则，直接跳往下一个规则炼（natostrouting）。
REJECT 拦阻该封包，并传送封包通知对方，可以传送的封包有几个选择：ICMP port-unreachable、ICMP echo-reply 或是 
tcp-reset（这个封包会要求对方关闭联机），进行完此处理动作后，将不再比对其它规则，直接 中断过滤程序。 范例如下：
iptables -A FORWARD -p TCP --dport 22 -j REJECT --reject-with tcp-reset
DROP 丢弃封包不予处理，进行完此处理动作后，将不再比对其它规则，直接中断过滤程序。
REDIRECT 将封包重新导向到另一个端口（PNAT），进行完此处理动作后，将 会继续比对其它规则。 这个功能可以用来实作通透式
porxy 或用来保护 web 服务器。例如：iptables -t nat -A PREROUTING -p tcp --dport 80 -j REDIRECT --to-ports 8080
MASQUERADE 改写封包来源 IP 为防火墙 NIC IP，可以指定 port 对应的范围，进行完此处理动作后，直接跳往下一个规则（mangleostrouting）。这个功能与 SNAT 略有不同，当进行 IP 伪装时，不需指定要伪装成哪个 IP，IP 会从网卡直接读，当使用拨接连线时，IP 通常是由 ISP 公司的 DHCP 服务器指派的，这个时候 MASQUERADE 特别有用。范例如下：
iptables -t nat -A POSTROUTING -p TCP -j MASQUERADE --to-ports 1024-31000
LOG 将封包相关讯息纪录在 /var/log 中，详细位置请查阅 /etc/syslog.conf 组态档，进行完此处理动作后，将会继续比对其规则。例如：
iptables -A INPUT -p tcp -j LOG --log-prefix "INPUT packets"
SNAT 改写封包来源 IP 为某特定 IP 或 IP 范围，可以指定 port 对应的范围，进行完此处理动作后，将直接跳往下一个规则（mangleostrouting）。范例如下：
iptables -t nat -A POSTROUTING -p tcp-o eth0 -j SNAT --to-source 194.236.50.155-194.236.50.160:1024-32000
DNAT 改写封包目的地 IP 为某特定 IP 或 IP 范围，可以指定 port 对应的范围，进行完此处理动作后，将会直接跳往下一个规炼（filter:input 或 filter:forward）。范例如下：
iptables -t nat -A PREROUTING -p tcp -d 15.45.23.67 --dport 80 -j DNAT --to-destination
192.168.1.1-192.168.1.10:80-100
MIRROR 镜射封包，也就是将来源 IP 与目的地 IP 对调后，将封包送回，进行完此处理动作后，将会中断过滤程序。
QUEUE 中断过滤程序，将封包放入队列，交给其它程序处理。透过自行开发的处理程序，可以进行其它应用，例如：计算联机费.......等。
RETURN 结束在目前规则炼中的过滤程序，返回主规则炼继续过滤，如果把自订规则炼看成是一个子程序，那么这个动作，就相当提早结束子程序并返回到主程序中。
MARK 将封包标上某个代号，以便提供作为后续过滤的条件判断依据，进行完此处理动作后，将会继续比对其它规则。范例如下：
iptables -t mangle -A PREROUTING -p tcp --dport 22 -j MARK --set-mark 2

### 自定义规则链
自定义链，最终要应用于默认链上，才能起作用。
iptables -N CJDNS_INPUT    --自定义一条名为CJDNS_INPUT的规则链
iptables -I INPUT -j CJDNS_INPUT;  --将CJDNS_INPUT应用到INPUT规则链上

## cjdns项目中的应用
### rule_flush方法中的使用
#### 源码
```
static int Policy_firewall_flush_linux(struct Policy_pvt* policy)
{
    int ret = 0;
    if (policy->ifName == NULL) {
        return ret;
    }

    /* Remove all old rules */
    const char* flushAll = "iptables -F CJDNS_INPUT;"
                         "iptables -D INPUT -j CJDNS_INPUT;"
                         "iptables -X CJDNS_INPUT;"
                         "iptables -F CJDNS_OUTPUT;"
                         "iptables -D OUTPUT -j CJDNS_OUTPUT;"
                         "iptables -X CJDNS_OUTPUT;"
                         "ip6tables -F CJDNS_INPUT;"
                         "ip6tables -D INPUT -j CJDNS_INPUT;"
                         "ip6tables -X CJDNS_INPUT;"
                         "ip6tables -F CJDNS_OUTPUT;"
                         "ip6tables -D OUTPUT -j CJDNS_OUTPUT;"
                         "ip6tables -X CJDNS_OUTPUT;";
    ret = system(flushAll);
    return ret;
}
```
#### 命令分析
* iptables -F CJDNS_INPUT;
删除CJDNS_INPUT规则链中的所有规则
* iptables -D INPUT -j CJDNS_INPUT;
将CJDNS_INPUT规则链从INPUT规则链上删除
* iptables -X CJDNS_INPUT;
删除CJDNS_INPUT规则链
```
iptables -F CJDNS_OUTPUT;
iptables -D OUTPUT -j CJDNS_OUTPUT;
iptables -X CJDNS_OUTPUT;
```
与上面三条类似
```
"ip6tables -F CJDNS_INPUT;"
"ip6tables -D INPUT -j CJDNS_INPUT;"
"ip6tables -X CJDNS_INPUT;"
"ip6tables -F CJDNS_OUTPUT;"
"ip6tables -D OUTPUT -j CJDNS_OUTPUT;"
"ip6tables -X CJDNS_OUTPUT;";
```
与上面六条类似，但作用于ipv6

### rule_apply方法中的使用
```
static int Policy_firewall_linux(struct Policy_pvt* policy,
                                  struct Policy_RuleList* rules)
{
    int ret = 0;
    if (policy->ifName == NULL) {
        return ret;
    }

    /* Remove all old rules */
    const char* initAll = "iptables -F CJDNS_INPUT;"
                         "iptables -D INPUT -j CJDNS_INPUT;"
                         "iptables -X CJDNS_INPUT;"
                         "iptables -N CJDNS_INPUT;"
                         "iptables -I INPUT -j CJDNS_INPUT;"
                         "iptables -F CJDNS_OUTPUT;"
                         "iptables -D OUTPUT -j CJDNS_OUTPUT;"
                         "iptables -N CJDNS_OUTPUT;"
                         "iptables -I OUTPUT -j CJDNS_OUTPUT;"
                         "ip6tables -F CJDNS_INPUT;"
                         "ip6tables -D INPUT -j CJDNS_INPUT;"
                         "ip6tables -X CJDNS_INPUT;"
                         "ip6tables -N CJDNS_INPUT;"
                         "ip6tables -I INPUT -j CJDNS_INPUT;"
                         "ip6tables -F CJDNS_OUTPUT;"
                         "ip6tables -D OUTPUT -j CJDNS_OUTPUT;"
                         "ip6tables -X CJDNS_OUTPUT;"
                         "ip6tables -N CJDNS_OUTPUT;"
                         "ip6tables -I OUTPUT -j CJDNS_OUTPUT;";
    ret = system(initAll);

    const char* fmt = "%s -A %s %s -s %s -d %s -j %s";
    char buff[4096];
    char ip[40];
    char addr[64];
    char iface[256], oface[256];
    char* localip = NULL;
    char* remoteip = NULL;
    char* action;
    char* dir;
    char* cmd;
    char* intf;

    Bits_memset(iface, 0, sizeof(iface));
    Bits_memset(oface, 0, sizeof(iface));
    snprintf(iface, sizeof(iface) - 1, "-i %s", policy->ifName->bytes);
    snprintf(oface, sizeof(oface) - 1, "-o %s", policy->ifName->bytes);

    for (int i = 0; i < rules->length; ++i) {
        struct Rule* rule = rules->rules[i];
        if (rule->location == Policy_INPUT) {
            dir = "CJDNS_INPUT";
            intf = iface;
        } else {
            dir = "CJDNS_OUTPUT";
            intf = oface;
        }

        if (rule->elem.af == Sockaddr_AF_INET) {
            cmd = "iptables";
        } else if (rule->elem.af == Sockaddr_AF_INET6){
            cmd = "ip6tables";
        } else {
            continue;
        }

        if (rule->elem.sprefix == 0) {
            if (rule->elem.af == Sockaddr_AF_INET) {
                localip = "0.0.0.0/0";
            } else {
                localip = "::/0";
            }
        } else {
            Bits_memset(ip, 0, sizeof(ip));
            if (rule->elem.af == Sockaddr_AF_INET) {
                snprintf(addr, sizeof(addr), "%u.%u.%u.%u/%u",
                         ((uint8_t *)&rule->elem.src)[0],
                         ((uint8_t *)&rule->elem.src)[1],
                         ((uint8_t *)&rule->elem.src)[2],
                         ((uint8_t *)&rule->elem.src)[3],
                         rule->elem.sprefix);
                localip = addr;
            } else if (rule->elem.af == Sockaddr_AF_INET6) {
                AddrTools_printIp(ip, rule->elem.src.ip);
                snprintf(addr, sizeof(addr), "%s/%u", ip, rule->elem.sprefix);
                localip = addr;
            }
        }
        if (rule->elem.dprefix == 0) {
            if (rule->elem.af == Sockaddr_AF_INET) {
                remoteip = "0.0.0.0/0";
            } else {
                remoteip = "::/0";
            }
        } else {
            Bits_memset(ip, 0, sizeof(ip));
            if (rule->elem.af == Sockaddr_AF_INET) {
                snprintf(addr, sizeof(addr), "%u.%u.%u.%u/%u",
                         ((uint8_t *)&rule->elem.dst)[0],
                         ((uint8_t *)&rule->elem.dst)[1],
                         ((uint8_t *)&rule->elem.dst)[2],
                         ((uint8_t *)&rule->elem.dst)[3],
                         rule->elem.dprefix);
                remoteip = addr;
            } else if (rule->elem.af == Sockaddr_AF_INET6) {
                AddrTools_printIp(ip, rule->elem.dst.ip);
                snprintf(addr, sizeof(addr), "%s/%u", ip, rule->elem.dprefix);
                remoteip = addr;
            }
        }
        if (rule->action == Rule_ACCEPT) {
            action = "ACCEPT";
        } else {
            action = "DROP";
        }
        snprintf(buff, sizeof(buff), fmt, cmd, dir, intf, localip, remoteip, action);
        ret = system(buff);
    }

    /* firewall header */
    snprintf(buff, sizeof(buff),
             "iptables -I CJDNS_INPUT %s -m state --state RELATED,ESTABLISHED -j ACCEPT",
             iface);
    ret = system(buff);
    snprintf(buff, sizeof(buff),
             "ip6tables -I CJDNS_INPUT %s -m state --state RELATED,ESTABLISHED -j ACCEPT",
             iface);
    ret = system(buff);
    snprintf(buff, sizeof(buff),
             "iptables -I CJDNS_OUTPUT %s -m state --state RELATED,ESTABLISHED -j ACCEPT",
             oface);
    ret = system(buff);
    snprintf(buff, sizeof(buff),
             "ip6tables -I CJDNS_OUTPUT %s -m state --state RELATED,ESTABLISHED -j ACCEPT",
             oface);
    ret = system(buff);

    /* firewall footer */
    if (policy->pub.defaultAction == Policy_ACCEPT) {
        action = "ACCEPT";
    } else {
        action = "DROP";
    }
    snprintf(buff, sizeof(buff), "iptables -A CJDNS_INPUT %s -j %s", iface, action);
    ret = system(buff);
    snprintf(buff, sizeof(buff), "ip6tables -A CJDNS_INPUT %s -j %s", iface, action);
    ret = system(buff);
    snprintf(buff, sizeof(buff), "iptables -A CJDNS_OUTPUT %s -j %s", oface, action);
    ret = system(buff);
    snprintf(buff, sizeof(buff), "ip6tables -A CJDNS_OUTPUT %s -j %s", oface, action);
    ret = system(buff);

    return ret;
}
```
这个方法中，对于iptables命令的使用，主要分为四块：
#### Remove all old rules
```
"iptables -F CJDNS_INPUT;"
"iptables -D INPUT -j CJDNS_INPUT;"
"iptables -X CJDNS_INPUT;"
"iptables -N CJDNS_INPUT;"
"iptables -I INPUT -j CJDNS_INPUT;"
"iptables -F CJDNS_OUTPUT;"
"iptables -D OUTPUT -j CJDNS_OUTPUT;"
"iptables -N CJDNS_OUTPUT;"
"iptables -I OUTPUT -j CJDNS_OUTPUT;"
"ip6tables -F CJDNS_INPUT;"
"ip6tables -D INPUT -j CJDNS_INPUT;"
"ip6tables -X CJDNS_INPUT;"
"ip6tables -N CJDNS_INPUT;"
"ip6tables -I INPUT -j CJDNS_INPUT;"
"ip6tables -F CJDNS_OUTPUT;"
"ip6tables -D OUTPUT -j CJDNS_OUTPUT;"
"ip6tables -X CJDNS_OUTPUT;"
"ip6tables -N CJDNS_OUTPUT;"
"ip6tables -I OUTPUT -j CJDNS_OUTPUT;";
```
以CJDNS_INPUT为例，这一部分首先做了和rule_flush一样的操作，删除相关数据
```
"iptables -F CJDNS_INPUT;"
"iptables -D INPUT -j CJDNS_INPUT;"
"iptables -X CJDNS_INPUT;"
```
然后，
新建自定义规则链CJDNS_INPUT
```
iptables -N CJDNS_INPUT;
```
将它应用到INPUT规则链上
```
iptables -I INPUT -j CJDNS_INPUT;
```

#### 根据规则，拼接iptables命令
```
    const char* fmt = "%s -A %s %s -s %s -d %s -j %s";
    char buff[4096];
    char ip[40];
    char addr[64];
    char iface[256], oface[256];
    char* localip = NULL;
    char* remoteip = NULL;
    char* action;
    char* dir;
    char* cmd;
    char* intf;

    Bits_memset(iface, 0, sizeof(iface));
    Bits_memset(oface, 0, sizeof(iface));
    snprintf(iface, sizeof(iface) - 1, "-i %s", policy->ifName->bytes);
    snprintf(oface, sizeof(oface) - 1, "-o %s", policy->ifName->bytes);

    for (int i = 0; i < rules->length; ++i) {
        struct Rule* rule = rules->rules[i];
        if (rule->location == Policy_INPUT) {
            dir = "CJDNS_INPUT";
            intf = iface;
        } else {
            dir = "CJDNS_OUTPUT";
            intf = oface;
        }

        if (rule->elem.af == Sockaddr_AF_INET) {
            cmd = "iptables";
        } else if (rule->elem.af == Sockaddr_AF_INET6){
            cmd = "ip6tables";
        } else {
            continue;
        }

        if (rule->elem.sprefix == 0) {
            if (rule->elem.af == Sockaddr_AF_INET) {
                localip = "0.0.0.0/0";
            } else {
                localip = "::/0";
            }
        } else {
            Bits_memset(ip, 0, sizeof(ip));
            if (rule->elem.af == Sockaddr_AF_INET) {
                snprintf(addr, sizeof(addr), "%u.%u.%u.%u/%u",
                         ((uint8_t *)&rule->elem.src)[0],
                         ((uint8_t *)&rule->elem.src)[1],
                         ((uint8_t *)&rule->elem.src)[2],
                         ((uint8_t *)&rule->elem.src)[3],
                         rule->elem.sprefix);
                localip = addr;
            } else if (rule->elem.af == Sockaddr_AF_INET6) {
                AddrTools_printIp(ip, rule->elem.src.ip);
                snprintf(addr, sizeof(addr), "%s/%u", ip, rule->elem.sprefix);
                localip = addr;
            }
        }
        if (rule->elem.dprefix == 0) {
            if (rule->elem.af == Sockaddr_AF_INET) {
                remoteip = "0.0.0.0/0";
            } else {
                remoteip = "::/0";
            }
        } else {
            Bits_memset(ip, 0, sizeof(ip));
            if (rule->elem.af == Sockaddr_AF_INET) {
                snprintf(addr, sizeof(addr), "%u.%u.%u.%u/%u",
                         ((uint8_t *)&rule->elem.dst)[0],
                         ((uint8_t *)&rule->elem.dst)[1],
                         ((uint8_t *)&rule->elem.dst)[2],
                         ((uint8_t *)&rule->elem.dst)[3],
                         rule->elem.dprefix);
                remoteip = addr;
            } else if (rule->elem.af == Sockaddr_AF_INET6) {
                AddrTools_printIp(ip, rule->elem.dst.ip);
                snprintf(addr, sizeof(addr), "%s/%u", ip, rule->elem.dprefix);
                remoteip = addr;
            }
        }
        if (rule->action == Rule_ACCEPT) {
            action = "ACCEPT";
        } else {
            action = "DROP";
        }
        snprintf(buff, sizeof(buff), fmt, cmd, dir, intf, localip, remoteip, action);
        ret = system(buff);
    }
```
在这一过程中，会根据策略规则，获取相关信息，最终拼接出格式如下的规则。
```
iptables -A CJDNS_INPUT -i tun0 -s srcip/srcport -d dstip/dstport -j ACCEPT
```

#### firewall header
```
    snprintf(buff, sizeof(buff),
             "iptables -I CJDNS_INPUT %s -m state --state RELATED,ESTABLISHED -j ACCEPT",
             iface);
    ret = system(buff);
    snprintf(buff, sizeof(buff),
             "ip6tables -I CJDNS_INPUT %s -m state --state RELATED,ESTABLISHED -j ACCEPT",
             iface);
    ret = system(buff);
    snprintf(buff, sizeof(buff),
             "iptables -I CJDNS_OUTPUT %s -m state --state RELATED,ESTABLISHED -j ACCEPT",
             oface);
    ret = system(buff);
    snprintf(buff, sizeof(buff),
             "ip6tables -I CJDNS_OUTPUT %s -m state --state RELATED,ESTABLISHED -j ACCEPT",
             oface);
    ret = system(buff);
```
-I 参数的规则默认是插入到规则表最上面的，也就是说，这四条规则是最先进行匹配的。
从这四条规则可以看出，连接一旦进入RELATED或ESTABLISHED状态，就ACCEPT，不再进行策略审查。换言之，策略检查都是针对NEW状态的连接数据包的。

#### firewall footer
```
    if (policy->pub.defaultAction == Policy_ACCEPT) {
        action = "ACCEPT";
    } else {
        action = "DROP";
    }
    snprintf(buff, sizeof(buff), "iptables -A CJDNS_INPUT %s -j %s", iface, action);
    ret = system(buff);
    snprintf(buff, sizeof(buff), "ip6tables -A CJDNS_INPUT %s -j %s", iface, action);
    ret = system(buff);
    snprintf(buff, sizeof(buff), "iptables -A CJDNS_OUTPUT %s -j %s", oface, action);
    ret = system(buff);
    snprintf(buff, sizeof(buff), "ip6tables -A CJDNS_OUTPUT %s -j %s", oface, action);
    ret = system(buff);
```
最后这四条用于设置默认操作，-A参数保证它们会加入到规则表的最后。
