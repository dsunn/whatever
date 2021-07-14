---
title: "Debian搭建clash透明代理"
date: 2021-07-13T22:21:40+08:00
---


写在前面:
为使得clash设计的dns防污染能更好地工作, 客户端的dns需要指向clash dns, 可以使用dhcp serve配置此项. 这样做的目的是使clash能够拿到连接的域名信息, 再配以适当的域名规则, 很多情形下, 即使在dns污染的情况下, 也能正常工作.




记录一下自己的搭建过程记录一下自己的搭建过程

最好用root用户运行, 特别是后面用clash premium installer搭建时, 不用root会报错

一. clash程序
- 下载clash premium (https://github.com/Dreamacro/clash/releases/tag/premium) 解压保存为/usr/local/bin/clash
- 设置为systemd deamon
              

编辑/lib/systemd/system/clash.service

Description=A rule based proxy tunnel in Go  
After=network.target  
[Service]  
Type=simple  
Restart=on-abort  
ExecStart=/usr/local/bin/clash  
[Install]  
WantedBy=multi-user.target  

默认运行目录为~/.config/clash, config.yaml要存放在工作目录中

systemctl daemon-reload  
systemctl enable clash  

二. 透明代理

开启ipv4/ipv6的转发  
echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf && sysctl -p  
echo "net.ipv6.conf.all.forwarding = 1" >> /etc/sysctl.conf && sysctl -p

关闭systemd-resolved 服务, 让clash占有53端口  
systemctl disable systemd-resolved  
/etc/reslov.conf里我是设置为127.0.0.1  

iptables持久化  
apt install iptables-persistent netfilter-persistent  
netfilter-persistent save  
netfilter-persistent start  
iptables-save  > /etc/iptables/rules.v4  
ip6tables-save  > /etc/iptables/rules.v6  

加载恢复  
iptables-restore  < /etc/iptables/rules.v4  
ip6tables-restore  < /etc/iptables/rules.v6  


三. iptables

推荐模式3和4

1. redirect模式
   这模式下google home时有something went wrong, 可能其他google原生设备/app也会时有报错, 网上说可能google app内置查询设定了dns.
2. 准tproxy模式
   但没用到tproxy端口, 依然用的是redir port: 7892, 我也不清楚是不是真正的tproxy模式. 这种模式联网都比较快, 但ch能进房间却没有声音, google app工作正常, 在yacd连接状况中显示为redir
3. tproxy模式
   使用tproxy port: 7893, 也是全局代理; ch工作正常, google app工作正常, 在yacd连接状况中显示为tproxy
4. tun模式
   虚拟了tun网卡, 也称为全局代理, ch等都正常
5. 所有模式都要排除clash主机自己的流量, 为防止循环(如192.168.10.87这句)

四. 注意点
1. config.yaml中开启redir port: 7892, tproxy port: 7893, 开启clash dns, 监听port: 53
2. ipv6的dns解析开启, 据说可能对翻墙有影响(慢..?), 不如我开启了解析后, 并没特别的感觉,后来发现我的ipv6流量走的默认网关是ros, 没走到clash; 目前还没找到怎么象v4的dns一样设置ipv6能自动指定客户的网关和dns, 如果要让ipv6走网关, 现在还心须在客户机上手动设置. 

五. clash dns

clash dns中, nameserver采用114和233, fallback必须采用无污染反投毒的dns, 也就是国外的dns如8.8.8.8, 1.0.0.1等, 至于是不是dot/doh我个人认为并不重要; 我认为dot/doh是保护隐私, 对于去污染反投毒并无额外帮助. tls://8.8.8.8不污染, 那udp的8.8.8.8:53也一样, 反之国内dns也不会因为加了dot, doh而得到清污. 只是, 问题在于, 目前无论是tcp/udp, 还是dot/doh, 国外的主流dns都已被墙, 而clash dns目前的设计是不走代理过墙的 (tun模式下劫持了dns是过墙的, 我试过在tun模式下fallback是不被查询到的); 那么, 如果要使得fullback能按clash设计要求正常工作, 就要在clash的外面搭建过墙的dns服务器, 走clash过墙, 然后充当fallback server. 当然, 按D大的说法, 即使不设置使用fallback, 只要规则配合得好, 也能让最后的流量绝大多数分流正常.

说下对clash dns作用的个人理解, clash dns分nameserver和fallback server二组, 各自可填若干dns, 并发查询; D大建议各组不超4个. 其中nameserver建议填写国内响应快的dns, 比如isp的, 或114等公共dns, 无需dot/doh; fallback需要无污染的dns. 当clash收到dns查询请求, clash将请求同时发给nameserver和fallback, 取二组返回的最快的结果, 与geoip-cn进行比对, 若:
- 结果是落在geoip-cn中(二组一致), 则取nameserver的解析结果, 备用
- 结果不是落在geoip-cn中(二组一致), 则取fallback的解析结果
- 查询结果与geoip-cn比对二组不一致, 即nameserver结果是geoip-cn而fallback不是, 或反之,  则取fallback的解析结果
clash dns比对后确定的解析结果, 只有在确定为直连时使用; 而若随后规则判定为代理的话, clash是发送FQDN到节点, 由节点解析连接, 这样有可能能获取对节点友好的cdn的ip.

讨论一下不设置fallback的情形. 按clash的设计, 即使域名被污染了, 如果有规则判定该域名走代理, 则还是发送FQDN到节点, 理论上还是能够正确连接. 另一种情形下, 如域名应该走代理, 但却并没有规则设定走代理, 这时即使nameserver污染了该域名, 只要对geoip-cn的对比是正确的话, 比方把google.com解析为facebook.com, 则clash最后的模式(一般是大陆白名单) 还是会判定该域名走代理; 但如果nameserver把google.com解析为国内ip或保留地址时, clash最后会用这个被污染的解析结果走直连, 导致无法正确访问.

用fake-ip时, 省掉一次dns查询, 根据域名规则, 如果走代理, 就把FQDN直接发给节点; 不走代理的, 再查询dns


-coredns

搭建过墙的dns server部分, 我参照了https://blog.minidump.info/2019/07/coredns-no-dns-poisoning/ 该文的方案, 这是采用coredns来搭建dns服务器(虚拟了一个centos7的lxc, 网关为clash以便能翻墙查询)

coredns (https://github.com/coredns/coredns), 有很多插件, 官方的和第三方的, 可方便用于dns分流, 该方案是采用按https://github.com/felixonmars/dnsmasq-china-list  按域名分流查询, 命中名单的用国内114/233或isp的dns查询, 没命中名单的用国外dns查询, 支持各种查询方式. coredns可自己编译, 也可下载minidump编译的内核.

目前clash dns, nameserver我采用114/233,  事实上我是在主路由ros上的dns设置了114/233, clash nameserver就指向ros的ip; fallback 指向coredns. 在此之上, 虚拟一个lxc的centos7搭建adguard home, 上游指向clash的ip, adg过滤一些广告(感觉作用不强), 还能管理host劫持, ipv6解析开关等. 这些dns套娃用下来, dns是慢了一些, 不过尚属可以接受(adg 平均处理时间10到30ms); 感觉dns的延迟主要是查询dot/doh造成的, 套娃只是forward, 并无有感的影响, 毕竟dns查询机制本来就是层层向上查询.

其他可能的dns方案:
a): 已实践通过.  nameserver设置为coredns, fallbabk不设置. 这种情形下, clash dns的解析就是nameserver的解析, 也就是coredns的解析结果; 由于coredns是按国内域名分流查询了, 原则上clash dns是未污染的.

上述其他方案的好处(即将nameserver设置为coredns)是: 国内的dns不会收到你对国内域名的查询; 而在我目前使用的方案(即nameserver为114/233, fallback为coredns)下, 由于clash dns会同时将查询forward到nameserver和fallback, 则国内dns也会收到所有的域名查询.

因此, 将coredns用于nameserver可以更好地保护隐私. 而且这种情形下不必再设置fallback了, 事实上我试过把nameserver和fullback同时设为coredns, clash反而报错了, 没深查原因. 只把nameserver设为coredns时, 也报错了...

---
redirect模式  

iptables -t nat -N clash  
iptables -t nat -A clash -d 0.0.0.0/8 -j RETURN  
iptables -t nat -A clash -d 10.0.0.0/8 -j RETURN  
iptables -t nat -A clash -d 127.0.0.0/8 -j RETURN  
iptables -t nat -A clash -d 169.254.0.0/16 -j RETURN  
iptables -t nat -A clash -d 172.16.0.0/12 -j RETURN  
iptables -t nat -A clash -d 192.168.0.0/16 -j RETURN  
iptables -t nat -A clash -d 224.0.0.0/4 -j RETURN  
iptables -t nat -A clash -d 240.0.0.0/4 -j RETURN  
iptables -t nat -A clash -d 192.168.10.87 -j RETURN  
iptables -t nat -A clash -p tcp -j RETURN -m mark --mark 0xff  
iptables -t nat -A clash -p tcp -j REDIRECT --to-ports 7892  
iptables -t nat -A PREROUTING -p tcp -j clash 

---
准tproxy模式  (所谓准tproxy模式是我认为的, 网上有的教程的tproxy模式就是类似这个)

iptables -t nat -N clash  
iptables -t nat -A clash -d 0.0.0.0/8 -j RETURN  
iptables -t nat -A clash -d 10.0.0.0/8 -j RETURN  
iptables -t nat -A clash -d 127.0.0.0/8 -j RETURN  
iptables -t nat -A clash -d 169.254.0.0/16 -j RETURN  
iptables -t nat -A clash -d 172.16.0.0/12 -j RETURN  
iptables -t nat -A clash -d 192.168.0.0/16 -j RETURN  
iptables -t nat -A clash -d 224.0.0.0/4 -j RETURN  
iptables -t nat -A clash -d 240.0.0.0/4 -j RETURN  
iptables -t nat -A clash -d 192.168.10.87 -j RETURN  
iptables -t nat -A clash -p tcp -j REDIRECT --to-port 7892  
iptables -t nat -I PREROUTING -p tcp -d 8.8.8.8 -j REDIRECT --to-port 7892  
iptables -t nat -I PREROUTING -p tcp -d 8.8.4.4 -j REDIRECT --to-port 7892  
iptables -t nat -A PREROUTING -p tcp -j clash  

ip rule add fwmark 1 table 100  
ip route add local default dev lo table 100  
iptables -t mangle -N clash  
iptables -t mangle -A clash -d 0.0.0.0/8 -j RETURN  
iptables -t mangle -A clash -d 10.0.0.0/8 -j RETURN  
iptables -t mangle -A clash -d 127.0.0.0/8 -j RETURN  
iptables -t mangle -A clash -d 169.254.0.0/16 -j RETURN  
iptables -t mangle -A clash -d 172.16.0.0/12 -j RETURN  
iptables -t mangle -A clash -d 192.168.0.0/16 -j RETURN  
iptables -t mangle -A clash -d 224.0.0.0/4 -j RETURN  
iptables -t mangle -A clash -d 240.0.0.0/4 -j RETURN  
iptables -t mangle -A clash -d 192.168.10.87 -j RETURN  
iptables -t mangle -A clash -p udp -j TPROXY --on-port 7892 --tproxy-mark 1  
iptables -t mangle -A PREROUTING -p udp -j clash 

---
tproxy模式 

ip rule add fwmark 1 table 100  
ip route add local default dev lo table 100  
iptables -t mangle -N clash  
iptables -t mangle -A clash -d 0.0.0.0/8 -j RETURN  
iptables -t mangle -A clash -d 10.0.0.0/8 -j RETURN  
iptables -t mangle -A clash -d 127.0.0.0/8 -j RETURN  
iptables -t mangle -A clash -d 169.254.0.0/16 -j RETURN  
iptables -t mangle -A clash -d 172.16.0.0/12 -j RETURN  
iptables -t mangle -A clash -d 192.168.0.0/16 -j RETURN  
iptables -t mangle -A clash -d 224.0.0.0/4 -j RETURN  
iptables -t mangle -A clash -d 240.0.0.0/4 -j RETURN  
iptables -t mangle -A clash -d 192.168.10.87 -j RETURN  
iptables -t mangle -A clash -p tcp -j TPROXY --on-port 7893 --tproxy-mark 1  
iptables -t mangle -A clash -p udp -j TPROXY --on-port 7893 --tproxy-mark 1  
iptables -t mangle -A clash -p tcp -j RETURN -m mark --mark 1  
iptables -t mangle -A clash -p udp -j RETURN -m mark --mark 1  
iptables -t mangle -A PREROUTING -p tcp -j clash  
iptables -t mangle -A PREROUTING -p udp -j clash  

---
tun模式
系installer安装时部署的, 略

---
clash dns
(此处1053端口是clash dns listening port, 它这种情况下53端口是dnsmasq占用的, 我的debian没安装dnsmasq所以没应用这一段)

iptables -t nat -N CLASH_DNS  
iptables -t nat -F CLASH_DNS   
iptables -t nat -A CLASH_DNS -p udp -j REDIRECT --to-port 1053  
iptables -t nat -I OUTPUT -p udp --dport 53 -j CLASH_DNS  
iptables -t nat -I PREROUTING -p udp --dport 53 -j REDIRECT --to 1053  

---
ipv6 (还没有成功过)
下面是我测试的, 但没成功, 不知是节点的问题, 还是iptables规则的问题

ipset create localnetwork6 hash:net family inet6  
ipset add localnetwork6 ::/128  
ipset add localnetwork6 ::1/128  
ipset add localnetwork6 ::ffff:0:0/96  
ipset add localnetwork6 ::ffff:0:0:0/96  
ipset add localnetwork6 64:ff9b::/96  
ipset add localnetwork6 100::/64  
ipset add localnetwork6 2001::/32  
ipset add localnetwork6 2001:20::/28  
ipset add localnetwork6 2001:db8::/32  
ipset add localnetwork6 2002::/16  
ipset add localnetwork6 fc00::/7  
ipset add localnetwork6 fe80::/10  
ipset add localnetwork6 ff00::/8  

ipset create clashhost6 hash:net family inet6  
ipset add clashhost6 fe80::d466:a5ff:fed9:6028/64  

ip6tables -t mangle -N clash  
ip6tables -t mangle -F clash  
ip6tables -t mangle -A clash -m set --match-set localnetwork6 dst -j RETURN  
ip6tables -t mangle -A clash -m set --match-set clashhost6 dst -j RETURN  

ip -6 rule add fwmark 0x162 table 0x162  
ip -6 route add local ::/0 dev lo table 0x162  

ip6tables -t mangle -A clash -p tcp -j TPROXY --on-port 7893 --tproxy-mark 0x162  
ip6tables -t mangle -A clash -p udp -j TPROXY --on-port 7893 --tproxy-mark 0x162  
ip6tables -t mangle -A clash -p tcp -j RETURN -m mark --mark 0x162  
ip6tables -t mangle -A clash -p udp -j RETURN -m mark --mark 0x162  
ip6tables -t mangle -A PREROUTING -p tcp -j clash  
ip6tables -t mangle -A PREROUTING -p udp -j clash  

---
用ipset简化一下规则

ipset create localnetwork hash:net  
ipset add localnetwork 127.0.0.0/8  
ipset add localnetwork 10.0.0.0/8  
ipset add localnetwork 192.168.0.0/16  
ipset add localnetwork 224.0.0.0/4  
ipset add localnetwork 172.16.0.0/12  
ipset add localnetwork 0.0.0.0/8  
ipset add localnetwork 169.254.0.0/16  
ipset add localnetwork 240.0.0.0/4  

ipset create clashhost hash:net  
ipset add clashhost 192.168.10.87  

-------------------------------------------

ipset create localnetwork6 hash:net family inet6  
ipset add localnetwork6 ::/128  
ipset add localnetwork6 ::1/128  
ipset add localnetwork6 ::ffff:0:0/96  
ipset add localnetwork6 ::ffff:0:0:0/96  
ipset add localnetwork6 64:ff9b::/96  
ipset add localnetwork6 100::/64  
ipset add localnetwork6 2001::/32  
ipset add localnetwork6 2001:20::/28  
ipset add localnetwork6 2001:db8::/32  
ipset add localnetwork6 2002::/16  
ipset add localnetwork6 fc00::/7  
ipset add localnetwork6 fe80::/10  
ipset add localnetwork6 ff00::/8  

ipset create clashhost6 hash:net family inet6  
ipset add clashhost6 fe80::d466:a5ff:fed9:6028/64  

---
在ipset后应用的iptables规则(tproxy模式, 不含ipv6部分)

ip rule add fwmark 1 table 100  
ip route add local default dev lo table 100  
iptables -t mangle -N clash  
iptables -t mangle -A clash -m set --match-set localnetwork dst -j RETURN  
iptables -t mangle -A clash -m set --match-set clashhost dst -j RETURN  
iptables -t mangle -A clash -p tcp -j TPROXY --on-port 7893 --tproxy-mark 1  
iptables -t mangle -A clash -p udp -j TPROXY --on-port 7893 --tproxy-mark 1  
iptables -t mangle -A clash -p tcp -j RETURN -m mark --mark 1  
iptables -t mangle -A clash -p udp -j RETURN -m mark --mark 1  
iptables -t mangle -A PREROUTING -p tcp -j clash  
iptables -t mangle -A PREROUTING -p udp -j clash  

---
另外在准tproxy模式下, dig @114.114.114.114 baidu.com 是不通的, 而 dig @192.168.10.251 baidu.com 是可行的, 可能是iptables规则的问题吧, 规则中192.168.10.251是绕过clash的, 114.114.114.114是经过clash, 什么原因导致不通我也不全懂; 其实在准tproxy模式下, coredns设置114.114.114.114也是有问题, (国外dns的没问题), 估计也是这原因; 但在tproxy模式下, 则无此问题.

---

研究一下(目前还没加入到规则中, 现在的规则还是用-m set --match-set clashhost dst -j RETURN)  
iptables -t nat -I OUTPUT -m owner ! --uid-owner clash -p udp --dport 53 -j CLASH_DNS  

用-m owner ! --uid-owner clash改写上述iptables中绕过clash流量的部分 (这里用clash通不过, 改为0, 不知对不对?)
->>  
iptables -t mangle -A clash -m owner --uid-owner 0 -j RETURN   #放行clash发出的数据  
iptables -t nat -A OUTPUT -m owner ! --uid-owner 0 -j clash #转发本机非clash的流量  

(以上二行我在加入到tproxy的iptables规则中时报错, 第二条可能是原规则没有新建OUTPUT的链吧, 这我不是很懂, 第一条报错不懂什么原因)

kr328的脚本中用cgroup PID来控制clash的流量的方法, 可能是更好的, 还要研究一下cgroup的管理命令

