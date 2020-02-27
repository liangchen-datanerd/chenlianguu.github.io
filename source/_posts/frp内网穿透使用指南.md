---
title: frp内网穿透使用指南
top: false
toc: true
mathjax: true
date: 2020-02-27 10:37:06
index_img: https://i.loli.net/2020/02/27/WbYe8Q6Ah2dL4Ty.png
summary:
tags:
 - 部署
 - 内网穿透
categories: 运维
---

## 背景

frp 是一个可用于内网穿透的高性能的反向代理应用，支持 tcp, udp 协议，为 http 和 https 应用协议提供了额外的能力，且尝试性支持了点对点穿透。

假如部署的k8s集群，需要暴露给外网使用。花生壳儿、向日葵等这些共可以解决，但是这些工具都为收费工具，使用不够灵活，这时frp就派上了用场，使用起来非常简单便捷。

 👉 [官方github地址](https://github.com/fatedier/frp/blob/master/README_zh.md)

## 配置

frp是标准的c/s架构，配置分为server端还有client端，首先需要在github页release端下载对应的版本

### server端配置

server端需要部署在有公网ip地址的实例上，这里以centos为例

```shell
# 下载server端
wget https://github.com/fatedier/frp/releases/download/v0.31.1/frp_0.31.1_linux_arm64.tar.gz
# 解压
tar -zxcf frp_0.31.1_linux_arm64.tar.gz
cd frp_0.31.1_linux_arm64
```

frps.ini为server端的配置文件

```shell
[common]
# 绑定端口
bind_port = 7000

vhost_http_port = 8076

# web ui访问端口
dashboard_port = 7500

# web ui用户名及密码
dashboard_user = admin
dashboard_pwd =123456

# 最大连接数
max_pool_count = 10

authentication_timeout = 900

[ssh]

listen_port = 22

auth_token =abcdefg

```

后台启动frp server端服务

```shell
nohup ./frps -c ./frps.ini &
```

访问web ui  http://your ip:7500

![微信截图_20200116165729.png](https://i.loli.net/2020/01/16/6s4YhmldxGnV15a.png)

### client端配置

假定kubernetes dashboard需要暴露外网访问，这里需要在提供服务的节点，开启frp服务，这里以centos为例

```shell
# 下载server端
wget https://github.com/fatedier/frp/releases/download/v0.31.1/frp_0.31.1_linux_arm64.tar.gz
# 解压
tar -zxcf frp_0.31.1_linux_arm64.tar.gz
cd frp_0.31.1_linux_arm64
```

配置客户端

```ini
[common]
server_addr = 161.0.0.0 #填写server端地址
server_port = 7000
auth_token=abcdefg
pool_count=1

[wu-k8s]
type = tcp
local_ip = 127.0.0.1
local_port = 31628  # dashboad访问端口
remote_port = 31628
```

后台启动frp client端服务

```shell
nohup ./frpc -c ./frpc.ini &
```

启动之后查看frp web ui，确定是否配置成功

![微信截图_20200116172139.png](https://i.loli.net/2020/01/16/4dKhBWfYv9VGywP.png)



最后通过访问外网ip及映射端口访问对应服务即可，实现了内网穿透。其他组件需要暴露外网直接修改frpc.ini文件重新启动client端即可



frp除了实现内网穿透功能，还可以绑定自定义域名等，详细见官方文档。