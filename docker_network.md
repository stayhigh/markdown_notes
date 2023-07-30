--- 
marp: true
--- 
# Docker 网络模型
---
# CNM (Container Network Model)
## CNM 的三个核心组件：[浅谈 Docker 网络：单节点单容器](https://www.cnblogs.com/xingzheanan/p/14233291.html)
- sandbox：沙盒，沙盒包含了容器网络栈的信息，它可以对容器的接口、路由和 DNS 设置等进行管理。一个沙盒可以有多个 endpoint 和 network，它的实现机制一般是 Liunx network space。
- endpoint：端点，一个端点可以加入 sandbox 和 network 中，它可以通过 veth pair、Open vSwitch 或其它网络设备实现，一个端点只可以属于一个网络并且只属于一个沙盒。
- network：网络，是一组可以直接互相连通的端点，它可以通过 Liunx Bridge、VLAN 等实现。
--- 
## CNM 的五种内置驱动：实际上经常使用的还是 bridge driver
- bridge driver：Docker 默认的驱动类型，使用该驱动时 libnetwork 将创建的容器连接到 Docker 网桥上。它可以满足大部分容器网络应用场景，但是在容器访问外网时需要做 NAT， 增加了通信的复杂度。
- host driver：使用该驱动类型时 libnetwork 将不会为容器创建 network namespace，容器和宿主机共用 network namespace。
- overlay driver：该驱动类型使用标准的 VXLAN 方式进行通信，使用该驱动需要额外配置存储服务，如 etcd 等。
- remote driver：该驱动并未做真正的网络实现，它通过调用用户自行实现的网络驱动插件，使 libnetwork 实现驱动的插件化。
- null driver：该驱动会为容器创建 network space，但是不对容器进行任何网络配置，即创建的容器只有 loopback 地址，没有其它网卡、路由和 IP 等信息。
--- 

# 2.单容器网络
- Docker 自带三种网络 bridge, host 和 none，它们对应的 driver 分别是 bridge，host 和 null。
- 以交互模式运行 container，在不指定 network 情况下 Docker 默认 container 使用的网络为 bridge。通过 inspect 查看 bridge 的属性：
- Docker 为 bridge 网络驱动添加了一个网桥 docker0，docker0 上分配的子网是 172.17.0.0/16，并且 ip 172.17.0.1 被作为子网的网关配置在 docker0 上。这里需要注意的是，网桥 docker0 是二层设备，相当于交换机，它的作用是为连在其上的设备转发数据帧，其上配置的 ip 可认为是网桥内部有一个专门用来配置 IP 信息的网卡接口，该接口作为 container 的默认网关地址。同样的，我们看到 container 分到的 ip 地址为 172.17.0.2。
- 执行 brctl show 查看连在 docker0 的接口
--- 

# 3.单容器访问外网
进入 container 中分别 ping 网关，宿主机网口 ip 和外网地址：
--- 
# 4.外网访问单容器
- 容器的 ip 是从 docker subnet 分配的私有地址，对于外网来说是不可见的。
- 为了使外网可以访问容器，需要在容器和宿主机间建立端口映射。

---
```bash
# 容器 demo1 的 80 端口被映射到宿主机的 32768 端口，外网可通过宿主机 ip + 端口的方式访问容器：
$ docker run -d -p 80 --name demo1 httpd
$ ip a

$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWNgroup default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: ens3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 02:42:ac:11:00:44 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.68/16 brd 172.17.255.255 scope global ens3
       valid_lft forever preferred_lft forever
    inet6 fe80::42:acff:fe11:44/64 scope link
       valid_lft forever preferred_lft forever
3: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:42:b6:5f:a7:1b brd ff:ff:ff:ff:ff:ff
    inet 172.18.0.1/24 brd 172.18.0.255 scope global docker0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:b6ff:fe5f:a71b/64 scope link
       valid_lft forever preferred_lft forever
5: veth0541dca@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP group default
    link/ether 2e:31:fe:95:a3:b8 brd ff:ff:ff:ff:ff:ff link-netnsid0
    inet6 fe80::2c31:feff:fe95:a3b8/64 scope link
       valid_lft forever preferred_lft forever

$ curl 172.17.0.68:32768
<html><body><h1>It works!</h1></body></html>

# Docker 在做端口映射时，会启一个 docker-proxy 进程实现宿主机流量到容器 80 端口流量的转发：
$ ps -elf | grep docker-proxy | grep -v grep
root /usr/bin/docker-proxy -proto tcp -host-ip 0.0.0.0 -host-port32768 -container-ip 172.18.0.2 -container-port 80

# 查看路由表：
-A DOCKER ! -i docker0 -p tcp -m tcp --dport 32768 -j DNAT --to-destination 172.18.0.2:80
```