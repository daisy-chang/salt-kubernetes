### 1 创建deployment
```
kubectl create -f nginx-deployment.yaml
```
### 2 查看deployment
```
kubectl get deployment
```
### 3 查看deployment详细信息
```
kubectl describe deployment nginx-deployment
```
### 4 查看Pod
```
kubectl get pod -o wide
```
### 5 测试pod访问
```
curl --head http://10.2.97.3
```
### 6 更新deployment
```
kubectl set image deployment/nginx-deployment nginx=nginx:1.12.2 --record
kubectl get deployment -o wide
```
### 7 查看更新历史
```
kubectl rollout history deployment/nginx-deployment
```
### 8 查看具体某一个版本的升级历史
```
kubectl rollout history deployment/nginx-deployment --revision=1
```
### 9 快速回滚到上一个版本
```
kubectl rollout undo deployment/nginx-deployment
```
### 10 扩容到5个节点
```
kubectl scale deployment nginx-deployment --replicas 5
```
### 11 创建service
```
kubectl create -f nginx-service.yaml
```
### 12 node节点访问service
```
[root@linux-node2 ~]# curl --head http://10.1.1.73
HTTP/1.1 200 OK
Server: nginx/1.10.3
Date: Tue, 05 Jun 2018 05:43:52 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 31 Jan 2017 15:01:11 GMT
Connection: keep-alive
ETag: "5890a6b7-264"
Accept-Ranges: bytes

[root@linux-node2 ~]# ipvsadm -Ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  10.1.0.1:443 rr persistent 10800
  -> 192.168.56.11:6443           Masq    1      0          0         
TCP  10.1.1.73:80 rr
  -> 10.2.71.3:80                 Masq    1      0          1         
  -> 10.2.71.5:80                 Masq    1      0          0         
  -> 10.2.71.6:80                 Masq    1      0          0         
  -> 10.2.97.3:80                 Masq    1      0          0         
  -> 10.2.97.4:80                 Masq    1      0          0    
```
### 13 做镜像，编写deployment，service，镜像做好，直接更新
