## Set up a NFS Server

```bash
yum -y install nfs-utils 
mkdir -pv /data/k8s/
echo "/data/k8s/  172.20.0.0/16(rw,no_root_squash)" >> /etc/exports
echo "/data/nfs-k8sdata/  172.20.0.0/16(rw,no_root_squash)" >> /etc/exports
systemctl  enable nfs --now
```

```bash 
diff values.yaml values.yaml.bak_orig 
5c5
<   repository: ccr.ccs.tencentyun.com/dengyouf/nfs-subdir-external-provisioner
---
>   repository: k8s.gcr.io/sig-storage/nfs-subdir-external-provisioner
11,12c11,12
<   server: 172.20.59.116
<   path: /data/nfs-k8sdata/
---
>   server:
>   path: /nfs-storage

helm install  nfs-subdir-external-provisioner .
```
## 
[项目地址](`https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner`)

