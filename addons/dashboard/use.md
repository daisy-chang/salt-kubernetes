### 1 创建dashborad,当前目录下四个yaml文件
```
kubectl create -f .
```
### 2 查看pod文件，-n命名空间
```
kubectl logs pod/kubernetes-dashboard-66c9d98865-tmtsf -n kube-system
```
### 3 部署kube-proxy的节点+端口就可以登录dashboard
###  https：192.168.57.12:30602
