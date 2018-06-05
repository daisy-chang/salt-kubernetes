0 名词释义
```
pod 包含一个或者多个容器，pod有标签,pod之间通过localhost通信，是互通的
rc通过监控运行中的pod来保证集群中运行指定数目的pod副本，少于指定数目，rc就会启动运行新的pod副本，多于指定数目,rc就会杀死多于的pod副本。
rs是新一代的rc，提供同样的高可用能力，支持更多的匹配模式，k8s中1.2中出现的概念，一般和deployment共同使用。
deployment表示用户对k8s集群的一次更新操作。
rc，rs，deployment只是保证了支撑服务的pod数量，但是没有解决如何访问这些服务的问题。一个pod一个运行服务的实例，随时可能在一个节点上停止，在另一个节点以一个新的IP启动一个新的pod，因此不能以确定的IP和端口号提供服务。要稳定的提供服务需要服务发现和负载均衡的能力，服务发现完成的工作，是针对客户端访问的服务，找到对应的后端服务实例。
k8s集群中，客户端需要访问的服务就是service对象，每个service会对应一个集群内部有效的虚拟ip，集群内部通过虚拟ip访问一个服务。
node ip：节点设备ip，如物理机，虚拟机等容器宿主的实际ip
pod ip：pod的IP地址，是根据docker0网格IP段进行分配的
ClusterIP：serviceIP，是一个虚拟IP，仅作用与service对象，由k8s管理和分配，需要结合service port才能使用，单独的IP没有通信功能，集群外访问需要一些修改。"10.1.0.1"。在K8s集群内部，nodeip podip clusterip的通信机制是由k8s制定的路由规则，不是IP路由，访问ClusterIP的VIP就会转到APIserver上。
[root@linux-node2 kubelet]# ipvsadm -Ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  10.1.0.1:443 rr persistent 10800
  -> 192.168.56.11:6443           Masq    1      0          0  
flannel是一个覆盖网络，做网络封装实现多个节点之间容器的通信，k8s来说就是pod之间的通信，flannel要使用etcd，保证给每个node分配出来的ip地址段都不同，保证每个机器的docker0都分配到不同的地址段，帮你做数据包的封装和转发
```
1.为Flannel生成证书
```
[root@linux-node1 ~]# vim flanneld-csr.json
{
  "CN": "flanneld",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
```

2.生成证书
```
[root@linux-node1 ~]# cfssl gencert -ca=/opt/kubernetes/ssl/ca.pem \
   -ca-key=/opt/kubernetes/ssl/ca-key.pem \
   -config=/opt/kubernetes/ssl/ca-config.json \
   -profile=kubernetes flanneld-csr.json | cfssljson -bare flanneld
```
3.分发证书
```
[root@linux-node1 ~]# cp flanneld*.pem /opt/kubernetes/ssl/
[root@linux-node1 ~]# scp flanneld*.pem 192.168.56.12:/opt/kubernetes/ssl/
[root@linux-node1 ~]# scp flanneld*.pem 192.168.56.13:/opt/kubernetes/ssl/
```

4.下载Flannel软件包
```
[root@linux-node1 ~]# cd /usr/local/src
# wget
 https://github.com/coreos/flannel/releases/download/v0.10.0/flannel-v0.10.0-linux-amd64.tar.gz
[root@linux-node1 src]# tar zxf flannel-v0.10.0-linux-amd64.tar.gz
[root@linux-node1 src]# cp flanneld mk-docker-opts.sh /opt/kubernetes/bin/
复制到linux-node2节点
[root@linux-node1 src]# scp flanneld mk-docker-opts.sh 192.168.56.12:/opt/kubernetes/bin/
[root@linux-node1 src]# scp flanneld mk-docker-opts.sh 192.168.56.13:/opt/kubernetes/bin/
复制对应脚本到/opt/kubernetes/bin目录下。
[root@linux-node1 ~]# cd /usr/local/src/kubernetes/cluster/centos/node/bin/
[root@linux-node1 bin]# cp remove-docker0.sh /opt/kubernetes/bin/
[root@linux-node1 bin]# scp remove-docker0.sh 192.168.56.12:/opt/kubernetes/bin/
[root@linux-node1 bin]# scp remove-docker0.sh 192.168.56.13:/opt/kubernetes/bin/
```

5.配置Flannel
```
[root@linux-node1 ~]# vim /opt/kubernetes/cfg/flannel
FLANNEL_ETCD="-etcd-endpoints=https://192.168.56.11:2379,https://192.168.56.12:2379,https://192.168.56.13:2379"
FLANNEL_ETCD_KEY="-etcd-prefix=/kubernetes/network"
FLANNEL_ETCD_CAFILE="--etcd-cafile=/opt/kubernetes/ssl/ca.pem"
FLANNEL_ETCD_CERTFILE="--etcd-certfile=/opt/kubernetes/ssl/flanneld.pem"
FLANNEL_ETCD_KEYFILE="--etcd-keyfile=/opt/kubernetes/ssl/flanneld-key.pem"
复制配置到其它节点上
[root@linux-node1 ~]# scp /opt/kubernetes/cfg/flannel 192.168.56.12:/opt/kubernetes/cfg/
[root@linux-node1 ~]# scp /opt/kubernetes/cfg/flannel 192.168.56.13:/opt/kubernetes/cfg/
```

6.设置Flannel系统服务
```
[root@linux-node1 ~]# vim /usr/lib/systemd/system/flannel.service
[Unit]
Description=Flanneld overlay address etcd agent
After=network.target
Before=docker.service

[Service]
EnvironmentFile=-/opt/kubernetes/cfg/flannel
ExecStartPre=/opt/kubernetes/bin/remove-docker0.sh
ExecStart=/opt/kubernetes/bin/flanneld ${FLANNEL_ETCD} ${FLANNEL_ETCD_KEY} ${FLANNEL_ETCD_CAFILE} ${FLANNEL_ETCD_CERTFILE} ${FLANNEL_ETCD_KEYFILE}
ExecStartPost=/opt/kubernetes/bin/mk-docker-opts.sh -d /run/flannel/docker

Type=notify

[Install]
WantedBy=multi-user.target
RequiredBy=docker.service
复制系统服务脚本到其它节点上
# scp /usr/lib/systemd/system/flannel.service 192.168.56.12:/usr/lib/systemd/system/
# scp /usr/lib/systemd/system/flannel.service 192.168.56.13:/usr/lib/systemd/system/
```

## Flannel CNI集成
下载CNI插件
```
https://github.com/containernetworking/plugins/releases
wget https://github.com/containernetworking/plugins/releases/download/v0.7.1/cni-plugins-amd64-v0.7.1.tgz
[root@linux-node1 ~]# mkdir /opt/kubernetes/bin/cni
[root@linux-node1 src]# tar zxf cni-plugins-amd64-v0.7.1.tgz -C /opt/kubernetes/bin/cni
# scp -r /opt/kubernetes/bin/cni/* 192.168.56.12:/opt/kubernetes/bin/cni/
# scp -r /opt/kubernetes/bin/cni/* 192.168.56.13:/opt/kubernetes/bin/cni/
```
创建Etcd的key
```
/opt/kubernetes/bin/etcdctl --ca-file /opt/kubernetes/ssl/ca.pem --cert-file /opt/kubernetes/ssl/flanneld.pem --key-file /opt/kubernetes/ssl/flanneld-key.pem \
      --no-sync -C https://192.168.56.11:2379,https://192.168.56.12:2379,https://192.168.56.13:2379 \
mk /kubernetes/network/config '{ "Network": "10.2.0.0/16", "Backend": { "Type": "vxlan", "VNI": 1 }}' >/dev/null 2>&1
```
问题解决
```
配置上面的etcd key后，启动flannel时报错：
Jun  6 06:15:37 linux-node1 flanneld: E0606 06:15:37.158899    3068 main.go:349] Couldn't fetch network config: 100: Key not found (/kubernetes) [12]
后在网上查，应该是与etcd key无法通信，所以应该是创建etcd的key时没有成功，再次尝试创建key，如下:
/opt/kubernetes/bin/etcdctl mkdir /kubernetes/network/config
Error:  dial tcp 127.0.0.1:4001: getsockopt: connection refused
但是直接创建又报SSL通信错误，查出应该是我的SSL证书有问题，后换了一个认证解决，如下：
 etcdctl --endpoints=https://192.168.56.11:2379,https://192.168.56.12:2379,https://192.168.56.13:2379 \
  --ca-file=/opt/kubernetes/ssl/ca.pem \
  --cert-file=/opt/kubernetes/ssl/kubernetes.pem \
  --key-file=/opt/kubernetes/ssl/kubernetes-key.pem \
mkdir /kubernetes/network/config
 etcdctl --endpoints=https://192.168.56.11:2379,https://192.168.56.12:2379,https://192.168.56.13:2379 \
  --ca-file=/opt/kubernetes/ssl/ca.pem \
  --cert-file=/opt/kubernetes/ssl/kubernetes.pem \
  --key-file=/opt/kubernetes/ssl/kubernetes-key.pem \
 mk /kubernetes/network/config '{ "Network": "10.2.0.0/16", "Backend": { "Type": "vxlan", "VNI": 1 }}' >/dev/null 2>&1
 还需要注意的是：/opt/kubernetes/cfg/flannel文件中FLANNEL_ETCD_KEY="-etcd-prefix=/kubernetes/network"是否与你生产的key的配置路径一致
 参考链接：https://zhangkesheng.github.io/2018/01/25/kubernetes-ha/
 
```
启动flannel
```
[root@linux-node1 ~]# systemctl daemon-reload
[root@linux-node1 ~]# systemctl enable flannel
[root@linux-node1 ~]# chmod +x /opt/kubernetes/bin/*
[root@linux-node1 ~]# systemctl start flannel
```
查看服务状态（三台机器均需要启动flannel，key只需要在matser上执行）
```
[root@linux-node1 ~]# systemctl status flannel
```

## 配置Docker使用Flannel
```
[root@linux-node1 ~]# vim /usr/lib/systemd/system/docker.service
[Unit] #在Unit下面修改After和增加Requires
After=network-online.target firewalld.service flannel.service
Wants=network-online.target
Requires=flannel.service

[Service] #增加EnvironmentFile=-/run/flannel/docker
Type=notify
EnvironmentFile=-/run/flannel/docker
ExecStart=/usr/bin/dockerd $DOCKER_OPTS
```
将配置复制到另外两个阶段
```
# scp /usr/lib/systemd/system/docker.service 192.168.56.12:/usr/lib/systemd/system/
# scp /usr/lib/systemd/system/docker.service 192.168.56.13:/usr/lib/systemd/system/
```
重启Docker
```
[root@linux-node1 ~]# systemctl daemon-reload
[root@linux-node1 ~]# systemctl restart docker
```
查看docke网络,通过配置docker0和flannel在同一网段
```
[root@linux-node2 ~]# cat /run/flannel/docker
DOCKER_OPT_BIP="--bip=10.2.71.1/24"
DOCKER_OPT_IPMASQ="--ip-masq=true"
DOCKER_OPT_MTU="--mtu=1450"
DOCKER_OPTS=" --bip=10.2.71.1/24 --ip-masq=true --mtu=1450"
[root@linux-node1 ~]# ifconfig
docker0: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
        inet 10.2.102.1  netmask 255.255.255.0  broadcast 10.2.102.255
        ether 02:42:d8:ca:c7:e2  txqueuelen 0  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

flannel.1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1450
        inet 10.2.102.0  netmask 255.255.255.255  broadcast 0.0.0.0
        inet6 fe80::ec4e:19ff:fe1d:2066  prefixlen 64  scopeid 0x20<link>
        ether ee:4e:19:1d:20:66  txqueuelen 0  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 8 overruns 0  carrier 0  collisions 0

```
