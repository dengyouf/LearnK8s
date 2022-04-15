## 前后端分离 

目标：通过 Ingress Nginx 的 Rewrite 功能，将 "/api-a" 重写为 "/"

```
[root@k8s-master01 ~]# kubectl  get svc -n learn-ingress
NAME              TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)   AGE
backend-api-svc   ClusterIP   10.68.91.12   <none>        80/TCP    3m50s
[root@k8s-master01 ~]# curl  10.68.91.12
<h1> backend for ingress rewrite </h1>

<h2> Path: /api-a </h2>


<a href="http://gaoxin.kubeasy.com"> Kubeasy </a>
[root@k8s-master01 ~]# kubectl  get ingress -n learn-ingress
NAME                     CLASS   HOSTS               ADDRESS   PORTS   AGE
backend-api-ingress      nginx   redirect.linux.io             80      3m9s
myapp-ingress-redirect   nginx   myapp.redirect.io             80      100m
[root@k8s-master01 ~]# curl -H "Host:redirect.linux.io" 172.20.47.4/api-a
<h1> backend for ingress rewrite </h1>

<h2> Path: /api-a </h2>


<a href="http://gaoxin.kubeasy.com"> Kubeasy </a>
```
