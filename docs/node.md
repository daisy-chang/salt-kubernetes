## 部署kubelet

1.二进制包准备
将软件包从linux-node1复制到linux-node2中去。
```
[root@linux-node1 ~]# cd /usr/local/src/kubernetes/server/bin/
[root@linux-node1 bin]# cp kubelet kube-proxy /opt/kubernetes/bin/
[root@linux-node1 bin]# scp kubelet kube-proxy 192.168.56.12:/opt/kubernetes/bin/
[root@linux-node1 bin]# scp kubelet kube-proxy 192.168.56.13:/opt/kubernetes/bin/
```

2.创建角色绑定
```
[root@linux-node1 ~]# kubectl create clusterrolebinding kubelet-bootstrap --clusterrole=system:node-bootstrapper --user=kubelet-bootstrap
clusterrolebinding "kubelet-bootstrap" created
```
```
kubelet启动时需要向API发送tsl bootstrap请求，所以要将bootstrap的token设置成对应我们的角色，这样kubectl才有权限创建请求，kubelet启来的时候会动态的获取api
```
3.创建 kubelet bootstrapping kubeconfig 文件
设置集群参数
```
[root@linux-node1 ~]# kubectl config set-cluster kubernetes \
   --certificate-authority=/opt/kubernetes/ssl/ca.pem \
   --embed-certs=true \
   --server=https://192.168.56.11:6443 \
   --kubeconfig=bootstrap.kubeconfig
Cluster "kubernetes" set.
```

设置客户端认证参数
（注意：这个token每次安装集群都不一样，是你安装apiserver的时候生成的，可以在/opt/kubernetes/ssl/ bootstrap-token.csv查看）
```
[root@linux-node1 ~]# kubectl config set-credentials kubelet-bootstrap \
   --token=3cb4b809d48cd5ebab06156210af11a1 \
   --kubeconfig=bootstrap.kubeconfig   
User "kubelet-bootstrap" set.
```

设置上下文参数
```
[root@linux-node1 ~]# kubectl config set-context default \
   --cluster=kubernetes \
   --user=kubelet-bootstrap \
   --kubeconfig=bootstrap.kubeconfig
Context "default" created.
```

选择默认上下文
```
[root@linux-node1 ~]# kubectl config use-context default --kubeconfig=bootstrap.kubeconfig
Switched to context "default".
[root@linux-node1 kubernetes]# cp bootstrap.kubeconfig /opt/kubernetes/cfg
[root@linux-node1 kubernetes]# scp bootstrap.kubeconfig 192.168.56.12:/opt/kubernetes/cfg
[root@linux-node1 kubernetes]# scp bootstrap.kubeconfig 192.168.56.13:/opt/kubernetes/cfg
```

部署kubelet(计算节点，cni网络接口插件)
1.设置CNI支持
```
[root@linux-node2 ~]# mkdir -p /etc/cni/net.d
[root@linux-node2 ~]# vim /etc/cni/net.d/10-default.conf
{
        "name": "flannel",
        "type": "flannel",
        "delegate": {
            "bridge": "docker0",
            "isDefaultGateway": true,
            "mtu": 1400
        }
}


```

2.创建kubelet目录（未来的kubelet数据放在 /var/lib/kubelet）
```
[root@linux-node2 ~]# mkdir /var/lib/kubelet
```

3.创建kubelet服务配置
```
[root@k8s-node2 ~]# vim /usr/lib/systemd/system/kubelet.service
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=docker.service
Requires=docker.service

[Service]
WorkingDirectory=/var/lib/kubelet
ExecStart=/opt/kubernetes/bin/kubelet \
  --address=192.168.56.12 \
  --hostname-override=192.168.56.12 \
  --pod-infra-container-image=mirrorgooglecontainers/pause-amd64:3.0 \
  --experimental-bootstrap-kubeconfig=/opt/kubernetes/cfg/bootstrap.kubeconfig \
  --kubeconfig=/opt/kubernetes/cfg/kubelet.kubeconfig \
  --cert-dir=/opt/kubernetes/ssl \
  --network-plugin=cni \
  --cni-conf-dir=/etc/cni/net.d \
  --cni-bin-dir=/opt/kubernetes/bin/cni \
  --cluster-dns=10.1.0.2 \
  --cluster-domain=cluster.local. \
  --hairpin-mode hairpin-veth \
  --allow-privileged=true \
  --fail-swap-on=false \
  --logtostderr=true \
  --v=2 \
  --logtostderr=false \
  --log-dir=/opt/kubernetes/log
Restart=on-failure
RestartSec=5
```

4.启动Kubelet
```
[root@linux-node2 ~]# systemctl daemon-reload
[root@linux-node2 ~]# systemctl enable kubelet
[root@linux-node2 ~]# systemctl start kubelet
```

5.查看服务状态
```
[root@linux-node2 kubernetes]# systemctl status kubelet
```

6.查看csr请求
注意是在linux-node1上执行。
```
[root@linux-node1 ~]# kubectl get csr
NAME                                                   AGE       REQUESTOR           CONDITION
node-csr-0_w5F1FM_la_SeGiu3Y5xELRpYUjjT2icIFk9gO9KOU   1m        kubelet-bootstrap   Pending
```

7.批准kubelet 的 TLS 证书请求
```
[root@linux-node1 ~]# kubectl get csr|grep 'Pending' | awk 'NR>0{print $1}'| xargs kubectl certificate approve
证书批准后会看到client认证文件
[root@linux-node2 ~]# ll /opt/kubernetes/ssl/
-rw-r--r-- 1 root root 1046 Jun  4 08:43 kubelet-client.crt
-rw------- 1 root root  227 Jun  4 08:37 kubelet-client.key
-rw-r--r-- 1 root root 2185 Jun  4 07:57 kubelet.crt
-rw------- 1 root root 1675 Jun  4 07:57 kubelet.key
```

执行完毕后，查看节点状态已经是Ready的状态了,还未部署kube-proxy
```
[root@linux-node1 ssl]# kubectl get node 
NAME            STATUS    ROLES     AGE       VERSION
192.168.56.12   Ready     <none>    2m        v1.10.1
192.168.56.13   Ready     <none>    2m        v1.10.1
```

## 部署Kubernetes Proxy
1.配置kube-proxy使用LVS
```
[root@linux-node2 ~]# yum install -y ipvsadm ipset conntrack
```

2.创建 kube-proxy 证书请求
```
[root@linux-node1 ~]# cd /usr/local/src/ssl/
[root@linux-node1 ~]# vim kube-proxy-csr.json
{
  "CN": "system:kube-proxy",
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
   
3.生成证书
```
[root@linux-node1~]# cfssl gencert -ca=/opt/kubernetes/ssl/ca.pem \
   -ca-key=/opt/kubernetes/ssl/ca-key.pem \
   -config=/opt/kubernetes/ssl/ca-config.json \
   -profile=kubernetes  kube-proxy-csr.json | cfssljson -bare kube-proxy
```
4.分发证书到所有Node节点
```
[root@linux-node1 ssl]# cp kube-proxy*.pem /opt/kubernetes/ssl/
[root@linux-node1 ssl]# scp kube-proxy*.pem 192.168.56.12:/opt/kubernetes/ssl/
[root@linux-node1 ssl]# scp kube-proxy*.pem 192.168.56.12:/opt/kubernetes/ssl/
```

5.创建kube-proxy配置文件
```
[root@linux-node2 ~]# kubectl config set-cluster kubernetes \
   --certificate-authority=/opt/kubernetes/ssl/ca.pem \
   --embed-certs=true \
   --server=https://192.168.56.11:6443 \
   --kubeconfig=kube-proxy.kubeconfig
Cluster "kubernetes" set.

[root@linux-node2 ~]# kubectl config set-credentials kube-proxy \
   --client-certificate=/opt/kubernetes/ssl/kube-proxy.pem \
   --client-key=/opt/kubernetes/ssl/kube-proxy-key.pem \
   --embed-certs=true \
   --kubeconfig=kube-proxy.kubeconfig
User "kube-proxy" set.

[root@linux-node2 ~]# kubectl config set-context default \
   --cluster=kubernetes \
   --user=kube-proxy \
   --kubeconfig=kube-proxy.kubeconfig
Context "default" created.

[root@linux-node2 ~]# kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
Switched to context "default".
```
6.分发kubeconfig配置文件
```
[root@linux-node1 ssl]# cp kube-proxy.kubeconfig /opt/kubernetes/cfg/
[root@linux-node1 ~]# scp kube-proxy.kubeconfig 192.168.56.12:/opt/kubernetes/cfg/
[root@linux-node1 ~]# scp kube-proxy.kubeconfig 192.168.56.13:/opt/kubernetes/cfg/
```

7.创建kube-proxy服务配置
```
[root@linux-node2 bin]# mkdir /var/lib/kube-proxy

[root@k8s-node2 ~]# vim /usr/lib/systemd/system/kube-proxy.service
[Unit]
Description=Kubernetes Kube-Proxy Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target

[Service]
WorkingDirectory=/var/lib/kube-proxy
ExecStart=/opt/kubernetes/bin/kube-proxy \
  --bind-address=192.168.56.12 \
  --hostname-override=192.168.56.12 \
  --kubeconfig=/opt/kubernetes/cfg/kube-proxy.kubeconfig \
--masquerade-all \
  --feature-gates=SupportIPVSProxyMode=true \
  --proxy-mode=ipvs \
  --ipvs-min-sync-period=5s \
  --ipvs-sync-period=5s \
  --ipvs-scheduler=rr \
  --logtostderr=true \
  --v=2 \
  --logtostderr=false \
  --log-dir=/opt/kubernetes/log

Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
8.启动Kubernetes Proxy
[root@linux-node2 ~]# systemctl daemon-reload
[root@linux-node2 ~]# systemctl enable kube-proxy
[root@linux-node2 ~]# systemctl start kube-proxy
```

9.查看服务状态
查看kube-proxy服务状态
```
[root@linux-node2 scripts]# systemctl status kube-proxy

检查LVS状态
[root@linux-node2 ~]# ipvsadm -L -n
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  10.1.0.1:443 rr persistent 10800
  -> 192.168.56.11:6443           Masq    1      0          0         
```
如果你在两台实验机器都安装了kubelet和proxy服务，使用下面的命令可以检查状态：
```
[root@linux-node1 ssl]#  kubectl get node
NAME            STATUS    ROLES     AGE       VERSION
192.168.56.12   Ready     <none>    22m       v1.10.1
192.168.56.13   Ready     <none>    3m        v1.10.1
```
linux-node3节点请自行部署。
