---
title: gitlab开启https
top: false
cover: false
toc: true
mathjax: true
date: 2020-01-07 17:11:59
password:
summary:
tags: 
 - gitlab
 - ssl
 - https
 - 证书
categories: 运维
---

## Gitlab开启Https

### 建立认证目录

```shell
mkdir -p /etc/gitlab/ssl
chmod 700 /etc/gitlab/ssl
```

### 建立证书

```shell
# step 1 创建private key （记住输入的密码（Pass phrase））
openssl genrsa -des3 -out /etc/gitlab/ssl/server.key 2048

# step 2 生成 Certificate Request
openssl req -new -key /etc/gitlab/ssl/gitlab.domain.com.key -out /etc/gitlab/ssl/server.csr
```

>  Enter Country Name CN
>  Enter State or Province Full Name HB
>  Enter City Name WuHan
>  Enter Organization Name 
>  Enter Company Name EV-IV
>  Enter Organizational Unit Name
>  Enter server hostname i.e. URL gitlab.domain.com
>  Enter Admin Email Address
>  Skip Challenge Password (Hit Enter)
>  Skip Optional Company Name (Hit Enter)

```shell
#在加载SSL支持的Nginx并使用上述私钥时要除去刚才设置的口令： 
#step3 备份csr文件及去除命令，直接覆盖了server.key了
openssl rsa -inserver.key.org -out server.key
#step 4最后标记证书使用上述私钥和CSR：（把csr标记后转换成了crt nginx要用key和crt文件）
openssl x509 -req -days 365 -in server.csr -signkey server.key -out server.crt
```

### 修改gitlab配置

```shell
vim /etc/gitlab/gitlab.rb

external_url 'https://161.189.27.8:8090/'

nginx['redirect_http_to_https']= true
nginx['ssl_client_certificate'] = "/etc/gitlab/ssl/server.crt"
nginx['ssl_certificate']= "/etc/gitlab/ssl/server.crt"
nginx['ssl_certificate_key']= "/etc/gitlab/ssl/server.key"
```

### 重启gitlab

```shell
gitlab-ctl reconfigure
gitlab-ctl restart
```

最后通过访问https地址进行访问测试



### 相关问题

Q1 由于ssl证书为自签证书 git clone报错

```shell
git clone https://161.189.27.8:8090/dqdev/pythogoras.git
Cloning into 'pythogoras'...
fatal: unable to access 'https://161.189.27.8:8090/dqdev/pythogoras.git/': SSL certificate problem: self signed certificate
```

关闭git ssl验证

```bash
git config --global http.sslVerify false 关闭
git config --global http.sslVerify true  开启
```

