## 实现 HTTPS 访问 

- 生成证书：`openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "/CN=myapp.linux.io"`
- 创建 secret ：`kubectl create secret tls ca-secret --cert=tls.crt --key=tls.key -n learn-ingress`
- 使用浏览器访问 `http://myapp.linux.io` 会自动跳转到https
