
###  0 文件中如果要修改的话就是clusterIP
```
clusterIP：10.1.0.2 这个是虚拟IP，需要根据实际情况修改
```
###  1 创建
```
kubectl create -f coredns.yaml
```
###  2 查看deployment，单独放在kube-system默认的命名空间中
```
kubectl get deployment -n kube-system
```
###  3 查看service
```
kubectl get service -n kube-system
```
###  4 查看pod
```
kubectl get pod -n kube-system
```
###  5 测试dns，进入容器中ping baidu.com是否成功
```
kubectl run dns-test --rm -it --image=alpine /bin/sh
```
