在 NFS 服务器上创建目录

```
mkdir -pv /data/k8s/zookeeper/data-{1,2,3}
```

- 创建 pv： `kubectl apply -f zookeeper-pv.yaml`
- 创建 pvc： `kubectl apply -f zookeeper-pvc.yaml`
- 创建 deploy： `kubectl apply -f zookeeper-deploy.yaml`
