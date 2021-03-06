# About

使用ss-redir搭建二级代理

## 步骤

以下环境为ubuntu 18.04容器

### 使用一级ss代理建立转发

```
# echo > ss2redir.json <-EOF
{
"server":"<address>",
"server_port":<port>,
"local_address": "0.0.0.0",
"local_port":1081,
"password":"<password>",
"timeout":600,
"method":"aes-256-cfb"
}
EOF

# ss-redir -c ss2redir.json &
```

### 使用 ipset 建立直接列表

不需要转发的流量包括私网网段，包括ss服务的地址，也可以放到iptables去过滤

```
# ipset create route_direct hash:net
# ipset add route_direct 192.168.0.0/16
# ipset add route_direct 127.0.0.0/8
# ipset add route_direct 10.0.0.0/8
# ipset add route_direct 172.16.0.0/12
# ipset add route_direct <ss_server>
......
```

### 使用 iptables 过滤流量

如果是远程操作，记得加上ssh client的地址，否则被重定向了，就无法ssh了

应用iptables 建议使用 iptables-apply，防止误操作

```
# Generated by iptables-save v1.6.1
*nat
:PREROUTING ACCEPT [10777:646620]
:INPUT ACCEPT [10777:646620]
:OUTPUT ACCEPT [10716:643018]
:POSTROUTING ACCEPT [21410:1284658]
:SS - [0:0]
-A OUTPUT -j SS
-A SS -d <ssh_client>/32 -j RETURN
-A SS -p tcp -m tcp --sport 22 -j RETURN
-A SS -m set --match-set route_direct dst -j RETURN
-A SS -p tcp -m tcp -j REDIRECT --to-ports 1081
COMMIT
```

### 使用DoH防止DNS污染

DoH使用tcp，所以可以被上面的tcp重定向到梯子上

下载对应 [cloudflared](https://github.com/cloudflare/cloudflared/releases) 版本，解压运行

```
# wget https://github.com/cloudflare/cloudflared/releases/download/2021.4.0/cloudflared-linux-amd64
```

修改/etc/resolv.conf中的nameserver为127.0.0.1，如果之前有本地dns服务，先停止

```
# ./cloudflared-linux-amd64 proxy-dns --address 0.0.0.0 &
```

测试

```
# dig ipv4.appspot.com -t A

; <<>> DiG 9.11.3-1ubuntu1.14-Ubuntu <<>> ipv4.appspot.com -t A
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 14292
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
; COOKIE: b29b3bce4122188d (echoed)
;; QUESTION SECTION:
;ipv4.appspot.com.              IN      A

;; ANSWER SECTION:
ipv4.appspot.com.       205     IN      A       172.217.174.212

```

通过tcpdump可以看到，没有去公网的udp包了

### 搭建本地SS服务

安装 shadowsocks-libev 搭建多用户的 ss 服务

或者使用 [Web](https://github.com/shadowsocks/shadowsocks-manager)

### 测试

```
ss-local -c sslocal.json
curl --socks5-hostname 127.0.0.1:1082 https://ipv4.appspot.com/
```

