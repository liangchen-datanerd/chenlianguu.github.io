---
title: Kubernetes单节点部署
top: false
toc: true
mathjax: true
date: 2020-04-03 14:17:11
index_img: https://i.loli.net/2020/05/20/Bb7yY1MGgtRL8Kl.png
summary:
tags: 
 - kubernetes
 - 部署
categories: kubernetes
---

## 准备环境

```shell
关闭防火墙：
$ systemctl stop firewalld
$ systemctl disable firewalld

关闭selinux：
$ sed -i 's/enforcing/disabled/' /etc/selinux/config 
$ setenforce 0
关闭swap：
$ swapoff -a $ 临时
$ vim /etc/fstab $ 永久

添加主机名与IP对应关系（记得设置主机名）：
$ cat /etc/hosts
192.168.31.61 k8s-master
192.168.31.62 k8s-node1
192.168.31.63 k8s-node2

将桥接的IPv4流量传递到iptables的链：
$ cat > /etc/sysctl.d/k8s.conf << EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
$ sysctl --system

卸载docker
因为k8s默认会安装docker，所以如果系统安装过就需要将其卸载，不然可能出现版本不兼容安装上的情况，卸载命令如下。
$ yum remove -y 'rpm -qa |grep docker'
$ rm -rf /var/lib/docker

配置docker源镜像
vim /etc/docker/daemin.json

{
  "registry-mirrors": [
        "https://dockerhub.azk8s.cn",
        "https://b3sst9pc.mirror.aliyuncs.com",
        "https://hub-mirror.c.163.com"
]
}

systemctl daemon-reload
systemctl restart docker

```

## 安装etcd kuberetes

```shell
# 配置yum源
$ cat > /etc/yum.repos.d/kubernetes.repo << EOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg 
https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

# 安装
yum install -y etcd kubernetes
```

## 修改配置

```
# 将KUBE_ADMISSION_CONTROL选项中的ServiceAccount删除掉
vim /etc/kubernetes/apiserver

# KUBE_ADMISSION_CONTROL="--admission-control=NamespaceLifecycle,NamespaceExists,LimitRanger,SecurityContextDeny,ServiceAccount,ResourceQuota"
KUBE_ADMISSION_CONTROL="--admission_control=NamespaceLifecycle,NamespaceExists,LimitRanger,SecurityContextDeny,ResourceQuota"

```

## 启动服务

```shell
systemctl start etcd

systemctl start docker

systemctl start kube-apiserver

systemctl start kube-controller-manager

systemctl start kube-scheduler

systemctl start kubelet

systemctl start kube-proxy
```

## 部署dashboard

```yaml
kind: Deployment
apiVersion: extensions/v1beta1
metadata:
  labels:
    app: kubernetes-dashboard
    version: v1.10.1  
  name: kubernetes-dashboard
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kubernetes-dashboard
  template:
    metadata:
      labels:
        app: kubernetes-dashboard
    spec:
      containers:
      - name: kubernetes-dashboard
        image: lizhenliang/kubernetes-dashboard-amd64:v1.10.1 #已修改为国内镜像
        imagePullPolicy: Always
        ports:
        - containerPort: 9090
          protocol: TCP
        livenessProbe:
          httpGet:
            path: /
            port: 9090
          initialDelaySeconds: 30
          timeoutSeconds: 30
---
kind: Service
apiVersion: v1
metadata:
  labels:
    app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kube-system
  type: NodePort
  ports:
  - port: 80
    targetPort: 9090
  selector:
    app: kubernetes-dashboard
```

创建服务

```shell
kubectl create -f kubernetes-dashboard.yaml
kubectl get pods --all-namespaces=true
NAMESPACE     NAME                                    READY     STATUS              RESTARTS   AGE
kube-system   kubernetes-dashboard-1745970253-90ppp   0/1       ContainerCreating   0          24m
```

完成后可通过浏览器访问 http://yourip:8080/ui