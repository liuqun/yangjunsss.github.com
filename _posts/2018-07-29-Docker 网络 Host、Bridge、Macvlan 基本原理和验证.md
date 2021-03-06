---
description: "
Docker 作为容器的主流平台，不仅仅提供了虚拟化隔离，同时也配备的网络隔离技术，并使用不同的网络驱动满足不同的场景，这篇文章对 Docker 的3种网络实现Host、Bridge、Macvlan进行模拟验证，并在实践中理解背后的基本原理。
"
---

## 摘要

Docker 作为容器的主流平台，不仅仅提供了虚拟化隔离，同时也配备的网络隔离技术，并使用不同的网络驱动满足不同的场景，这篇文章对 Docker 的3种网络实现Host、Bridge、Macvlan进行模拟验证，并在实践中理解背后的基本原理。

## Host 模式
Host 模式为容器实例直接使用 Host 网络能力，与 Host 共享网卡、路由、转发表等，不创建 netns，不进行隔离，如容器实例绑定了 80 端口，则可以通过访问 Host 的 80 端口访问到容器实例，这种模式当前只支持 Linux，不支持 MAC、windows 系统，容器实例中运行`ip a`如下：

![img](http://yangjunsss.github.io/images/docker_host.png)

不仅仅 netns 可以共享，同一个 namespace 可以被多个容器实例共享。

## Bridge 模式
Bridge 模式为在 Host 机器上为每一个容器或者多个容器创建 Network Namespace 进行网络隔离，并创建一对 veth，一端连接着 netns，一端连接着 Host 上的 bridge 设备，bridge 作为二层交换设备进行数据转发，可以用软件或硬件实现，Docker 使用 linux bridge 软件实现方式，并且 docker 使 FORWARD chain 默认策略为 DROP，不允许 bridge 容器实例与其他链路连通。在虚拟化技术中这种方式使最普遍最经典的方式，以下我们通过 netns、bridge 模拟实现。

模拟组网：

![img](http://yangjunsss.github.io/images/docker_bridge.png)

组网思路：
1. 创建2个 bridge，2 台 Host，3 个 netns，3对 veth
2. 分配 IP 地址，bridge 网关地址
3. 检查 iptables 的配置，允许 FORWARD 为 ACCEPT，开启 ipv4 forward 转发标识位
4. 给 Host 配置路由地址

网络接口配置如下：

```sh
[root@i-7dlclo08 ~]# ip -d a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00 promiscuity 0
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether 52:54:ca:9d:db:ff brd ff:ff:ff:ff:ff:ff promiscuity 0
    inet 192.168.100.2/24 brd 192.168.100.255 scope global dynamic eth0
       valid_lft 36106sec preferred_lft 36106sec
    inet6 fe80::76ef:824d:95ef:18a3/64 scope link
       valid_lft forever preferred_lft forever
8: br0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP qlen 1000
    link/ether 3a:a6:23:20:30:e0 brd ff:ff:ff:ff:ff:ff promiscuity 0
    bridge forward_delay 1500 hello_time 200 max_age 2000
    inet 10.20.1.1/24 scope global br0
       valid_lft forever preferred_lft forever
    inet6 fe80::cc1f:b2ff:fee9:4ffd/64 scope link
       valid_lft forever preferred_lft forever
14: veth0@if13: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master br0 state UP qlen 1000
    link/ether 3a:a6:23:20:30:e0 brd ff:ff:ff:ff:ff:ff link-netnsid 0 promiscuity 1
    veth
    bridge_slave
    inet6 fe80::38a6:23ff:fe20:30e0/64 scope link
       valid_lft forever preferred_lft forever
16: veth1@if15: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master br0 state UP qlen 1000
    link/ether fa:49:d2:11:17:94 brd ff:ff:ff:ff:ff:ff link-netnsid 1 promiscuity 1
    veth
    bridge_slave
    inet6 fe80::f849:d2ff:fe11:1794/64 scope link
       valid_lft forever preferred_lft forever
```

确认和清理 iptables 规则
```sh
#remove/flush all rules & delete chains
iptables -F
iptables -X
iptables -t nat -F
iptables -t nat -X
iptables -t mangle -F
iptables -t mangle -X
iptables -P INPUT ACCEPT
iptables -P OUTPUT ACCEPT
iptables -P FORWARD ACCEPT
```

开启 ipv4 转发
```sh
sysctl -w net.ipv4.ip_forward=1
```

配置对端路由
```sh
# HOST0
ip r add 10.20.2.0/24 via 192.168.100.3 dev eth0

# HOST1
ip r add 10.20.1.0/24 via 192.168.100.2 dev eth0
```

测试同一个 bridge 下二层连通性:

```sh
[root@i-7dlclo08 ~]# ip netns exec ns0 ping 10.20.1.3
PING 10.20.1.3 (10.20.1.3) 56(84) bytes of data.
64 bytes from 10.20.1.3: icmp_seq=1 ttl=64 time=0.114 ms
64 bytes from 10.20.1.3: icmp_seq=2 ttl=64 time=0.098 ms
```

测试不同 bridge 下三层连通性:
```sh
[root@i-7dlclo08 ~]# ip netns exec ns0 ping 10.20.2.2
PING 10.20.2.2 (10.20.2.2) 56(84) bytes of data.
64 bytes from 10.20.2.2: icmp_seq=502 ttl=62 time=0.978 ms
64 bytes from 10.20.2.2: icmp_seq=503 ttl=62 time=0.512 ms
```

## macvlan 模式
在一些特定场景中，比如一些传统应用或者监控应用需要直接使用 HOST 的物理网络，则可以使用 kernel 提供的 macvlan 的方式，macvlan 是在 HOST 网卡上创建多个子网卡，并分配独立的 IP 地址和 MAC 地址，把子网卡分配给容器实例来实现实例与物理网络的直通，并同时保持容器实例的隔离性。Host 收到数据包后，则根据不同的 MAC 地址把数据包从转发给不同的子接口，在外界来看就相当于多台主机。macvlan 要求物理网卡支持混杂 promisc 模式并且要求 kernel 为 v3.9-3.19 和 4.0+，因为是直接通过子接口转发数据包，所以可想而知，性能比 bridge 要高，不需要经过 NAT。

结构如下：

![img](http://yangjunsss.github.io/images/macvlan/docker_macvlan.png)

macvlan 支持四种模式：
1. private：子接口之间不允许通信，子接口能与物理网络通讯，所有数据包都经过父接口 eth0
2. vepa（Virtual Ethernet Port Aggregator)：子接口之间、子接口与物理网络允许通讯，数据包都经过 eth0 进出，要求交换机支持 IEEE 802.1Q。
3. bridge：子接口之间直接通讯，不经过父接口 eth0 ，性能较高，但是父接口 down 之后也同样丧失通讯能力。
4. passthru：Allows a single VM to be connected directly to the physical interface. The advantage of this mode is that VM is then able to change MAC address and other interface parameters.

所以模式都不能与 eth0 通信，并且 macvlan 在公有云上的支持并不友好。

### private mode 模式

```sh
## HOST0
[root@i-7dlclo08 ~]# ip netns exec ns0 ping 192.168.100.3
PING 192.168.100.3 (192.168.100.3) 56(84) bytes of data.
64 bytes from 192.168.100.3: icmp_seq=1 ttl=64 time=1.48 ms
64 bytes from 192.168.100.3: icmp_seq=2 ttl=64 time=0.355 ms
^C
--- 192.168.100.3 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 0.355/0.917/1.480/0.563 ms

[root@i-7dlclo08 ~]# ip netns exec ns0 ping 192.168.100.11
PING 192.168.100.11 (192.168.100.11) 56(84) bytes of data.
^C
--- 192.168.100.11 ping statistics ---
3 packets transmitted, 0 received, 100% packet loss, time 1999ms
[root@i-7dlclo08 ~]# ip netns exec ns1 ping 192.168.100.12
PING 192.168.100.12 (192.168.100.12) 56(84) bytes of data.
64 bytes from 192.168.100.12: icmp_seq=1 ttl=64 time=1.33 ms
```
![img](http://yangjunsss.github.io/images/macvlan/macvlan_private.png)

### vepa 模式

```sh
[root@i-7dlclo08 ~]# sudo ip netns exec ns0 ping 192.168.100.11
PING 192.168.100.11 (192.168.100.11) 56(84) bytes of data.
^C
--- 192.168.100.11 ping statistics ---
2 packets transmitted, 0 received, 100% packet loss, time 999ms

[root@i-7dlclo08 ~]# sudo ip netns exec ns0 ping 192.168.100.12
PING 192.168.100.12 (192.168.100.12) 56(84) bytes of data.
64 bytes from 192.168.100.12: icmp_seq=1 ttl=64 time=1.37 ms
64 bytes from 192.168.100.12: icmp_seq=2 ttl=64 time=0.530 ms
^C
--- 192.168.100.12 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 0.530/0.954/1.379/0.425 ms
```

![img](http://yangjunsss.github.io/images/macvlan/macvlan_vepa.png)

与预期不符合的是不 10 不能 ping 通 11。

### bridge 模式

```sh
## HOST0
[root@i-7dlclo08 ~]# sudo ip netns exec ns0 ping 192.168.100.11
PING 192.168.100.11 (192.168.100.11) 56(84) bytes of data.
64 bytes from 192.168.100.11: icmp_seq=1 ttl=64 time=0.075 ms
^C
--- 192.168.100.11 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.075/0.075/0.075/0.000 ms
[root@i-7dlclo08 ~]# sudo ip netns exec ns0 ping 192.168.100.12
PING 192.168.100.12 (192.168.100.12) 56(84) bytes of data.
64 bytes from 192.168.100.12: icmp_seq=1 ttl=64 time=1.29 ms
64 bytes from 192.168.100.12: icmp_seq=2 ttl=64 time=0.469 ms
^C
--- 192.168.100.12 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 0.469/0.880/1.292/0.412 ms
```

![img](http://yangjunsss.github.io/images/macvlan/macvlan_bridge.png)

### passthru 模式

passthru 的模式在公有云上直接导致虚拟机网络不通，无法验证


## 参考

[http://hicu.be/bridge-vs-macvlan](http://hicu.be/bridge-vs-macvlan)
