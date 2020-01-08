---
title: K8s监控部署方案
top: false
cover: https://i.loli.net/2020/01/08/vN8FjS6aoxEGYRI.png
toc: true
mathjax: true
date: 2020-01-07 17:17:22
password:
summary:
tags:
 - kubernetes
 - grafana
 - prometheus
 - 监控
categories: kubernetes
---

## 架构
![grafana](https://i.loli.net/2020/01/08/DVBiRAwaNsnYbgL.png)

## 部署

**yaml文件地址 👉🏻   [pythagoras](https://161.189.27.8:8090/dqdev/pythogoras/tree/master/k8s-yaml/prometheus)**

### 1 在k8s集群中创建namespace

```yaml
apiVersion: v1
kind: Namespace
metadata: 
  name: ns-monitor
  labels:
    name: ns-monitor
    
kubectl apply -f namespace.yaml
```

### 2 安装node-exporter

在kubernetest集群中部署node-exporter，Node-exporter用于采集kubernetes集群中各个节点的物理指标，比如：Memory、CPU等。可以直接在每个物理节点是直接安装，这里我们使用DaemonSet部署到每个节点上，使用 hostNetwork: true 和 hostPID: true 使其获得Node的物理指标信息，配置tolerations使其在master节点也启动一个pod。

node-exporter.yaml

```yaml
kind: DaemonSet
apiVersion: apps/v1beta2
metadata: 
  labels:
    app: node-exporter
  name: node-exporter
  namespace: ns-monitor
spec:
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: node-exporter
  template:
    metadata:
      labels:
        app: node-exporter
    spec:
      containers:
        - name: node-exporter
          image: prom/node-exporter:v0.16.0
          ports:
            - containerPort: 9100
              protocol: TCP
              name:	http
      hostNetwork: true
      hostPID: true
      tolerations:
        - effect: NoSchedule
          operator: Exists

---
kind: Service
apiVersion: v1
metadata:
  labels:
    app: node-exporter
  name: node-exporter-service
  namespace: ns-monitor
spec:
  ports:
    - name:	http
      port: 9100
      nodePort: 31672
      protocol: TCP
  type: NodePort
  selector:
    app: node-exporter
```

```shell
kubectl apply -f node-exporter.yaml
```

***检查是否执行成功(对应pod及svc)** 👉🏻 **

```shell
➜  ~ kubectl get pod -n ns-monitor
NAME                          READY   STATUS    RESTARTS   AGE
grafana-547699f75-lxljq       1/1     Running   0          3h37m
node-exporter-75nmc           0/1     Pending   0          20h
node-exporter-t29kx           1/1     Running   0          20h
node-exporter-z6s7x           1/1     Running   0          20h
prometheus-7d7654554d-f5fvf   1/1     Running   0          3h45m
➜  ~ kubectl get svc -n ns-monitor
NAME                    TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
grafana-service         NodePort   10.1.242.56    <none>        3000:31026/TCP   3h38m
node-exporter-service   NodePort   10.1.130.7     <none>        9100:31672/TCP   20h
prometheus-service      NodePort   10.1.133.130   <none>        9090:30753/TCP   3h45m
```

![image-20200103143320329](https://i.loli.net/2020/01/08/TP6tIlvFKhgzHuZ.png)

### 3 部署Prometheus pod

prometheus.yaml 中包含rbac认证、ConfigMap等

```shell
kubectl apply -f prometheus.yaml 
```

*检查是否执行成功(对应pod及svc)** 👉🏻 

```shell
➜  ~ kubectl get pod -n ns-monitor
NAME                          READY   STATUS    RESTARTS   AGE
grafana-547699f75-lxljq       1/1     Running   0          3h37m
node-exporter-75nmc           0/1     Pending   0          20h
node-exporter-t29kx           1/1     Running   0          20h
node-exporter-z6s7x           1/1     Running   0          20h
prometheus-7d7654554d-f5fvf   1/1     Running   0          3h45m
➜  ~ kubectl get svc -n ns-monitor
NAME                    TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
grafana-service         NodePort   10.1.242.56    <none>        3000:31026/TCP   3h38m
node-exporter-service   NodePort   10.1.130.7     <none>        9100:31672/TCP   20h
prometheus-service      NodePort   10.1.133.130   <none>        9090:30753/TCP   3h45m
```

![image-20200103143645494](https://i.loli.net/2020/01/08/fdFs8EN2xhHwCM6.png)

### 4 在k8s中部署grafana

```shell
kubectl apply -f grafana.yaml
```

**检查是否执行成功(对应pod及svc)** 👉🏻 

```shell
➜  ~ kubectl get pod -n ns-monitor
NAME                          READY   STATUS    RESTARTS   AGE
grafana-547699f75-lxljq       1/1     Running   0          3h37m
node-exporter-75nmc           0/1     Pending   0          20h
node-exporter-t29kx           1/1     Running   0          20h
node-exporter-z6s7x           1/1     Running   0          20h
prometheus-7d7654554d-f5fvf   1/1     Running   0          3h45m
➜  ~ kubectl get svc -n ns-monitor
NAME                    TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
grafana-service         NodePort   10.1.242.56    <none>        3000:31026/TCP   3h38m
node-exporter-service   NodePort   10.1.130.7     <none>        9100:31672/TCP   20h
prometheus-service      NodePort   10.1.133.130   <none>        9090:30753/TCP   3h45m
```

### 5 配置grafana数据源

把prometheus配置成数据源 ：http://prometheus-service.ns-monitor:9090

![image-20200103144416930](https://i.loli.net/2020/01/08/fzxnrR5iguDjlHk.png)

### 6 倒入dashboard

把 kubernetes的Dashboard的模板导入进来，直接把JSON格式内容复制进来。

![image-20200103145516630](https://i.loli.net/2020/01/08/6XkA5hEjN1OWioS.png)

## 效果图

![image-20200103145627198](https://i.loli.net/2020/01/08/bqEolIi8KVSGuHk.png)

## Ref

- [使用Prometheus监控kubernetes集群](https://jimmysong.io/kubernetes-handbook/practice/using-prometheus-to-monitor-kuberentes-cluster.html)
- [k8s安装Prometheus+Grafana](https://www.jianshu.com/p/ac8853927528)

- [github-kubernetes-promethues](https://github.com/giantswarm/kubernetes-prometheus)
- [github-giantswarm-promethues](https://github.com/giantswarm/prometheus)