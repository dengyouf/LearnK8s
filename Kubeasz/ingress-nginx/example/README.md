- 查找：`helm   search repo  ingress-nginx`

- 拉取 chart 到本地：`helm pull ingress-nginx/ingress-nginx --version 4.0.19`

- 修改 `value.yml` 如下：
    - hostNetwork 设置为 true
    - dnsPolicy 设置为 ClusterFirstWithHostNet
    - 类型更改为 kind: DaemonSet
    - 设置ingressClassResource 为默认 default: true
    - 指定节点部署 nodeSelector 为 ingress: "true"
  ```nashorn js
  diff values.yaml values.yaml.orig
  63c63
  <   dnsPolicy: ClusterFirstWithHostNet
  ---
  >   dnsPolicy: ClusterFirst
  86c86
  <   hostNetwork: true
  ---
  >   hostNetwork: false
  110c110
  <     default: true
  ---
  >     default: false
  191c191
  <   kind: DaemonSet
  ---
  >   kind: Deployment
  291d290
  <     ingress: "true"
  ```

- 给节点你打标记：`kubectl label node 172.20.47.4 ingress=true`
- 部署ingress：`helm install ingress-nginx -f values.yaml . -n ingress-nginx`

```bash
 kubectl  get pod -n ingress-nginx -o wide
NAME                             READY   STATUS    RESTARTS   AGE   IP            NODE          NOMINATED NODE   READINESS GATES
ingress-nginx-controller-6g4pg   1/1     Running   0          79s   172.20.47.4   172.20.47.4   <none>           <none>
```


