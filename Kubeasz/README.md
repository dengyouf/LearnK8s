## 集群规划

> 本此集群搭建使用 Kubeasz 项目进行部署，基于二进制方式部署和利用 ansible-playbook 实现自动化。

| 主机名 | IP 地址 | 角色 |
| --- | --- | --- |
| k8s-master01 | 172.20.47.4 | master & worker |
| k8s-node01 | 172.20.47.203 | master & worker |
| k8s-node02 | 172.20.47.204 | master & worker |


### 准备工作

- hosts 解析

- [时间同步](https://github.com/easzlab/kubeasz/blob/master/docs/guide/chrony.md)

- [升级内核](https://github.com/easzlab/kubeasz/blob/master/docs/guide/kernel_upgrade.md)

- [主机互信](https://github.com/easzlab/kubeasz/blob/master/docs/setup/00-planning_and_overall_intro.md)
  > 在控制节点(k8s-master)上执行：`ssh-keygen -t rsa -b 2048 -N '' -f ~/.ssh/id_rsa`
  > $IPs为所有节点地址包括控制节点：`ssh-copy-id $IPs #`
  
- 在控制节点(k8s-master)上安装 ansible
    ```bash
    curl -O https://bootstrap.pypa.io/pip/2.7/get-pip.py`
     python get-pip.py`
    python -m pip install --upgrade "pip < 21.0"`
     pip install ansible -i https://mirrors.aliyun.com/pypi/simple/`
    ```
  
- 下载项目源码、二进制及离线镜像
    ```bash
    # 下载工具脚本ezdown，举例使用kubeasz版本3.0.0
    export release=3.0.0
    wget https://github.com/easzlab/kubeasz/releases/download/${release}/ezdown
    chmod +x ./ezdown
    # 使用工具脚本下载
    ./ezdown -D
    ```



### 安装集群

- 创建集群配置 ：`cd /etc/kubeasz/ && ./ezctl new k8s-01`
    - 规划主机：`/etc/kubeasz/clusters/k8s-01/hosts`
    - 规划集群配置：`/etc/kubeasz/clusters/k8s-01/config.yml`
  
- [创建证书和环境准备](https://github.com/easzlab/kubeasz/blob/master/docs/setup/01-CA_and_prerequisite.md) `./ezctl setup k8s-01 01 |tee install-log/01.log`

- [安装etcd集群](https://github.com/easzlab/kubeasz/blob/master/docs/setup/02-install_etcd.md): `./ezctl setup k8s-01 02 |tee install-log/02.log`
  ```bash
  export NODE_IPS="172.20.47.4 172.20.47.203 172.20.47.204"
  for ip in ${NODE_IPS}; do
  ETCDCTL_API=3 /opt/kube/bin/etcdctl  \
  --endpoints=https://${ip}:2379  \
  --cacert=/etc/kubernetes/ssl/ca.pem \
  --cert=/etc/kubernetes/ssl/etcd.pem \
  --key=/etc/kubernetes/ssl/etcd-key.pem \
  endpoint health; done
  https://172.20.47.4:2379 is healthy: successfully committed proposal: took = 7.584065ms
  https://172.20.47.203:2379 is healthy: successfully committed proposal: took = 7.341181ms
  https://172.20.47.204:2379 is healthy: successfully committed proposal: took = 7.616303ms
  ```

- [安装容器运行时（docker or containerd）](https://github.com/easzlab/kubeasz/blob/master/docs/setup/03-container_runtime.md): `./ezctl setup k8s-01 03 |tee install-log/03.log`

  ```bash
  docker version
  Client: Docker Engine - Community
  Version:           19.03.14
  API version:       1.40
  Go version:        go1.13.15
  Git commit:        5eb3275
  Built:             Tue Dec  1 19:14:24 2020
  OS/Arch:           linux/amd64
  Experimental:      false
  
  Server: Docker Engine - Community
  Engine:
  Version:          19.03.14
  API version:      1.40 (minimum version 1.12)
  Go version:       go1.13.15
  Git commit:       5eb3275
  Built:            Tue Dec  1 19:21:08 2020
  OS/Arch:          linux/amd64
  Experimental:     false
  containerd:
  Version:          v1.3.9
  GitCommit:        ea765aba0d05254012b0b9e595e995c09186427f
  runc:
  Version:          1.0.0-rc10
  GitCommit:        dc9208a3303feef5b3839f4323d9beb36df0a9dd
  docker-init:
  Version:          0.18.0
  GitCommit:        fec3683
  ```

- [安装kube_master节点](https://github.com/easzlab/kubeasz/blob/master/docs/setup/04-install_kube_master.md) ：`./ezctl setup k8s-01 04 |tee install-log/04.log`

  ```bash
  systemctl status kube-apiserver kube-controller-manager kube-scheduler
  kubectl get componentstatus 
  Warning: v1 ComponentStatus is deprecated in v1.19+
  NAME                 STATUS    MESSAGE             ERROR
  controller-manager   Healthy   ok                  
  scheduler            Healthy   ok                  
  etcd-2               Healthy   {"health":"true"}   
  etcd-1               Healthy   {"health":"true"}   
  etcd-0               Healthy   {"health":"true"} 
  ```

- [安装kube_node节点](https://github.com/easzlab/kubeasz/blob/master/docs/setup/05-install_kube_node.md) : `./ezctl setup k8s-01 05 |tee install-log/05.log`

  ```bash
  systemctl status kubelet kube-proxy
  kubectl  get nodes
  NAME            STATUS   ROLES    AGE     VERSION
  172.20.47.203   Ready    master   3m16s   v1.20.2
  172.20.47.204   Ready    master   3m16s   v1.20.2
  172.20.47.4     Ready    master   3m16s   v1.20.2
  ```

- [安装网络组件](https://github.com/easzlab/kubeasz/blob/master/docs/setup/06-install_network_plugin.md): `./ezctl setup k8s-01 06 |tee install-log/06.log`

  ```bash
   kubectl get pod --all-namespaces
  NAMESPACE     NAME                                      READY   STATUS    RESTARTS   AGE
  kube-system   calico-kube-controllers-5677ffd49-mjfgj   1/1     Running   0          47s
  kube-system   calico-node-2qzfm                         1/1     Running   0          48s
  kube-system   calico-node-8fmn7                         1/1     Running   0          48s
  kube-system   calico-node-cfnf8                         1/1     Running   0          48s
  
  kubectl  create deployment myapp --image=ikubernetes/myapp:v1 --replicas=3
   kubectl  get pod -o wide -w
  NAME                    READY   STATUS              RESTARTS   AGE   IP               NODE            NOMINATED NODE   READINESS GATES
  myapp-7d4b7b84b-6g8xm   1/1     Running             0          13s   192.168.85.196   172.20.47.203   <none>           <none>
  myapp-7d4b7b84b-pjg7p   1/1     Running             0          13s   192.168.32.131   172.20.47.4     <none>           <none>
  myapp-7d4b7b84b-t86sm   1/1     Running             0          60s   192.168.58.193   172.20.47.204   <none>           <none>
  ```

- [安装集群主要插](https://github.com/easzlab/kubeasz/blob/master/docs/setup/07-install_cluster_addon.md) ：`./ezctl setup k8s-01 07 |tee install-log/07.log`
  - coredns
  - metrics-server
  - dashboard
  
  ```bash
   kubectl  expose deployment/myapp  --port=80 --target-port=80
   kubectl  exec -it myapp-7d4b7b84b-6g8xm -- sh
   / # wget -q myapp
   / # wget -q -O  - myapp
   Hello MyApp | Version: v1 | <a href="hostname.html">Pod Name</a>
  ```