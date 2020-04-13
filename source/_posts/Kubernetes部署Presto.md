---
title: Kubernetes部署Presto
top: false
toc: true
mathjax: true
date: 2020-03-28 17:47:45
index_img: https://i.loli.net/2020/03/28/InC7YBLkgTVEAJ4.png
summary:
tags:
 - kubernetes
 - presto
 - rook
 - ceph
 - 部署
categories: kubernetes
---
## 一、介绍

> Presto

Presto是一个分布式SQL查询引擎，用于查询分布在一个或多个不同数据源中的大数据集。完整安装包括一个Coordinator和多个Worker。 由客户端提交查询，从Presto命令行CLI提交到Coordinator。 Coordinator进行解析，分析并执行查询计划，然后分发处理队列到Worker。


Presto是完全基于内存的分布式大数据查询引擎，所有查询和计算都在内存中执行。

Presto的输入是SQL语句；输出是具体的SQL执行结果。

Presto可以对接不同的数据源，例如MySQL、Hive等。

Presto可以对SQL的查询过程进行优化，包括SQL本身的执行计划优化，以及用分布式查询提高并发等。

Presto不是数据库，并不能处理在线事务。

> ceph 

Ceph是当前非常流行的开源分布式存储系统，具有高扩展性、高性能、高可靠性等优点，同时提供块存储服务(rbd)、对象存储服务(rgw)以及文件系统存储服务(cephfs)，Ceph在存储的时候充分利用存储节点的计算能力，在存储每一个数据时都会通过计算得出该数据的位置，尽量的分布均衡。

> rook

Rook是一个开放源码的云本机存储协调器，提供平台、框架和对各种存储解决方案的支持，以便与云本机环境进行本机集成。

Rook将存储软件转变为自我管理、自我扩展和自我修复的存储服务。它通过自动化部署、引导、配置、供应、扩展、升级、迁移、灾难恢复、监视和资源管理来实现这一点。Rook使用底层云本地容器管理、调度和协调平台提供的设施来执行其职责。

Rook利用扩展点深入集成到云本机环境中，为调度、生命周期管理、资源管理、安全、监控和用户体验提供无缝体验。

> 基于kubernetes部署presto

将presto数据持久化到ceph集群中，保证presto的数据高可用

## 二、软件版本

- Kubernetes v1.15（`单节点环境亦可`）
- Rook v1.0.2
- CentOS 7 

## 三、部署rook及ceph

rook的部署可以使用kubernetes的包管理工具[helm](https://helm.sh/)安装，由于rook及kubernetes版本众多以及涉及到相关源等问题，本文没有采用helm安装方式部署rook

```shell
$ git clone https://github.com/rook/rook.git 
$ cd rook 
$ git checkout v1.0.2
$ cd cluster/examples/kubernetes/ceph/
```

`cluster/examples/kubernetes/ceph/`目录包含部署rook相关文件

安装Rook Common Objects & Operator

```shell
kubectl create -f common.yaml
kubectl create -f operator.yaml
```

检查`rook-ceph-operator,rook-ceph-agent`, rook-discover`安装成功

```shell
$ kubectl get pods -n rook-ceph 
NAME                                   READY       STATUS        RESTARTS    AGE 
rook-ceph-agent-chj4l                  1/1         Running       0           84s 
rook-ceph-operator-548b56f995-v4mvp    1/1         Running       0           4m5s 
rook-discover-vkkvl                    1/1         Running       0    
```

### 3.1 部署Rook Ceph Cluster

```shell
kubectl apply -f cluster.yml
```

检查是否部署成功

```shell
kubectl get pods -n rook-ceph
NAME                                  READY   STATUS      RESTARTS   AGE
rook-ceph-agent-chl7b                 1/1     Running     0          4h14m
rook-ceph-agent-tbx4l                 1/1     Running     2          4h14m
rook-ceph-mgr-a-bf88cfdc4-wvhjb       1/1     Running     0          3h52m
rook-ceph-mon-a-df88f8cbf-dklvj       1/1     Running     0          3h53m
rook-ceph-operator-548b56f995-q47jb   1/1     Running     5          4h14m
rook-ceph-osd-0-655957b867-p8tf7      1/1     Running     0          3h51m
rook-ceph-osd-1-7578dc7874-f9q64      1/1     Running     0          3h50m
rook-ceph-osd-prepare-node-2-nvr7d    0/2     Completed   0          3h45m
rook-ceph-osd-prepare-node-3-2mm2p    0/2     Completed   0          3h45m
rook-discover-44789                   1/1     Running     3          4h14m
rook-discover-c4r7d                   1/1     Running     0          4h14m
```

### 3.2 测试ceph cluster

创建StorageClass以便ceph cluster动态创建PV、

```shell
kubectl create -f storageclass-test.yaml

# 查找Rook Operator
$ kubectl get pod -n rook-ceph --selector=app=rook-ceph-operator 
NAME                                  READY   STATUS    RESTARTS   AGE
rook-ceph-operator-548b56f995-q47jb   1/1     Running   5          4h23m

# 检查ceph cluster健康状态
kubectl exec -n rook-ceph -it rook-ceph-operator-548b56f995-q47jb -- ceph status
  cluster:
    id:     bed40efd-b3e8-4b18-b9a0-e8723f51e747
    health: HEALTH_OK

  services:
    mon: 1 daemons, quorum a (age 4h)
    mgr: a(active, since 3h)
    osd: 2 osds: 2 up (since 3h), 2 in (since 3h)

  data:
    pools:   1 pools, 100 pgs
    objects: 0 objects, 0 B
    usage:   13 GiB used, 22 GiB / 35 GiB avail
    pgs:     100 active+clean
    
# 查看ceph磁盘信息
kubectl exec -n rook-ceph -it rook-ceph-operator-548b56f995-q47jb -- ceph df
RAW STORAGE:
    CLASS     SIZE       AVAIL      USED       RAW USED     %RAW USED
    hdd       35 GiB     22 GiB     13 GiB       13 GiB         37.40
    TOTAL     35 GiB     22 GiB     13 GiB       13 GiB         37.40

POOLS:
    POOL            ID     STORED     OBJECTS     USED     %USED     MAX AVAIL
    replicapool      1        0 B           0      0 B         0        19 GiB
```

### 3.3 配置ceph dashborad

在cluster.yaml文件中默认已经启用了ceph dashboard，查看dashboard的service：

```shell
[root@node-1 presto-kubernetes]# kubectl get svc -n rook-ceph
NAME                                     TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)             AGE
rook-ceph-mgr                            ClusterIP   10.1.73.175    <none>        9283/TCP            6h29m
rook-ceph-mgr-dashboard                  ClusterIP   10.1.228.105   <none>        8443/TCP            6h29m
rook-ceph-mgr-dashboard-external-https   NodePort    10.1.38.28     <none>        8443:30102/TCP      21m
rook-ceph-mon-a                          ClusterIP   10.1.130.153   <none>        6789/TCP,3300/TCP   6h31m
```

rook-ceph-mgr-dashboard监听的端口是8443，创建nodeport类型的service以便集群外部访问。

```shell
kubectl apply -f rook/cluster/examples/kubernetes/ceph/dashboard-external-https.yaml
```

查看一下nodeport暴露的端口，这里是30102端口：

```
[root@node-1 presto-kubernetes]# kubectl get svc -n rook-ceph
NAME                                     TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)             AGE
rook-ceph-mgr                            ClusterIP   10.1.73.175    <none>        9283/TCP            6h29m
rook-ceph-mgr-dashboard                  ClusterIP   10.1.228.105   <none>        8443/TCP            6h29m
rook-ceph-mgr-dashboard-external-https   NodePort    10.1.38.28     <none>        8443:30102/TCP      21m
rook-ceph-mon-a                          ClusterIP   10.1.130.153   <none>        6789/TCP,3300/TCP   6h31m
```

获取Dashboard的登陆账号和密码

```shell
[root@node-1 ~]# MGR_POD=`kubectl get pod -n rook-ceph | grep mgr | awk '{print $1}'`
[root@node-1 ~]# kubectl -n rook-ceph logs $MGR_POD | grep password
debug 2020-02-25 23:54:04.280 7ffb08d89700  0 log_channel(audit) log [DBG] : from='client.4198 -' entity='client.admin' cmd=[{"username": "admin", "prefix": "dashboard set-login-credentials", "password": "Wo9nf64nDs", "target": ["mgr", ""], "format": "json"}]: dispatch
debug 2020-02-25 23:58:11.177 7f8606e0e700  0 log_channel(audit) log [DBG] : from='client.4373 -' entity='client.admin' cmd=[{"username": "admin", "prefix": "dashboard set-login-credentials", "password": "Wo9nf64nDs", "target": ["mgr", ""], "format": "json"}]: dispatch
```

找到username和password字段，我这里是admin，Wo9nf64nDs
打开浏览器输入任意一个Node的IP+nodeport端口，这里使用master节点 ip访问：https://localhost:30102/#/dashboard

## 四、部署presto

### 4.1 部署步骤

- 构建镜像`Dockerfile`，hdfs-site及core-site配置文件可以构建到镜像中，方便后续配置hive connector

  ```dockerfile
  FROM centos:centos7.5.1804
  MAINTAINER **@mail.com
  
  RUN mkdir -p /etc/hadoop/conf
  
  ADD jdk-8u101-linux-x64.tar.gz /opt
  ADD presto-server-0.208.tar.gz /opt
  ADD core-site.xml /etc/hadoop/conf
  ADD hdfs-site.xml /etc/hadoop/conf
  
  ENV PRESTO_HOME /opt/presto-server-0.208
  ENV JAVA_HOME /opt/jdk1.8.0_101
  ENV PATH $JAVA_HOME/bin:$PATH
  ```

- 配置server端相关配置 `presto-config-cm.yaml`

  ```yaml
  apiVersion: v1
  kind: ConfigMap
  metadata:
    name: presto-config-cm
    labels:
      app: presto-coordinator
  data:
    bootstrap.sh: |-
      #!/bin/bash
  
      cd /root/bootstrap
  
      mkdir -p $PRESTO_HOME/etc/catalog
  
      cat ./node.properties > $PRESTO_HOME/etc/node.properties
      cat ./jvm.config > $PRESTO_HOME/etc/jvm.config
      cat ./config.properties > $PRESTO_HOME/etc/config.properties
      cat ./log.properties > $PRESTO_HOME/etc/log.properties
  
      sed -i 's/${COORDINATOR_NODE}/'$COORDINATOR_NODE'/g' $PRESTO_HOME/etc/config.properties
  
      for cfg in ../catalog/*; do
        cat $cfg > $PRESTO_HOME/etc/catalog/${cfg##*/}
      done
  
      $PRESTO_HOME/bin/launcher run --verbose
    node.properties: |-
      node.environment=production
      node.data-dir=/var/presto/data
    jvm.config: |-
      -server
      -Xmx16G
      -XX:+UseG1GC
      -XX:G1HeapRegionSize=32M
      -XX:+UseGCOverheadLimit
      -XX:+ExplicitGCInvokesConcurrent
      -XX:+HeapDumpOnOutOfMemoryError
      -XX:+ExitOnOutOfMemoryError
    config.properties: |-
      coordinator=${COORDINATOR_NODE}
      node-scheduler.include-coordinator=true
      http-server.http.port=8080
      query.max-memory=10GB
      query.max-memory-per-node=1GB
      query.max-total-memory-per-node=2GB
      discovery-server.enabled=true
      discovery.uri=http://presto-coordinator-service:8080
    log.properties: |-
      com.facebook.presto=INFO
  ```

- catalog相关配置`presto-catalog-config-cm.yaml`

  ```yaml
  apiVersion: v1
  kind: ConfigMap
  metadata:
    name: presto-catalog-config-cm
    labels:
      app: presto-coordinator
  data:
    hive.properties: |-
      connector.name=hive-hadoop2
      hive.metastore.uri=thrift://ip:9083
      hive.config.resources=/etc/hadoop/conf/core-site.xml,/etc/hadoop/conf/hdfs-site.xml
    mysql.properties: |-
      connector.name=mysql
      connection-url=jdbc:mysql://ip:3306
      connection-user=root
      connection-password=123456
  ```

- worker及coordinator的部署`deployment.yaml`

  ```yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: presto-coordinator
  spec:
    replicas: 1
    revisionHistoryLimit: 10
    selector:
      matchLabels:
        app: presto-coordinator
    template:
      metadata:
        labels:
          app: presto-coordinator
      spec:
        containers:
          - name: presto-coordinator
            image: chenlianguu/presto-server:dm-0.208
            command: ["bash", "-c", "sh /root/bootstrap/bootstrap.sh"]
            ports:
              - name: http-coord
                containerPort: 8080
                protocol: TCP
            env:
              - name: COORDINATOR_NODE
                value: "true"
            volumeMounts:
              - name: presto-config-volume
                mountPath: /root/bootstrap
              - name: presto-catalog-config-volume
                mountPath: /root/catalog
              - name: presto-data-volume
                mountPath: /var/presto/data
            readinessProbe:
              initialDelaySeconds: 10
              periodSeconds: 5
              httpGet:
                path: /v1/cluster
                port: http-coord
        volumes:
          - name: presto-config-volume
            configMap:
              name: presto-config-cm
          - name: presto-catalog-config-volume
            configMap:
              name: presto-catalog-config-cm
          - name: presto-data-volume
            emptyDir: {}
  ---
  kind: Service
  apiVersion: v1
  metadata:
    labels:
      app: presto-coordinator
    name: presto-coordinator-service
  spec:
    ports:
      - port: 8080
        targetPort: http-coord
        name: http-coord
    selector:
      app: presto-coordinator
    type: NodePort
  ---
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: presto-worker
  spec:
    replicas: 2
    revisionHistoryLimit: 10
    selector:
      matchLabels:
        app: presto-worker
    template:
      metadata:
        labels:
          app: presto-worker
      spec:
        initContainers:
          - name: wait-coordinator
            image:  chenlianguu/presto-server:dm-0.208
            command: ["bash", "-c", "until curl -sf http://presto-coordinator-service:8080/ui/; do echo 'waiting for coordinator started...'; sleep 2; done;"]
        containers:
          - name: presto-worker
            image: chenlianguu/presto-server:dm-0.208
            command: ["bash", "-c", "sh /root/bootstrap/bootstrap.sh"]
            ports:
              - name: http-coord
                containerPort: 8080
                protocol: TCP
            env:
              - name: COORDINATOR_NODE
                value: "false"
            volumeMounts:
              - name: presto-config-volume
                mountPath: /root/bootstrap
              - name: presto-catalog-config-volume
                mountPath: /root/catalog
              - name: presto-data-volume
                mountPath: /var/presto/data
            readinessProbe:
              initialDelaySeconds: 10
              periodSeconds: 5
              exec:
                command: ["bash", "-c", "curl -s http://presto-coordinator-service:8080/v1/node | tr ',' '\n' | grep -s $(hostname -i)"]
        volumes:
          - name: presto-config-volume
            configMap:
              name: presto-config-cm
          - name: presto-catalog-config-volume
            configMap:
              name: presto-catalog-config-cm
          - name: presto-data-volume
            emptyDir: {}
  ```

- 启动presto

  ```shell
  kubectl create -f presto-config-cm.yaml
  kubectl create -f presto-catalog-config-cm.yaml
  kubectl create -f deployment.yaml 
  ```

- 使用presto，找出外部地址

  ```shell
  [root@node-1 ~]# kubectl get svc presto-coordinator-service
  NAME     TYPE       CLUSTER-IP    EXTERNAL-IP   PORT(S)          AGE
  presto   NodePort   10.1.27.143   <none>        8080:32151/TCP   27h
  ```

- 测试Presto客户端连接

  ```shell
  wget https://repo1.maven.org/maven2/com/facebook/presto/presto-cli/0.208/presto-cli-0.208-executable.jar
  
  chmod +x presto-cli-0.208-executable.jar 
  
  ./presto-cli-0.208-executable.jar --server 192.168.112.240:30418 --catalog hive --schema default
  presto:default>
  ```



### 4.2 挂载ceph卷到presto

> 由于github开源版本，没有提供数据卷挂载，因此需要相关yaml文件进行修改，配置rook提供RBD服务

rook可以提供以下3类型的存储：
 Block: Create block storage to be consumed by a pod
 Object: Create an object store that is accessible inside or outside the Kubernetes cluster
 Shared File System: Create a file system to be shared across multiple pods
在提供（Provisioning）块存储之前，需要先创建StorageClass和存储池。K8S需要这两类资源，才能和Rook交互，进而分配持久卷（PV）。
在kubernetes集群里，要提供rbd块设备服务，需要有如下步骤：

1. 创建rbd-provisioner pod
2. 创建rbd对应的storageclass
3. 创建pvc，使用rbd对应的storageclass
4. 创建pod使用rbd pvc

通过rook创建Ceph Cluster之后，rook自身提供了rbd-provisioner服务，所以我们不需要再部署其provisioner。

```shell
# 配置storageclass
kubectl apply -f rook/cluster/examples/kubernetes/ceph/storageclass.yaml

# 检查storageclass
kubectl get storageclass
NAME              PROVISIONER          AGE
rook-ceph-block   ceph.rook.io/block   171m
```

> 修改preso-kubernetes文件夹中的worker相关配置

```shell
vim deployment.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: presto-worker
spec:
  replicas: 2
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: presto-worker
  template:
    metadata:
      labels:
        app: presto-worker
    spec:
      initContainers:
        - name: wait-coordinator
          image:  chenlianguu/presto-server:dm-0.208
          command: ["bash", "-c", "until curl -sf http://presto-coordinator-service:8080/ui/; do echo 'waiting for coordinator started...'; sleep 2; done;"]
      containers:
        - name: presto-worker
          image: chenlianguu/presto-server:dm-0.208
          command: ["bash", "-c", "sh /root/bootstrap/bootstrap.sh"]
          ports:
            - name: http-coord
              containerPort: 8080
              protocol: TCP
          env:
            - name: COORDINATOR_NODE
              value: "false"
          volumeMounts:
            - name: presto-config-volume
              mountPath: /root/bootstrap
            - name: presto-catalog-config-volume
              mountPath: /root/catalog
            - name: presto-data-volume
              mountPath: /var/presto/data
          readinessProbe:
            initialDelaySeconds: 10
            periodSeconds: 5
            exec:
              command: ["bash", "-c", "curl -s http://presto-coordinator-service:8080/v1/node | tr ',' '\n' | grep -s $(hostname -i)"]
      volumes:
        - name: presto-config-volume
          configMap:
            name: presto-config-cm
        - name: presto-catalog-config-volume
          configMap:
            name: presto-catalog-config-cm
        - name: presto-data-volume
          persistentVolumeClaim:
            claimName: presto-data-claim
```

重新apply yaml文件

```shell
kubectl aplly -f worker-deployment.yaml
```

检查是否配置成功

```shell
[root@node-1 ~]# kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                       STORAGECLASS      REASON   AGE
pvc-1ff7bb97-238e-4ebd-bfc3-fb9db1ae5656   10Gi       RWO            Delete           Bound    default/presto-data-claim   rook-ceph-block            100m
[root@node-1 ~]# kubectl get pvc
NAME                STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS      AGE
presto-data-claim   Bound    pvc-1ff7bb97-238e-4ebd-bfc3-fb9db1ae5656   10Gi       RWO            rook-ceph-block   100m
```

注意：这里的pv会自动创建，当提交了包含 StorageClass 字段的 PVC 之后，Kubernetes 就会根据这个 StorageClass 创建出对应的 PV，这是用到的是Dynamic Provisioning机制来动态创建pv，PV 支持 Static 静态请求，和动态创建两种方式。

```shell
# ceph集群端检查
[root@node-1 ~]# kubectl exec -n rook-ceph -it rook-ceph-operator-548b56f995-q47jb -- rbd info -p replicapool pvc-1ff7bb97-238e-4ebd-bfc3-fb9db1ae5656
rbd image 'pvc-1ff7bb97-238e-4ebd-bfc3-fb9db1ae5656':
        size 10 GiB in 2560 objects
        order 22 (4 MiB objects)
        snapshot_count: 0
        id: 1dd4352e100d
        block_name_prefix: rbd_data.1dd4352e100d
        format: 2
        features: layering
        op_features:
        flags:
        create_timestamp: Wed Feb 26 05:23:09 2020
        access_timestamp: Wed Feb 26 05:23:09 2020
        modify_timestamp: Wed Feb 26 05:23:09 2020
```

登陆pod检查rbd设备

```shell
[root@node-1 presto-kubernetes]# kubectl exec -it worker-7b66b5cb5-d2vzt bash
root@worker-7b66b5cb5-d2vzt:/usr/lib/presto-server-0.167-t.0.3/etc# df -h
Filesystem                                                                                        Size  Used Avail Use% Mounted on
/dev/mapper/docker-253:0-203738-8a1bc94ee31c6ca3e406a6c740b84cd8bd2c681c568dc8a4007f273208bbd9fd   10G  1.4G  8.7G  14% /
tmpfs                                                                                              64M     0   64M   0% /dev
tmpfs                                                                                             1.9G     0  1.9G   0% /sys/fs/cgroup
/dev/mapper/centos-root                                                                            18G  6.3G   12G  36% /etc/hosts
shm                                                                                                64M     0   64M   0% /dev/shm
/dev/rbd0                                                                                          10G   33M   10G   1% /var/presto/data
tmpfs                                                                                             1.9G   12K  1.9G   1% /run/secrets/kubernetes.io/serviceaccount
tmpfs                                                                                             1.9G     0  1.9G   0% /proc/acpi
tmpfs                                                                                             1.9G     0  1.9G   0% /proc/scsi
tmpfs                                                                                             1.9G     0  1.9G   0% /sys/firmware
```



### 4.3 presto多源异构查询

> 相关connector采用configmap方式进行灵活配置，不同connector的配置参考[官网Doc](https://prestosql.io/docs/current/connector.html)

```yaml
# 创建presto-catalog-config-cm.yaml
cat << EOF > ～/presto-kubernetes/presto-catalog-config-cm.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: presto-catalog-config-cm
  labels:
    app: presto-coordinator
data:
  # hive相关配置
  hive.properties: |-
    connector.name=hive-hadoop2
    hive.metastore.uri=thrift://10.8.8.252:9083
  # mysql相关配置
  mysql.properties: |-
    connector.name=mysql
    connection-url=jdbc:mysql://example.net:3306
    connection-user=root
    connection-password=secret
  cassandra.properties: |-
    connector.name=cassandra
    cassandra.contact-points=host1,host2
EOF
```

修改`coordinator-deployment.yaml` `worker-deployment.yaml`两个yaml文件，把configmap挂载到对应的卷上去，因为使用的presto镜像为[gihub第三方镜像](https://github.com/johandry/presto-docker)，通过查询dockerfile得知，该镜像presto的catalog配置目录在`/usr/lib/presto/etc/catalog/`,将configmap挂载至该目录即可。

- **coordinator-deployment.yaml**

  ```yaml
  kind: Deployment
  apiVersion: apps/v1beta1
  metadata:
    name: coordinator
    labels:
      presto: coordinator
  spec:
    replicas: 1
    template:
      metadata:
        labels:
          presto: coordinator
      spec:
        containers:
        - env:
          - name: HTTP_SERVER_PORT
            value: "8080"
          - name: PRESTO_JVM_HEAP_SIZE
            value: "8"
          - name: PRESTO_MAX_MEMORY
            value: "10"
          - name: PRESTO_MAX_MEMORY_PER_NODE
            value: "1"
          image: johandry/presto
          livenessProbe:
            exec:
              command:
              - /etc/init.d/presto status | grep -q 'Running as'
            failureThreshold: 3
            periodSeconds: 300
            timeoutSeconds: 10
          name: presto-coordinator
          ports:
          - containerPort: 8080
          volumeMounts:
              - name: presto-data-volume
                mountPath: /var/presto/data
              - name: presto-catalog-config-volume
                mountPath: /usr/lib/presto/etc/catalog
        restartPolicy: Always
        volumes:
          - name: presto-data-volume
            persistentVolumeClaim:
              claimName: presto-data-claim
          - name: presto-catalog-config-volume
            configMap:
              name: presto-catalog-config-cm
  ```

- worker-deployment.yaml

  ```yaml
  apiVersion: v1
  kind: PersistentVolumeClaim
  metadata:
    name: presto-data-claim-presto
  spec:
    storageClassName: rook-ceph-block
    accessModes:
    - ReadWriteOnce
    resources:
      requests:
        storage: 10Gi
  ---
  
  apiVersion: apps/v1beta1
  kind: Deployment
  metadata:
    labels:
      presto: worker
    name: worker
  spec:
    replicas: 1
    template:
      metadata:
        labels:
          presto: worker
      spec:
        containers:
        - env:
          - name: HTTP_SERVER_PORT
            value: "8080"
          - name: PRESTO_JVM_HEAP_SIZE
            value: "8"
          - name: PRESTO_MAX_MEMORY
            value: "10"
          - name: PRESTO_MAX_MEMORY_PER_NODE
            value: "1"
          - name : COORDINATOR
            value: "presto"
          image: johandry/presto
          livenessProbe:
            exec:
              command:
              - /etc/init.d/presto status | grep -q 'Running as'
            failureThreshold: 3
            periodSeconds: 300
            timeoutSeconds: 10
          name: worker
          ports:
          - containerPort: 8080
          volumeMounts:
              - name: presto-data-volume
                mountPath: /var/presto/data
              - name: presto-catalog-config-volume
                mountPath: /usr/lib/presto/etc/catalog
        restartPolicy: Always
        volumes:
           - name: presto-data-volume
             persistentVolumeClaim:
              claimName: presto-data-claim-presto
           - name: presto-catalog-config-volume
             configMap:
              name: presto-catalog-config-cm
  ```

---

配置好之后需要生效，重新apply `presto-catalog-config-cm.yaml` `coordinator-deployment.yaml` `worker-deployment.yaml` 3个文件，并**删除coordinator的pod**，删除之后会重新生成新的pod，新的pod会载入新的connector

```shell
kubectl apply -f presto-catalog-config-cm.yaml
kubectl apply -f coordinator-deployment.yaml
kubectl apply -f worker-deployment.yaml
kubectl get pods
NAME                           READY   STATUS    RESTARTS   AGE
coordinator-8549f46c58-q8ngm   1/1     Running   0          9m29s
worker-d64744f4d-8j2vg         1/1     Running   0          9m10s
kubectl delete pod coordinator-8549f46c58-q8ngm worker-d64744f4d-8j2vg
```

### 4.4 presto相关connect的测试

#### 4.4.1 presto查询hive中数据

首先准备hive环境，这里直接利用docker启动hive环境，简单方便高效快捷，参考github项目[docker-hive](https://github.com/big-data-europe/docker-hive)

```shell
git clone https://github.com/big-data-europe/docker-hive.git
cd docker-hive 
# 启动docker-hive
docker-compose up -d
# 查看启动状态
                 Name                                Command                  State                          Ports
--------------------------------------------------------------------------------------------------------------------------------------
docker-hive_datanode_1                    /entrypoint.sh /run.sh           Up (healthy)   0.0.0.0:50075->50075/tcp
docker-hive_hive-metastore-postgresql_1   /docker-entrypoint.sh postgres   Up             5432/tcp
`docker-hive_hive-metastore_1              entrypoint.sh /opt/hive/bi ...   Up             10000/tcp, 10002/tcp, 0.0.0.0:9083->9083/tcp`
docker-hive_hive-server_1                 entrypoint.sh /bin/sh -c s ...   Up             0.0.0.0:10000->10000/tcp, 10002/tcp
docker-hive_namenode_1                    /entrypoint.sh /run.sh           Up (healthy)   0.0.0.0:50070->50070/tcp
docker-hive_presto-coordinator_1          ./bin/launcher run               Up             0.0.0.0:8080->8080/tcp

# 往hive里边加载数据
docker-compose exec hive-server bash
# /opt/hive/bin/beeline -u jdbc:hive2://localhost:10000
  > CREATE TABLE pokes (foo INT, bar STRING);
  > LOAD DATA LOCAL INPATH '/opt/hive/examples/files/kv1.txt' OVERWRITE INTO TABLE pokes;
```

hive-metastore的连接端口为9083，确认该端口处于监听状态 `presto连接hive进行测试`

```shell
./presto-cli-0.208-executable.jar --server localhost:30025 --catalog hive --schema default
presto:default> show schemas;
       Schema
--------------------
 default
 information_schema
(2 rows)

Query 20200409_121447_00002_3jete, FINISHED, 2 nodes
Splits: 18 total, 18 done (100.00%)
0:00 [2 rows, 35B] [16 rows/s, 290B/s]

presto:default> show tables;
 Table
-------
 `pokes`
(1 row)

Query 20200409_121459_00003_3jete, FINISHED, 2 nodes
Splits: 18 total, 18 done (100.00%)
0:00 [1 rows, 22B] [9 rows/s, 201B/s]

presto:default> select * from pokes;

Query 20200409_121548_00004_3jete, FAILED, 1 node
Splits: 16 total, 0 done (0.00%)
0:01 [0 rows, 0B] [0 rows/s, 0B/s]

Query 20200409_121548_00004_3jete failed: java.net.UnknownHostException: namenode
```

可以找到hive中表，但是查询不成功，查询不成功是因为使用`docker-compose`方式搭建的hive环境，其中namenode节点和kubernetes中的presto不在一套网络环境中，无法解析namenode地址，有条件可以用hive集群测试，本次测试环境都基于单机。



#### 4.4.2 presto查询mysql中数据

首先准备mysql环境，这里直接利用docker启动mysql环境

```shell
docker run --name mysql -d -p 3306:3306 -e MYSQL_ROOT_PASSWORD=123456 mysql:5.6.36
docker exec -it mysql /bin/bash
# 造数据
mysql -uroot -p123456
mysql> create database test;
mysql> use test;
Database changed
mysql> create table user(id int not null, username varchar(32) not null, password varchar(32) not null);
mysql> insert into user values(1,'user1','password1');
mysql> insert into user values(2,'user2','password2');
mysql> insert into user values(3,'user3','password3');
mysql> select * from user;
+----+----------+-----------+
| id | username | password  |
+----+----------+-----------+
|  1 | user1    | password1 |
|  2 | user2    | password2 |
|  3 | user3    | password3 |
+----+----------+-----------+
3 rows in set (0.00 sec)
```

mysql的连接端口为3306，确认该端口处于监听状态 `presto连接mysql进行测试`

```shell
./presto-cli-0.208-executable.jar --server localhost:30025 --catalog mysql --schema test
presto:test> show tables;
 Table
-------
 user
(1 row)

Query 20200409_122956_00004_6mm78, FINISHED, 2 nodes
Splits: 18 total, 18 done (100.00%)
0:00 [1 rows, 18B] [10 rows/s, 187B/s]

presto:test> select * from user;
 id | username | password
----+----------+-----------
  1 | user1    | password1
  2 | user2    | password2
  3 | user3    | password3
(3 rows)

Query 20200409_123010_00005_6mm78, FINISHED, 1 node
Splits: 17 total, 17 done (100.00%)
0:00 [3 rows, 0B] [10 rows/s, 0B/s]
```

`presto`可以查到mysql中的表及表数据



#### **4.4.3 presto查询cassandra中的数据**

首先准备mysql环境，这里直接利用docker启动cassandra环境

```shell
docker run --name cassandra -p 9042:9042 -d cassandra:3.0
```

连接数据库

<img src="https://i.loli.net/2020/04/10/mQgKFSxtYNh8HGT.png" style="zoom:50%;" />

准备数据

```sql
CREATE KEYSPACE IF NOT EXISTS pimin_net
WITH REPLICATION = {'class': 'SimpleStrategy','replication_factor':1};

USE pimin_net;

CREATE TABLE users (
id int,
user_name varchar,
PRIMARY KEY (id) );

INSERT INTO users (id,user_name) VALUES (1,'china');
INSERT INTO users (id,user_name) VALUES (2,'taiwan');

select * from users;

```

<img src="https://i.loli.net/2020/04/13/FmflGnxRWAsLIgB.jpg" style="zoom:50%;" />

查询数据

```shell
[root@ecs-wuhan ~]# ./presto-cli-0.208-executable.jar --server localhost:30025 --catalog cassandra --schema pimin_net
presto:pimin_net> show tables;
 Table
-------
 users
(1 row)

Query 20200410_024828_00002_4ekb9, FINISHED, 2 nodes
Splits: 18 total, 18 done (100.00%)
0:00 [1 rows, 24B] [5 rows/s, 133B/s]

presto:pimin_net> select * from users;
 id | user_name
----+-----------
  1 | china
  2 | taiwan
(2 rows)

Query 20200410_024834_00003_4ekb9, FINISHED, 1 node
Splits: 273 total, 273 done (100.00%)
0:02 [2 rows, 2B] [1 rows/s, 1B/s]

presto:pimin_net>
```

可以查询到cassandra中的数据



### 4.5 弹性伸缩

```shell
kubectl get deployment
NAME                  DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
presto-coordinator    1         1         1            1           21m
presto-worker         2         2         2            2           21m

kubectl scale deployment presto-worker --replicas=3
deployment "presto-worker" scaled

kubectl get deployment
NAME                  DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
presto-coordinator    1         1         1            1           23m
presto-worker         3         3         3            3           23m
```

## 五、Trouble Shooting

- Docker拉取镜像源time out或者拉取不上，增加docker镜像源，把163，阿里，Azure的docker加速器最好都加上

  ```shell
  vim /etc/docker/daemon.json
  
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

  

## 六、Ref

- [Using the PostgreSQL Operator with Rook Ceph Storage](https://info.crunchydata.com/blog/crunchy-postgresql-operator-with-rook-ceph-storage)
- [presto-kubernetes](https://github.com/dharmeshkakadia/presto-kubernetes)
- [kubernetes部署rook+ceph存储系统](https://blog.csdn.net/networken/article/details/85772418)
- [kubernetes上部署rook-ceph存储系统](https://blog.csdn.net/ygqygq2/article/details/103014350)
- [在Kubernetes上部署Presto](https://blog.csdn.net/chenleiking/article/details/82493798)
- [Docker镜像加速器](https://docker_practice.gitee.io/install/mirror.html)
- [Presto连接MySQL](https://www.jianshu.com/p/ba730747cc8c)
- [docker-hive](