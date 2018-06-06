### 0 实验环境
```
主机名：hosts1/2/3
IP：192.168.56.11/12/13
配置：1C2G50G
系统：Centos-7.x minimal Install
网络：NAT网络
```
## 安装注意：
### 1 安装时在Install Centos7界面，光标选择“Install Centos7”，并且单机Tab，打开kernel启动选项后，增加net.ifnames=0 biosdevname=0
  指定网卡的命名方式为eth0
## 2 配置网络
```
vim /etc/sysconfig/network-scripts/ifcfg-eth0 
TYPE="Ethernet"
BOOTPROTO="static"
NAME="eth0"
DEVICE="eth0"
ONBOOT="yes"
IPADDR=192.168.56.11
NETMASK=255.255.255.0
GATEWAY=192.168.56.2
```
### 3 关闭Network Manager和防火墙开机自启动
```
  systemctl disable firewalld
  systemctl disable NetworkManager
```
### 4 设置主机名
```
 vi /etc/hostname
 linux-node1.example.com
```
### 5 设置主机名解析
```
[root@linux-node2 ~]# cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.56.11 linux-node1 linux-node1.example.com
192.168.56.12 linux-node2 linux-node2.example.com
192.168.56.13 linux-node3 linux-node3.example.com
```
### 6 设置DNS解析
```
vi /etc/resolv.conf
nameserver 192.168.56.2
```
### 7 安装EPEL仓库和常用命令
```
rpm -ivh http://mirrors.aliyun.com/epel/epel-release-latest-7.noarch.rpm
yum -y install net-tools vim lrzsz tree screen lsof tcpdump nc mtr nmap
```
### 8 关闭并确认Selinux处于关闭状态
```
vi /etc/sysconfig/selinux
SELINUX=disabled
```
### 9 重启
### 10 克隆虚拟机以及做快照
### 11 配置免密钥登录
设置部署节点到其它所有节点的SSH免密码登录（包括本机）
[root@linux-node1 ~]# ssh-keygen -t rsa
[root@linux-node1 ~]# ssh-copy-id linux-node1
[root@linux-node1 ~]# ssh-copy-id linux-node2
[root@linux-node1 ~]# ssh-copy-id linux-node3
### 12 安装Salt-SSH
安装Salt SSH（注意：老版本的Salt SSH不支持Roster定义Grains，需要2017.7.4以上版本）

[root@linux-node1 ~]# yum install https://repo.saltstack.com/yum/redhat/salt-repo-latest-2.el7.noarch.rpm 
[root@linux-node1 ~]# yum install -y salt-ssh git
