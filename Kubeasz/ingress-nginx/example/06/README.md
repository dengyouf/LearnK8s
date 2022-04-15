## Nginx 密码认证

- `yum install httpd -y`
- `htpasswd -c authfile  admin`
- `kubectl create secret generic basic-auth --from-file=authfile -n learn-ingress`
