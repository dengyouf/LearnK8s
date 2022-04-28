## Kubeasz 项目介绍

Kubeasz 提供快速部署高可用k8s集群的工具,其基于二进制方式部署和利用ansible-playbook实现自动化。

- k8s: v1.22.2
- etcd: v3.5.0
- docker: 20.10.8
- calico: v3.19.2
- coredns: 1.8.0
- pause: 3.5
- dashboard: v2.3.1
- metrics-server: v0.5.0

### 1. 集群规划

| 主机名字 | IP | 角色 |
| --- | --- | --- |
| k8s-master01 | 192.168.23.11 | master、etcd |
| k8s-master02 | 192.168.23.12 | master、etcd |
| k8s-master03 | 192.168.23.13 | master、etcd | 
| k8s-node01 | 192.168.23.21 | worker |
| k8s-node01 | 192.168.23.21 | worker |
| k8s-node01 | 192.168.23.21 | worker |
| ha-lb1 | 192.168.23.10 | haproxy、keepalived |
| ha-lb2 | 192.168.23.20 | haproxy、keepalived |

### 2. 环境准备

**2.1 时间同步**
```bash 
(echo "*/3 * * * * /usr/sbin/ntpdate ntp.aliyun.com &> /dev/null 2>&1";crontab -l)|crontab 
```

**2.2 升级内核**
```bash
# 载入公钥
rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
# 安装ELRepo
rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-3.el7.elrepo.noarch.rpm
# 载入elrepo-kernel元数据
yum --disablerepo=\* --enablerepo=elrepo-kernel repolist
# 查看可用的rpm包
yum --disablerepo=\* --enablerepo=elrepo-kernel list kernel*
# 安装长期支持版本的kernel
yum --disablerepo=\* --enablerepo=elrepo-kernel install -y kernel-lt.x86_64
# 删除旧版本工具包
yum remove kernel-tools-libs.x86_64 kernel-tools.x86_64 -y
# 安装新版本工具包
yum --disablerepo=\* --enablerepo=elrepo-kernel install -y kernel-lt-tools.x86_64

#查看默认启动顺序
awk -F\' '$1=="menuentry " {print $2}' /etc/grub2.cfg  
CentOS Linux (4.4.183-1.el7.elrepo.x86_64) 7 (Core)  
CentOS Linux (3.10.0-327.10.1.el7.x86_64) 7 (Core)  
CentOS Linux (0-rescue-c52097a1078c403da03b8eddeac5080b) 7 (Core)
#默认启动的顺序是从0开始，新内核是从头插入（目前位置在0，而4.4.4的是在1），所以需要选择0。
grub2-set-default 0
#重启并检查
reboot
```

**2.3 部署节点安装ansible及准备ssh免密登陆**
- 安装 ansible 
```bash
# 注意pip 21.0以后不再支持python2和python3.5，需要如下安装
# To install pip for Python 2.7 install it from https://bootstrap.pypa.io/2.7/ :
curl -O https://bootstrap.pypa.io/pip/2.7/get-pip.py
python get-pip.py
python -m pip install --upgrade "pip < 21.0"
 
# pip安装ansible(国内如果安装太慢可以直接用pip阿里云加速)
pip install ansible -i https://mirrors.aliyun.com/pypi/simple/
```
- 配置免密登陆
```bash
ssh-keygen -t rsa -b 2048 -N '' -f ~/.ssh/id_rsa
cat ssh-copy.sh 
#!/bin/bash
#
IP="192.168.23.10
192.168.23.20
192.168.23.11
192.168.23.12
192.168.23.13
192.168.23.21
192.168.23.22
192.168.23.23"

for node in ${IP};do
sshpass -p root ssh-copy-id  ${node}  -o StrictHostKeyChecking=no
if [ $? -eq 0 ];then
echo -e "\033[46;31m ${node} 秘钥copy成功 \033[0m"
else
echo -e "\033[43;31m ${node} 秘钥copy失败 \033[0m"
fi
done
```
  
### 3. 配置高可用负载均衡

**3.1 安装软件包**

```bash 
yum install -y haproxy keepalived psmisc
```

**3.2 配置 keepalived**
```bash 
echo "net.ipv4.ip_nonlocal_bind=1" | sudo tee /etc/sysctl.d/ip_nonlocal_bind.conf
sysctl --system
cat /etc/keepalived/keepalived.conf
global_defs {
notification_email {
root@localhost
}
notification_email_from root@localhost
smtp_server localhost
smtp_connect_timeout 30
}

# Script used to check if HAProxy is running
vrrp_script check_haproxy {
script "killall -0 haproxy" # check the haproxy process
interval 2 # every 2 seconds
weight 2 # add 2 points if OK
}

vrrp_instance VI_1 {
state BACKUP # MASTER on ha-lb1, BACKUP on ha-lb2
interface eth0
virtual_router_id 255
priority 100 # 101 on ha-lb1, 100 on ha-lb2
advert_int 1
authentication {
auth_type PASS
auth_pass EhazK1Y2MBK37gZktTl1zrUUuBk
}
virtual_ipaddress {
192.168.23.100/24
}

    track_script {
        check_haproxy
    }
}
```
```bash
systemctl enable --now keepalived
```
  
**3.3 配置 haproxy**
```bash
yum  install -y policycoreutils-python
semanage port -a -t http_cache_port_t 6443 -p tcp
```
```bash
cat /etc/haproxy/haproxy.cfg
global
log /dev/log  local0
log /dev/log  local1 notice
stats socket /var/lib/haproxy/stats level admin
chroot /var/lib/haproxy
user haproxy
group haproxy
daemon

defaults
log global
mode  http
option  httplog
option  dontlognull
timeout connect 5000
timeout client 50000
timeout server 50000

frontend kubernetes
bind 192.168.23.100:6443
option tcplog
mode tcp
default_backend kubernetes-master-nodes

backend kubernetes-master-nodes
mode tcp
balance roundrobin
option tcp-check
server k8s-master01 192.168.23.11:6443 check fall 3 rise 2
server k8s-master02 192.168.23.12:6443 check fall 3 rise 2
server k8s-master03 192.168.23.13:6443 check fall 3 rise 2

listen stats 192.168.23.100:8080
mode http
stats enable
stats uri /
stats realm HAProxy\ Statistics
stats auth admin:haproxy
```
```bash
systemctl enable --now haproxy
```

### 4. 在部署节点规划 k8s 安装
**4.1 下载项目源码、二进制离线镜像至 /etc/kubeasz 目录**
```bash 
export release=3.1.1
wget https://github.com/easzlab/kubeasz/releases/download/${release}/ezdown
chmod +x ./ezdown
./ezdown -D
```
  
**4.2 创建集群配置实例**
```bash 
cd /etc/kubeasz/ 
./ezctl new k8s-01
```
```bash
diff clusters/k8s-01/hosts.bak_orig  clusters/k8s-01/hosts
3,5c3,5
< 192.168.1.1
< 192.168.1.2
< 192.168.1.3
---
> 192.168.23.11
> 192.168.23.12
> 192.168.23.13
9,10c9,10
< 192.168.1.1
< 192.168.1.2
---
> 192.168.23.11
> 192.168.23.12
14,15c14,15
< 192.168.1.3
< 192.168.1.4
---
> 192.168.23.21
> 192.168.23.22
20c20
< #192.168.1.8 NEW_INSTALL=false
---
> #192.168.23.8 NEW_INSTALL=false
24,25c24,25
< #192.168.1.6 LB_ROLE=backup EX_APISERVER_VIP=192.168.1.250 EX_APISERVER_PORT=8443
< #192.168.1.7 LB_ROLE=master EX_APISERVER_VIP=192.168.1.250 EX_APISERVER_PORT=8443
---
> #192.168.23.20 LB_ROLE=backup EX_APISERVER_VIP=192.168.23.100 EX_APISERVER_PORT=6443
> #192.168.23.10 LB_ROLE=master EX_APISERVER_VIP=192.168.23.100 EX_APISERVER_PORT=6443
29c29
< #192.168.1.1
---
> #192.168.23.1
40c40
< CLUSTER_NETWORK="flannel"
---
> CLUSTER_NETWORK="calico"
52c52
< NODE_PORT_RANGE="30000-32767"
---
> NODE_PORT_RANGE="30000-52767"
59c59
< bin_dir="/opt/kube/bin"
---
> bin_dir="/usr/local/bin"
```
```bash 
# 规划集群组件
cat /etc/kubeasz/clusters/k8s-01/config.yml 
 diff clusters/k8s-01/config.yml.bak_orig  clusters/k8s-01/config.yml
46c46
< ENABLE_MIRROR_REGISTRY: false
---
> ENABLE_MIRROR_REGISTRY: true
70c70,76
<   - "10.1.1.1"
---
>   - "192.168.23.11"
>   - "192.168.23.12"
>   - "192.168.23.13"
>   - "192.168.23.14"
>   - "192.168.23.15"
>   - "192.168.23.16"
>   - "192.168.23.17"
87c93
< MAX_PODS: 110
---
> MAX_PODS: 200
180c186
< LOCAL_DNS_CACHE: "169.254.20.10"
---
> LOCAL_DNS_CACHE: "10.68.0.2"
```

### 5.分步骤安装集群

- 01-创建证书和环境准备: `./ezctl setup k8s-01 01 |tee 01.log`

- 02-安装etcd集群: `./ezctl setup k8s-01 02 |tee 02.log`
```bash
export NODE_IPS="192.168.23.11 192.168.23.12 192.168.23.13"
for ip in ${NODE_IPS}; do
ETCDCTL_API=3 etcdctl \
--endpoints=https://${ip}:2379  \
--cacert=/etc/kubernetes/ssl/ca.pem \
--cert=/etc/kubernetes/ssl/etcd.pem \
--key=/etc/kubernetes/ssl/etcd-key.pem \
endpoint health; done

https://192.168.23.11:2379 is healthy: successfully committed proposal: took = 11.16228ms
https://192.168.23.12:2379 is healthy: successfully committed proposal: took = 12.845246ms
https://192.168.23.13:2379 is healthy: successfully committed proposal: took = 12.897067ms
```  

- 03-安装容器运行时: `./ezctl setup k8s-01 03 |tee 03.log`
```bash 
systemctl status docker 	# 服务状态
journalctl -u docker 		# 运行日志
docker version
docker info
```

- 04-安装kube_master节点: `./ezctl setup k8s-01 04 |tee 04.log`
```bash
# 查看进程状态
systemctl status kube-apiserver
systemctl status kube-controller-manager
systemctl status kube-scheduler
# 查看进程运行日志
journalctl -u kube-apiserver
journalctl -u kube-controller-manager
journalctl -u kube-scheduler
```
```bash 
kubectl  get cs
Warning: v1 ComponentStatus is deprecated in v1.19+
NAME                 STATUS    MESSAGE                         ERROR
scheduler            Healthy   ok                              
controller-manager   Healthy   ok                              
etcd-1               Healthy   {"health":"true","reason":""}   
etcd-2               Healthy   {"health":"true","reason":""}   
etcd-0               Healthy   {"health":"true","reason":""}
```
> [kubeasz 3.1.1 版本问题反馈](https://github.com/easzlab/kubeasz/issues/1084) ：
controller-manager Unhealthy Get "https://127.0.0.1:10257/healthz": dial tcp 127.0.0.1:10257: connect: connection refused

- 05-安装kube_node节点: `./ezctl setup k8s-01 05 |tee 05.log`

```bash
systemctl status kubelet	# 查看状态
systemctl status kube-proxy
journalctl -u kubelet		# 查看日志
journalctl -u kube-proxy 
```
```bash
 kubectl get node 
NAME            STATUS                     ROLES    AGE   VERSION
192.168.23.11   Ready,SchedulingDisabled   master   24m   v1.22.2
192.168.23.12   Ready,SchedulingDisabled   master   24m   v1.22.2
192.168.23.21   Ready                      node     20m   v1.22.2
192.168.23.22   Ready                      node     20m   v1.22.2
```
  
- 06-安装网络组件: `./ezctl setup k8s-01 06 |tee 06.log`
```bash 
kubectl get pod --all-namespaces
NAMESPACE     NAME                                       READY   STATUS    RESTARTS   AGE
kube-system   calico-kube-controllers-59df8b6856-nsmjx   1/1     Running   0          41s
kube-system   calico-node-4vrmm                          1/1     Running   0          41s
kube-system   calico-node-72nj7                          1/1     Running   0          41s
kube-system   calico-node-99xjd                          1/1     Running   0          41s
kube-system   calico-node-fkxzh                          1/1     Running   0          41s
```
```bash
calicoctl node status
Calico process is running.

IPv4 BGP status
+---------------+-------------------+-------+----------+-------------+
| PEER ADDRESS  |     PEER TYPE     | STATE |  SINCE   |    INFO     |
+---------------+-------------------+-------+----------+-------------+
| 192.168.23.11 | node-to-node mesh | up    | 11:04:41 | Established |
| 192.168.23.21 | node-to-node mesh | up    | 11:04:39 | Established |
| 192.168.23.22 | node-to-node mesh | up    | 11:04:39 | Established |
```
```
kubectl  create deployment myapp --image=ikubernetes/myapp:v1 --replicas=4
kubectl  expose deployment myapp --port=80 --target-port=80
```
```bash
NAME                    READY   STATUS    RESTARTS   AGE    IP              NODE            NOMINATED NODE   READINESS GATES
myapp-7d4b7b84b-djtzg   1/1     Running   0          100s   172.20.85.194   192.168.23.21   <none>           <none>
myapp-7d4b7b84b-hgkqg   1/1     Running   0          100s   172.20.58.193   192.168.23.22   <none>           <none>
myapp-7d4b7b84b-qxfr6   1/1     Running   0          100s   172.20.85.193   192.168.23.21   <none>           <none>
myapp-7d4b7b84b-whckp   1/1     Running   0          100s   172.20.58.194   192.168.23.22   <none>           <none>
kubectl  get svc 
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.68.0.1       <none>        443/TCP   20m
myapp        ClusterIP   10.68.247.212   <none>        80/TCP    27s
```
```bash
curl 172.20.85.194/hostname.html
myapp-7d4b7b84b-djtzg
curl 172.20.85.193/hostname.html
myapp-7d4b7b84b-qxfr6
curl 10.68.247.212/hostname.html
myapp-7d4b7b84b-djtzg
curl 10.68.247.212/hostname.html
myapp-7d4b7b84b-qxfr6
```

### 6. 部署 Kubernetes 插件

**6.1 部署 coredns **

```bash
kubectl  exec -it myapp-7d4b7b84b-djtzg -- cat /etc/resolv.conf 
nameserver 10.68.0.2
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```
```bash
wget https://raw.githubusercontent.com/kubernetes/kubernetes/release-1.22/cluster/addons/dns/coredns/coredns.yaml.base
cp coredns.yaml.base coredns.yaml.base.bak_orig 
diff coredns.yaml.base.bak_orig coredns.yaml.base
70c70
<         kubernetes __DNS__DOMAIN__ in-addr.arpa ip6.arpa {
---
>         kubernetes cluster.local in-addr.arpa ip6.arpa {
135c135
<         image: k8s.gcr.io/coredns/coredns:v1.8.0
---
>         image: dengyouf/coredns:v1.8.0
139c139
<             memory: __DNS__MEMORY__LIMIT__
---
>             memory: 256Mi
205c205
<   clusterIP: __DNS__SERVER__
---
>   clusterIP: 10.68.0.2
~]# kubectl  apply -f  coredns.yaml.base
```
```bash
kubectl  get svc
NAME         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.68.0.1      <none>        443/TCP   43m
myapp        ClusterIP   10.68.210.45   <none>        80/TCP    37m

kubectl  exec myapp-7d4b7b84b-4v4sd -- nslookup myapp
nslookup: can't resolve '(null)': Name does not resolve

Name:      myapp
Address 1: 10.68.210.45 myapp.default.svc.cluster.local
[root@k8s-master01 kubeasz]# kubectl  exec myapp-7d4b7b84b-4v4sd -- nslookup kubernetes
nslookup: can't resolve '(null)': Name does not resolve

Name:      kubernetes
Address 1: 10.68.0.1 kubernetes.default.svc.cluster.local
```

**6.2 部署 metrics-server**
```bash
wget https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.5.0/components.yaml
diff components.yaml.bak_orig  components.yaml
137c137
<         image: k8s.gcr.io/metrics-server/metrics-server:v0.5.0
---
>         image: dengyouf/metrics-server:v0.5.0

kubectl  apply -f components.yaml
```
```bahs
kubectl  top nodes
NAME            CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%   
192.168.23.11   188m         9%     1165Mi          70%       
192.168.23.12   206m         10%    1267Mi          76%       
192.168.23.21   105m         5%     1348Mi          24%       
192.168.23.22   92m          4%     1292Mi          22% 
```

**6.3 部署 dashboard**
**6.3 部署 dashboard**
```bash
wget https://raw.githubusercontent.com/kubernetes/dashboard/v2.3.1/aio/deploy/recommended.yaml
kubectl  apply -f recommended.yaml 
kubectl apply -f admin-user.yaml
kubectl -n kubernetes-dashboard get secret $(kubectl -n kubernetes-dashboard get sa/admin-user -o jsonpath="{.secrets[0].name}") -o go-template="{{.data.token | base64decode}}"
```

  


