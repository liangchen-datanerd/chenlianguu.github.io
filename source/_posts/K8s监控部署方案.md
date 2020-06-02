---
title: K8s监控部署方案
top: false
index_img: https://www.ovh.com/blog/wp-content/uploads/2019/01/kubernetesblog02.jpg
toc: true
mathjax: true
date: 2020-01-07 17:17:22
password:
summary:
tags:
 - kubernetes
 - grafana
 - prometheus
 - 部署
 - 监控
categories: kubernetes
---

## 架构
![grafana](https://i.loli.net/2020/01/08/DVBiRAwaNsnYbgL.png)

## 部署

yaml文件地址见文章底部 ⬇️

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

## Yaml文件及Dashboard配置json文件

```yaml
apiVersion: v1
kind: Namespace
metadata: 
  name: ns-monitor
  labels:
    name: ns-monitor

---

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

---
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: prometheus
rules:
  - apiGroups: [""] # "" indicates the core API group
    resources:
      - nodes
      - nodes/proxy
      - services
      - endpoints
      - pods
    verbs:
      - get
      - watch
      - list
  - apiGroups:
      - extensions
    resources:
      - ingresses
    verbs:
      - get
      - watch
      - list
  - nonResourceURLs: ["/metrics"]
    verbs:
      - get
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus
  namespace: ns-monitor
  labels:
    app: prometheus
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: prometheus
subjects:
  - kind: ServiceAccount
    name: prometheus
    namespace: ns-monitor
roleRef:
  kind: ClusterRole
  name: prometheus
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-conf
  namespace: ns-monitor
  labels:
    app: prometheus
data:
  prometheus.yml: |-
    # my global config
    global:
      scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
      evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
      # scrape_timeout is set to the global default (10s).

    # Alertmanager configuration
    alerting:
      alertmanagers:
      - static_configs:
        - targets:
          # - alertmanager:9093

    # Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
    rule_files:
      # - "first_rules.yml"
      # - "second_rules.yml"

    # A scrape configuration containing exactly one endpoint to scrape:
    # Here it's Prometheus itself.
    scrape_configs:
      # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
      - job_name: 'prometheus'

        # metrics_path defaults to '/metrics'
        # scheme defaults to 'http'.

        static_configs:
          - targets: ['localhost:9090']
      - job_name: 'grafana'
        static_configs:
          - targets:
              - 'grafana-service.ns-monitor:3000'

      - job_name: 'kubernetes-apiservers'

        kubernetes_sd_configs:
        - role: endpoints

        # Default to scraping over https. If required, just disable this or change to
        # `http`.
        scheme: https

        # This TLS & bearer token file config is used to connect to the actual scrape
        # endpoints for cluster components. This is separate to discovery auth
        # configuration because discovery & scraping are two separate concerns in
        # Prometheus. The discovery auth config is automatic if Prometheus runs inside
        # the cluster. Otherwise, more config options have to be provided within the
        # <kubernetes_sd_config>.
        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
          # If your node certificates are self-signed or use a different CA to the
          # master CA, then disable certificate verification below. Note that
          # certificate verification is an integral part of a secure infrastructure
          # so this should only be disabled in a controlled environment. You can
          # disable certificate verification by uncommenting the line below.
          #
          # insecure_skip_verify: true
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token

        # Keep only the default/kubernetes service endpoints for the https port. This
        # will add targets for each API server which Kubernetes adds an endpoint to
        # the default/kubernetes service.
        relabel_configs:
        - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
          action: keep
          regex: default;kubernetes;https

      # Scrape config for nodes (kubelet).
      #
      # Rather than connecting directly to the node, the scrape is proxied though the
      # Kubernetes apiserver.  This means it will work if Prometheus is running out of
      # cluster, or can't connect to nodes for some other reason (e.g. because of
      # firewalling).
      - job_name: 'kubernetes-nodes'

        # Default to scraping over https. If required, just disable this or change to
        # `http`.
        scheme: https

        # This TLS & bearer token file config is used to connect to the actual scrape
        # endpoints for cluster components. This is separate to discovery auth
        # configuration because discovery & scraping are two separate concerns in
        # Prometheus. The discovery auth config is automatic if Prometheus runs inside
        # the cluster. Otherwise, more config options have to be provided within the
        # <kubernetes_sd_config>.
        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token

        kubernetes_sd_configs:
        - role: node

        relabel_configs:
        - action: labelmap
          regex: __meta_kubernetes_node_label_(.+)
        - target_label: __address__
          replacement: kubernetes.default.svc:443
        - source_labels: [__meta_kubernetes_node_name]
          regex: (.+)
          target_label: __metrics_path__
          replacement: /api/v1/nodes/${1}/proxy/metrics

      # Scrape config for Kubelet cAdvisor.
      #
      # This is required for Kubernetes 1.7.3 and later, where cAdvisor metrics
      # (those whose names begin with 'container_') have been removed from the
      # Kubelet metrics endpoint.  This job scrapes the cAdvisor endpoint to
      # retrieve those metrics.
      #
      # In Kubernetes 1.7.0-1.7.2, these metrics are only exposed on the cAdvisor
      # HTTP endpoint; use "replacement: /api/v1/nodes/${1}:4194/proxy/metrics"
      # in that case (and ensure cAdvisor's HTTP server hasn't been disabled with
      # the --cadvisor-port=0 Kubelet flag).
      #
      # This job is not necessary and should be removed in Kubernetes 1.6 and
      # earlier versions, or it will cause the metrics to be scraped twice.
      - job_name: 'kubernetes-cadvisor'

        # Default to scraping over https. If required, just disable this or change to
        # `http`.
        scheme: https

        # This TLS & bearer token file config is used to connect to the actual scrape
        # endpoints for cluster components. This is separate to discovery auth
        # configuration because discovery & scraping are two separate concerns in
        # Prometheus. The discovery auth config is automatic if Prometheus runs inside
        # the cluster. Otherwise, more config options have to be provided within the
        # <kubernetes_sd_config>.
        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token

        kubernetes_sd_configs:
        - role: node

        relabel_configs:
        - action: labelmap
          regex: __meta_kubernetes_node_label_(.+)
        - target_label: __address__
          replacement: kubernetes.default.svc:443
        - source_labels: [__meta_kubernetes_node_name]
          regex: (.+)
          target_label: __metrics_path__
          replacement: /api/v1/nodes/${1}/proxy/metrics/cadvisor

      # Scrape config for service endpoints.
      #
      # The relabeling allows the actual service scrape endpoint to be configured
      # via the following annotations:
      #
      # * `prometheus.io/scrape`: Only scrape services that have a value of `true`
      # * `prometheus.io/scheme`: If the metrics endpoint is secured then you will need
      # to set this to `https` & most likely set the `tls_config` of the scrape config.
      # * `prometheus.io/path`: If the metrics path is not `/metrics` override this.
      # * `prometheus.io/port`: If the metrics are exposed on a different port to the
      # service then set this appropriately.
      - job_name: 'kubernetes-service-endpoints'

        kubernetes_sd_configs:
        - role: endpoints

        relabel_configs:
        - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scrape]
          action: keep
          regex: true
        - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scheme]
          action: replace
          target_label: __scheme__
          regex: (https?)
        - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_path]
          action: replace
          target_label: __metrics_path__
          regex: (.+)
        - source_labels: [__address__, __meta_kubernetes_service_annotation_prometheus_io_port]
          action: replace
          target_label: __address__
          regex: ([^:]+)(?::\d+)?;(\d+)
          replacement: $1:$2
        - action: labelmap
          regex: __meta_kubernetes_service_label_(.+)
        - source_labels: [__meta_kubernetes_namespace]
          action: replace
          target_label: kubernetes_namespace
        - source_labels: [__meta_kubernetes_service_name]
          action: replace
          target_label: kubernetes_name

      # Example scrape config for probing services via the Blackbox Exporter.
      #
      # The relabeling allows the actual service scrape endpoint to be configured
      # via the following annotations:
      #
      # * `prometheus.io/probe`: Only probe services that have a value of `true`
      - job_name: 'kubernetes-services'

        metrics_path: /probe
        params:
          module: [http_2xx]

        kubernetes_sd_configs:
        - role: service

        relabel_configs:
        - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_probe]
          action: keep
          regex: true
        - source_labels: [__address__]
          target_label: __param_target
        - target_label: __address__
          replacement: blackbox-exporter.example.com:9115
        - source_labels: [__param_target]
          target_label: instance
        - action: labelmap
          regex: __meta_kubernetes_service_label_(.+)
        - source_labels: [__meta_kubernetes_namespace]
          target_label: kubernetes_namespace
        - source_labels: [__meta_kubernetes_service_name]
          target_label: kubernetes_name

      # Example scrape config for probing ingresses via the Blackbox Exporter.
      #
      # The relabeling allows the actual ingress scrape endpoint to be configured
      # via the following annotations:
      #
      # * `prometheus.io/probe`: Only probe services that have a value of `true`
      - job_name: 'kubernetes-ingresses'

        metrics_path: /probe
        params:
          module: [http_2xx]

        kubernetes_sd_configs:
          - role: ingress

        relabel_configs:
          - source_labels: [__meta_kubernetes_ingress_annotation_prometheus_io_probe]
            action: keep
            regex: true
          - source_labels: [__meta_kubernetes_ingress_scheme,__address__,__meta_kubernetes_ingress_path]
            regex: (.+);(.+);(.+)
            replacement: ${1}://${2}${3}
            target_label: __param_target
          - target_label: __address__
            replacement: blackbox-exporter.example.com:9115
          - source_labels: [__param_target]
            target_label: instance
          - action: labelmap
            regex: __meta_kubernetes_ingress_label_(.+)
          - source_labels: [__meta_kubernetes_namespace]
            target_label: kubernetes_namespace
          - source_labels: [__meta_kubernetes_ingress_name]
            target_label: kubernetes_name

      # Example scrape config for pods
      #
      # The relabeling allows the actual pod scrape endpoint to be configured via the
      # following annotations:
      #
      # * `prometheus.io/scrape`: Only scrape pods that have a value of `true`
      # * `prometheus.io/path`: If the metrics path is not `/metrics` override this.
      # * `prometheus.io/port`: Scrape the pod on the indicated port instead of the
      # pod's declared ports (default is a port-free target if none are declared).
      - job_name: 'kubernetes-pods'

        kubernetes_sd_configs:
        - role: pod

        relabel_configs:
        - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
          action: keep
          regex: true
        - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
          action: replace
          target_label: __metrics_path__
          regex: (.+)
        - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
          action: replace
          regex: ([^:]+)(?::\d+)?;(\d+)
          replacement: $1:$2
          target_label: __address__
        - action: labelmap
          regex: __meta_kubernetes_pod_label_(.+)
        - source_labels: [__meta_kubernetes_namespace]
          action: replace
          target_label: kubernetes_namespace
        - source_labels: [__meta_kubernetes_pod_name]
          action: replace
          target_label: kubernetes_pod_name
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-rules
  namespace: ns-monitor
  labels:
    app: prometheus
data:
  cpu-usage.rule: |
    groups:
      - name: NodeCPUUsage
        rules:
          - alert: NodeCPUUsage
            expr: (100 - (avg by (instance) (irate(node_cpu{name="node-exporter",mode="idle"}[5m])) * 100)) > 75
            for: 2m
            labels:
              severity: "page"
            annotations:
              summary: "{{$labels.instance}}: High CPU usage detected"
              description: "{{$labels.instance}}: CPU usage is above 75% (current value is: {{ $value }})"
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: "prometheus-data-pv"
  labels:
    name: prometheus-data-pv
    release: stable
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  nfs:
    path: /nfs
    server: 52.83.79.244

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: prometheus-data-pvc
  namespace: ns-monitor
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  selector:
    matchLabels:
      name: prometheus-data-pv
      release: stable

---
kind: Deployment
apiVersion: apps/v1beta2
metadata:
  labels:
    app: prometheus
  name: prometheus
  namespace: ns-monitor
spec:
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: prometheus
  template:
    metadata:
      labels:
        app: prometheus
    spec:
      serviceAccountName: prometheus
      securityContext:
        runAsUser: 0
      containers:
        - name: prometheus
          image: 52.83.79.244:8093/pythagoras/prometheus:latest
          imagePullPolicy: IfNotPresent
          volumeMounts:
            - mountPath: /prometheus
              name: prometheus-data-volume
            - mountPath: /etc/prometheus/prometheus.yml
              name: prometheus-conf-volume
              subPath: prometheus.yml
            - mountPath: /etc/prometheus/rules
              name: prometheus-rules-volume
          ports:
            - containerPort: 9090
              protocol: TCP
      volumes:
        - name: prometheus-data-volume
          persistentVolumeClaim:
            claimName: prometheus-data-pvc
        - name: prometheus-conf-volume
          configMap:
            name: prometheus-conf
        - name: prometheus-rules-volume
          configMap:
            name: prometheus-rules
      tolerations:
        - key: node-role.kubernetes.io/master
          effect: NoSchedule

---
kind: Service
apiVersion: v1
metadata:
  annotations:
    prometheus.io/scrape: 'true'
  labels:
    app: prometheus
  name: prometheus-service
  namespace: ns-monitor
spec:
  ports:
    - port: 9090
      targetPort: 9090
  selector:
    app: prometheus
  type: NodePort
---

apiVersion: v1
kind: PersistentVolume
metadata:
  name: "grafana-data-pv"
  labels:
    name: grafana-data-pv
    release: stable
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  nfs:
    path: /nfs/grafana
    server: 52.83.79.244
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: grafana-data-pvc
  namespace: ns-monitor
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  selector:
    matchLabels:
      name: grafana-data-pv
      release: stable
---
kind: Deployment
apiVersion: apps/v1beta2
metadata:
  labels:
    app: grafana
  name: grafana
  namespace: ns-monitor
spec:
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: grafana
  template:
    metadata:
      labels:
        app: grafana
    spec:
      securityContext:
        runAsUser: 0
      containers:
        - name: grafana
          image: 52.83.79.244:8093/pythagoras/grafana:latest
          imagePullPolicy: IfNotPresent
          env:
            - name: GF_AUTH_BASIC_ENABLED
              value: "true"
            - name: GF_AUTH_ANONYMOUS_ENABLED
              value: "false"
          readinessProbe:
            httpGet:
              path: /login
              port: 3000
          volumeMounts:
            - mountPath: /var/lib/grafana
              name: grafana-data-volume
          ports:
            - containerPort: 3000
              protocol: TCP
      volumes:
        - name: grafana-data-volume
          persistentVolumeClaim:
            claimName: grafana-data-pvc
---
kind: Service
apiVersion: v1
metadata:
  labels:
    app: grafana
  name: grafana-service
  namespace: ns-monitor
spec:
  ports:
    - port: 3000
      targetPort: 3000
  selector:
    app: grafana
  type: NodePort

```

> dashboard配置

```json
{
  "__inputs": [
    {
      "name": "DS_PROMETHEUS",
      "label": "prometheus",
      "description": "",
      "type": "datasource",
      "pluginId": "prometheus",
      "pluginName": "Prometheus"
    }
  ],
  "__requires": [
    {
      "type": "grafana",
      "id": "grafana",
      "name": "Grafana",
      "version": "5.0.4"
    },
    {
      "type": "panel",
      "id": "graph",
      "name": "Graph",
      "version": "5.0.0"
    },
    {
      "type": "datasource",
      "id": "prometheus",
      "name": "Prometheus",
      "version": "5.0.0"
    },
    {
      "type": "panel",
      "id": "singlestat",
      "name": "Singlestat",
      "version": "5.0.0"
    }
  ],
  "annotations": {
    "list": [
      {
        "builtIn": 1,
        "datasource": "-- Grafana --",
        "enable": true,
        "hide": true,
        "iconColor": "rgba(0, 211, 255, 1)",
        "name": "Annotations & Alerts",
        "type": "dashboard"
      }
    ]
  },
  "description": "Shows resource usage of Kubernetes pods.",
  "editable": true,
  "gnetId": 737,
  "graphTooltip": 0,
  "id": null,
  "iteration": 1524106098941,
  "links": [],
  "panels": [
    {
      "collapsed": false,
      "gridPos": {
        "h": 1,
        "w": 24,
        "x": 0,
        "y": 0
      },
      "id": 35,
      "panels": [],
      "title": "all pods",
      "type": "row"
    },
    {
      "cacheTimeout": null,
      "colorBackground": false,
      "colorValue": true,
      "colors": [
        "rgba(50, 172, 45, 0.97)",
        "rgba(237, 129, 40, 0.89)",
        "rgba(245, 54, 54, 0.9)"
      ],
      "datasource": "${DS_PROMETHEUS}",
      "editable": true,
      "error": false,
      "format": "percent",
      "gauge": {
        "maxValue": 100,
        "minValue": 0,
        "show": true,
        "thresholdLabels": false,
        "thresholdMarkers": true
      },
      "gridPos": {
        "h": 5,
        "w": 8,
        "x": 0,
        "y": 1
      },
      "height": "180px",
      "id": 4,
      "interval": null,
      "isNew": true,
      "links": [],
      "mappingType": 1,
      "mappingTypes": [
        {
          "name": "value to text",
          "value": 1
        },
        {
          "name": "range to text",
          "value": 2
        }
      ],
      "maxDataPoints": 100,
      "nullPointMode": "connected",
      "nullText": null,
      "postfix": "",
      "postfixFontSize": "50%",
      "prefix": "",
      "prefixFontSize": "50%",
      "rangeMaps": [
        {
          "from": "null",
          "text": "N/A",
          "to": "null"
        }
      ],
      "sparkline": {
        "fillColor": "rgba(31, 118, 189, 0.18)",
        "full": false,
        "lineColor": "rgb(31, 120, 193)",
        "show": false
      },
      "tableColumn": "",
      "targets": [
        {
          "expr": "sum (container_memory_working_set_bytes{id=\"/\",instance=~\"^$instance$\"}) / sum (machine_memory_bytes{instance=~\"^$instance$\"}) * 100",
          "interval": "",
          "intervalFactor": 2,
          "legendFormat": "",
          "refId": "A",
          "step": 2
        }
      ],
      "thresholds": "65, 90",
      "timeFrom": "1m",
      "timeShift": null,
      "title": "Memory Working Set",
      "transparent": false,
      "type": "singlestat",
      "valueFontSize": "80%",
      "valueMaps": [
        {
          "op": "=",
          "text": "N/A",
          "value": "null"
        }
      ],
      "valueName": "current"
    },
    {
      "cacheTimeout": null,
      "colorBackground": false,
      "colorValue": true,
      "colors": [
        "rgba(50, 172, 45, 0.97)",
        "rgba(237, 129, 40, 0.89)",
        "rgba(245, 54, 54, 0.9)"
      ],
      "datasource": "${DS_PROMETHEUS}",
      "decimals": 2,
      "editable": true,
      "error": false,
      "format": "percent",
      "gauge": {
        "maxValue": 100,
        "minValue": 0,
        "show": true,
        "thresholdLabels": false,
        "thresholdMarkers": true
      },
      "gridPos": {
        "h": 5,
        "w": 8,
        "x": 8,
        "y": 1
      },
      "height": "180px",
      "id": 6,
      "interval": null,
      "isNew": true,
      "links": [],
      "mappingType": 1,
      "mappingTypes": [
        {
          "name": "value to text",
          "value": 1
        },
        {
          "name": "range to text",
          "value": 2
        }
      ],
      "maxDataPoints": 100,
      "nullPointMode": "connected",
      "nullText": null,
      "postfix": "",
      "postfixFontSize": "50%",
      "prefix": "",
      "prefixFontSize": "50%",
      "rangeMaps": [
        {
          "from": "null",
          "text": "N/A",
          "to": "null"
        }
      ],
      "sparkline": {
        "fillColor": "rgba(31, 118, 189, 0.18)",
        "full": false,
        "lineColor": "rgb(31, 120, 193)",
        "show": false
      },
      "tableColumn": "",
      "targets": [
        {
          "expr": "sum(rate(container_cpu_usage_seconds_total{id=\"/\",instance=~\"^$instance$\"}[1m])) / sum (machine_cpu_cores{instance=~\"^$instance$\"}) * 100",
          "interval": "10s",
          "intervalFactor": 1,
          "refId": "A",
          "step": 10
        }
      ],
      "thresholds": "65, 90",
      "timeFrom": "1m",
      "timeShift": null,
      "title": "Cpu Usage",
      "type": "singlestat",
      "valueFontSize": "80%",
      "valueMaps": [
        {
          "op": "=",
          "text": "N/A",
          "value": "null"
        }
      ],
      "valueName": "current"
    },
    {
      "cacheTimeout": null,
      "colorBackground": false,
      "colorValue": true,
      "colors": [
        "rgba(50, 172, 45, 0.97)",
        "rgba(237, 129, 40, 0.89)",
        "rgba(245, 54, 54, 0.9)"
      ],
      "datasource": "${DS_PROMETHEUS}",
      "decimals": 2,
      "editable": true,
      "error": false,
      "format": "percent",
      "gauge": {
        "maxValue": 100,
        "minValue": 0,
        "show": true,
        "thresholdLabels": false,
        "thresholdMarkers": true
      },
      "gridPos": {
        "h": 5,
        "w": 8,
        "x": 16,
        "y": 1
      },
      "height": "180px",
      "id": 7,
      "interval": null,
      "isNew": true,
      "links": [],
      "mappingType": 1,
      "mappingTypes": [
        {
          "name": "value to text",
          "value": 1
        },
        {
          "name": "range to text",
          "value": 2
        }
      ],
      "maxDataPoints": 100,
      "nullPointMode": "connected",
      "nullText": null,
      "postfix": "",
      "postfixFontSize": "50%",
      "prefix": "",
      "prefixFontSize": "50%",
      "rangeMaps": [
        {
          "from": "null",
          "text": "N/A",
          "to": "null"
        }
      ],
      "sparkline": {
        "fillColor": "rgba(31, 118, 189, 0.18)",
        "full": false,
        "lineColor": "rgb(31, 120, 193)",
        "show": false
      },
      "tableColumn": "",
      "targets": [
        {
          "expr": "sum(container_fs_usage_bytes{id=\"/\",instance=~\"^$instance$\"}) / sum(container_fs_limit_bytes{id=\"/\",instance=~\"^$instance$\"}) * 100",
          "interval": "10s",
          "intervalFactor": 1,
          "legendFormat": "",
          "metric": "",
          "refId": "A",
          "step": 10
        }
      ],
      "thresholds": "65, 90",
      "timeFrom": "1m",
      "timeShift": null,
      "title": "Filesystem Usage",
      "type": "singlestat",
      "valueFontSize": "80%",
      "valueMaps": [
        {
          "op": "=",
          "text": "N/A",
          "value": "null"
        }
      ],
      "valueName": "current"
    },
    {
      "cacheTimeout": null,
      "colorBackground": false,
      "colorValue": false,
      "colors": [
        "rgba(50, 172, 45, 0.97)",
        "rgba(237, 129, 40, 0.89)",
        "rgba(245, 54, 54, 0.9)"
      ],
      "datasource": "${DS_PROMETHEUS}",
      "decimals": 2,
      "editable": true,
      "error": false,
      "format": "bytes",
      "gauge": {
        "maxValue": 100,
        "minValue": 0,
        "show": false,
        "thresholdLabels": false,
        "thresholdMarkers": true
      },
      "gridPos": {
        "h": 3,
        "w": 4,
        "x": 0,
        "y": 6
      },
      "height": "1px",
      "hideTimeOverride": true,
      "id": 9,
      "interval": null,
      "isNew": true,
      "links": [],
      "mappingType": 1,
      "mappingTypes": [
        {
          "name": "value to text",
          "value": 1
        },
        {
          "name": "range to text",
          "value": 2
        }
      ],
      "maxDataPoints": 100,
      "nullPointMode": "connected",
      "nullText": null,
      "postfix": "",
      "postfixFontSize": "20%",
      "prefix": "",
      "prefixFontSize": "20%",
      "rangeMaps": [
        {
          "from": "null",
          "text": "N/A",
          "to": "null"
        }
      ],
      "sparkline": {
        "fillColor": "rgba(31, 118, 189, 0.18)",
        "full": false,
        "lineColor": "rgb(31, 120, 193)",
        "show": false
      },
      "tableColumn": "",
      "targets": [
        {
          "expr": "sum(container_memory_working_set_bytes{id=\"/\",instance=~\"^$instance$\"})",
          "interval": "10s",
          "intervalFactor": 1,
          "refId": "A",
          "step": 10
        }
      ],
      "thresholds": "",
      "timeFrom": "1m",
      "title": "Used",
      "type": "singlestat",
      "valueFontSize": "50%",
      "valueMaps": [
        {
          "op": "=",
          "text": "N/A",
          "value": "null"
        }
      ],
      "valueName": "current"
    },
    {
      "cacheTimeout": null,
      "colorBackground": false,
      "colorValue": false,
      "colors": [
        "rgba(50, 172, 45, 0.97)",
        "rgba(237, 129, 40, 0.89)",
        "rgba(245, 54, 54, 0.9)"
      ],
      "datasource": "${DS_PROMETHEUS}",
      "decimals": 2,
      "editable": true,
      "error": false,
      "format": "bytes",
      "gauge": {
        "maxValue": 100,
        "minValue": 0,
        "show": false,
        "thresholdLabels": false,
        "thresholdMarkers": true
      },
      "gridPos": {
        "h": 3,
        "w": 4,
        "x": 4,
        "y": 6
      },
      "height": "1px",
      "hideTimeOverride": true,
      "id": 10,
      "interval": null,
      "isNew": true,
      "links": [],
      "mappingType": 1,
      "mappingTypes": [
        {
          "name": "value to text",
          "value": 1
        },
        {
          "name": "range to text",
          "value": 2
        }
      ],
      "maxDataPoints": 100,
      "nullPointMode": "connected",
      "nullText": null,
      "postfix": "",
      "postfixFontSize": "50%",
      "prefix": "",
      "prefixFontSize": "50%",
      "rangeMaps": [
        {
          "from": "null",
          "text": "N/A",
          "to": "null"
        }
      ],
      "sparkline": {
        "fillColor": "rgba(31, 118, 189, 0.18)",
        "full": false,
        "lineColor": "rgb(31, 120, 193)",
        "show": false
      },
      "tableColumn": "",
      "targets": [
        {
          "expr": "sum (machine_memory_bytes{instance=~\"^$instance$\"})",
          "interval": "10s",
          "intervalFactor": 1,
          "refId": "A",
          "step": 10
        }
      ],
      "thresholds": "",
      "timeFrom": "1m",
      "title": "Total",
      "type": "singlestat",
      "valueFontSize": "50%",
      "valueMaps": [
        {
          "op": "=",
          "text": "N/A",
          "value": "null"
        }
      ],
      "valueName": "current"
    },
    {
      "cacheTimeout": null,
      "colorBackground": false,
      "colorValue": false,
      "colors": [
        "rgba(50, 172, 45, 0.97)",
        "rgba(237, 129, 40, 0.89)",
        "rgba(245, 54, 54, 0.9)"
      ],
      "datasource": "${DS_PROMETHEUS}",
      "decimals": 2,
      "editable": true,
      "error": false,
      "format": "none",
      "gauge": {
        "maxValue": 100,
        "minValue": 0,
        "show": false,
        "thresholdLabels": false,
        "thresholdMarkers": true
      },
      "gridPos": {
        "h": 3,
        "w": 4,
        "x": 8,
        "y": 6
      },
      "height": "1px",
      "hideTimeOverride": true,
      "id": 11,
      "interval": null,
      "isNew": true,
      "links": [],
      "mappingType": 1,
      "mappingTypes": [
        {
          "name": "value to text",
          "value": 1
        },
        {
          "name": "range to text",
          "value": 2
        }
      ],
      "maxDataPoints": 100,
      "nullPointMode": "connected",
      "nullText": null,
      "postfix": " cores",
      "postfixFontSize": "30%",
      "prefix": "",
      "prefixFontSize": "50%",
      "rangeMaps": [
        {
          "from": "null",
          "text": "N/A",
          "to": "null"
        }
      ],
      "sparkline": {
        "fillColor": "rgba(31, 118, 189, 0.18)",
        "full": false,
        "lineColor": "rgb(31, 120, 193)",
        "show": false
      },
      "tableColumn": "",
      "targets": [
        {
          "expr": "sum (rate (container_cpu_usage_seconds_total{id=\"/\",instance=~\"^$instance$\"}[1m]))",
          "interval": "10s",
          "intervalFactor": 1,
          "refId": "A",
          "step": 10
        }
      ],
      "thresholds": "",
      "timeFrom": "1m",
      "timeShift": null,
      "title": "Used",
      "type": "singlestat",
      "valueFontSize": "50%",
      "valueMaps": [
        {
          "op": "=",
          "text": "N/A",
          "value": "null"
        }
      ],
      "valueName": "current"
    },
    {
      "cacheTimeout": null,
      "colorBackground": false,
      "colorValue": false,
      "colors": [
        "rgba(50, 172, 45, 0.97)",
        "rgba(237, 129, 40, 0.89)",
        "rgba(245, 54, 54, 0.9)"
      ],
      "datasource": "${DS_PROMETHEUS}",
      "decimals": 2,
      "editable": true,
      "error": false,
      "format": "none",
      "gauge": {
        "maxValue": 100,
        "minValue": 0,
        "show": false,
        "thresholdLabels": false,
        "thresholdMarkers": true
      },
      "gridPos": {
        "h": 3,
        "w": 4,
        "x": 12,
        "y": 6
      },
      "height": "1px",
      "hideTimeOverride": true,
      "id": 12,
      "interval": null,
      "isNew": true,
      "links": [],
      "mappingType": 1,
      "mappingTypes": [
        {
          "name": "value to text",
          "value": 1
        },
        {
          "name": "range to text",
          "value": 2
        }
      ],
      "maxDataPoints": 100,
      "nullPointMode": "connected",
      "nullText": null,
      "postfix": " cores",
      "postfixFontSize": "30%",
      "prefix": "",
      "prefixFontSize": "50%",
      "rangeMaps": [
        {
          "from": "null",
          "text": "N/A",
          "to": "null"
        }
      ],
      "sparkline": {
        "fillColor": "rgba(31, 118, 189, 0.18)",
        "full": false,
        "lineColor": "rgb(31, 120, 193)",
        "show": false
      },
      "tableColumn": "",
      "targets": [
        {
          "expr": "sum (machine_cpu_cores{instance=~\"^$instance$\"})",
          "interval": "10s",
          "intervalFactor": 1,
          "refId": "A",
          "step": 10
        }
      ],
      "thresholds": "",
      "timeFrom": "1m",
      "title": "Total",
      "type": "singlestat",
      "valueFontSize": "50%",
      "valueMaps": [
        {
          "op": "=",
          "text": "N/A",
          "value": "null"
        }
      ],
      "valueName": "current"
    },
    {
      "cacheTimeout": null,
      "colorBackground": false,
      "colorValue": false,
      "colors": [
        "rgba(50, 172, 45, 0.97)",
        "rgba(237, 129, 40, 0.89)",
        "rgba(245, 54, 54, 0.9)"
      ],
      "datasource": "${DS_PROMETHEUS}",
      "decimals": 2,
      "editable": true,
      "error": false,
      "format": "bytes",
      "gauge": {
        "maxValue": 100,
        "minValue": 0,
        "show": false,
        "thresholdLabels": false,
        "thresholdMarkers": true
      },
      "gridPos": {
        "h": 3,
        "w": 4,
        "x": 16,
        "y": 6
      },
      "height": "1px",
      "hideTimeOverride": true,
      "id": 13,
      "interval": null,
      "isNew": true,
      "links": [],
      "mappingType": 1,
      "mappingTypes": [
        {
          "name": "value to text",
          "value": 1
        },
        {
          "name": "range to text",
          "value": 2
        }
      ],
      "maxDataPoints": 100,
      "nullPointMode": "connected",
      "nullText": null,
      "postfix": "",
      "postfixFontSize": "50%",
      "prefix": "",
      "prefixFontSize": "50%",
      "rangeMaps": [
        {
          "from": "null",
          "text": "N/A",
          "to": "null"
        }
      ],
      "sparkline": {
        "fillColor": "rgba(31, 118, 189, 0.18)",
        "full": false,
        "lineColor": "rgb(31, 120, 193)",
        "show": false
      },
      "tableColumn": "",
      "targets": [
        {
          "expr": "sum(container_fs_usage_bytes{id=\"/\",instance=~\"^$instance$\"})",
          "interval": "10s",
          "intervalFactor": 1,
          "refId": "A",
          "step": 10
        }
      ],
      "thresholds": "",
      "timeFrom": "1m",
      "title": "Used",
      "type": "singlestat",
      "valueFontSize": "50%",
      "valueMaps": [
        {
          "op": "=",
          "text": "N/A",
          "value": "null"
        }
      ],
      "valueName": "current"
    },
    {
      "cacheTimeout": null,
      "colorBackground": false,
      "colorValue": false,
      "colors": [
        "rgba(50, 172, 45, 0.97)",
        "rgba(237, 129, 40, 0.89)",
        "rgba(245, 54, 54, 0.9)"
      ],
      "datasource": "${DS_PROMETHEUS}",
      "decimals": 2,
      "editable": true,
      "error": false,
      "format": "bytes",
      "gauge": {
        "maxValue": 100,
        "minValue": 0,
        "show": false,
        "thresholdLabels": false,
        "thresholdMarkers": true
      },
      "gridPos": {
        "h": 3,
        "w": 4,
        "x": 20,
        "y": 6
      },
      "height": "1px",
      "hideTimeOverride": true,
      "id": 14,
      "interval": null,
      "isNew": true,
      "links": [],
      "mappingType": 1,
      "mappingTypes": [
        {
          "name": "value to text",
          "value": 1
        },
        {
          "name": "range to text",
          "value": 2
        }
      ],
      "maxDataPoints": 100,
      "nullPointMode": "connected",
      "nullText": null,
      "postfix": "",
      "postfixFontSize": "50%",
      "prefix": "",
      "prefixFontSize": "50%",
      "rangeMaps": [
        {
          "from": "null",
          "text": "N/A",
          "to": "null"
        }
      ],
      "sparkline": {
        "fillColor": "rgba(31, 118, 189, 0.18)",
        "full": false,
        "lineColor": "rgb(31, 120, 193)",
        "show": false
      },
      "tableColumn": "",
      "targets": [
        {
          "expr": "sum (container_fs_limit_bytes{id=\"/\",instance=~\"^$instance$\"})",
          "interval": "10s",
          "intervalFactor": 1,
          "refId": "A",
          "step": 10
        }
      ],
      "thresholds": "",
      "timeFrom": "1m",
      "title": "Total",
      "type": "singlestat",
      "valueFontSize": "50%",
      "valueMaps": [
        {
          "op": "=",
          "text": "N/A",
          "value": "null"
        }
      ],
      "valueName": "current"
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": "${DS_PROMETHEUS}",
      "decimals": 2,
      "editable": true,
      "error": false,
      "fill": 1,
      "grid": {},
      "gridPos": {
        "h": 5,
        "w": 24,
        "x": 0,
        "y": 9
      },
      "height": "200px",
      "id": 32,
      "isNew": true,
      "legend": {
        "alignAsTable": true,
        "avg": true,
        "current": true,
        "max": false,
        "min": false,
        "rightSide": true,
        "show": true,
        "sideWidth": 200,
        "sort": "current",
        "sortDesc": true,
        "total": false,
        "values": true
      },
      "lines": true,
      "linewidth": 2,
      "links": [],
      "nullPointMode": "connected",
      "percentage": false,
      "pointradius": 5,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "expr": "sum(rate(container_network_receive_bytes_total{instance=~\"^$instance$\",namespace=~\"^$namespace$\"}[1m]))",
          "interval": "",
          "intervalFactor": 2,
          "legendFormat": "receive",
          "metric": "network",
          "refId": "A",
          "step": 240
        },
        {
          "expr": "- sum(rate(container_network_transmit_bytes_total{instance=~\"^$instance$\",namespace=~\"^$namespace$\"}[1m]))",
          "interval": "",
          "intervalFactor": 2,
          "legendFormat": "transmit",
          "metric": "network",
          "refId": "B",
          "step": 240
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeShift": null,
      "title": "Network",
      "tooltip": {
        "msResolution": false,
        "shared": true,
        "sort": 0,
        "value_type": "cumulative"
      },
      "transparent": false,
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "Bps",
          "label": "transmit / receive",
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        },
        {
          "format": "Bps",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": false
        }
      ]
    },
    {
      "collapsed": false,
      "gridPos": {
        "h": 1,
        "w": 24,
        "x": 0,
        "y": 14
      },
      "id": 36,
      "panels": [],
      "title": "each pod",
      "type": "row"
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": "${DS_PROMETHEUS}",
      "decimals": 3,
      "editable": true,
      "error": false,
      "fill": 0,
      "grid": {},
      "gridPos": {
        "h": 7,
        "w": 24,
        "x": 0,
        "y": 15
      },
      "height": "",
      "id": 17,
      "isNew": true,
      "legend": {
        "alignAsTable": true,
        "avg": true,
        "current": true,
        "hideEmpty": true,
        "hideZero": true,
        "max": false,
        "min": false,
        "rightSide": true,
        "show": true,
        "sideWidth": null,
        "sort": "current",
        "sortDesc": true,
        "total": false,
        "values": true
      },
      "lines": true,
      "linewidth": 2,
      "links": [],
      "nullPointMode": "connected",
      "percentage": false,
      "pointradius": 5,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "expr": "sum(rate(container_cpu_usage_seconds_total{image!=\"\",name=~\"^k8s_.*\",instance=~\"^$instance$\",namespace=~\"^$namespace$\"}[1m])) by (pod_name)",
          "interval": "",
          "intervalFactor": 2,
          "legendFormat": "{{ pod_name }}",
          "metric": "container_cpu",
          "refId": "A",
          "step": 240
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeShift": null,
      "title": "Cpu Usage",
      "tooltip": {
        "msResolution": true,
        "shared": false,
        "sort": 2,
        "value_type": "cumulative"
      },
      "transparent": false,
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "none",
          "label": "cores",
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        },
        {
          "format": "short",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": false
        }
      ]
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": "${DS_PROMETHEUS}",
      "decimals": 2,
      "editable": true,
      "error": false,
      "fill": 0,
      "grid": {},
      "gridPos": {
        "h": 7,
        "w": 24,
        "x": 0,
        "y": 22
      },
      "id": 33,
      "isNew": true,
      "legend": {
        "alignAsTable": true,
        "avg": true,
        "current": true,
        "hideEmpty": true,
        "hideZero": true,
        "max": false,
        "min": false,
        "rightSide": true,
        "show": true,
        "sideWidth": null,
        "sort": "current",
        "sortDesc": true,
        "total": false,
        "values": true
      },
      "lines": true,
      "linewidth": 2,
      "links": [],
      "nullPointMode": "null",
      "percentage": false,
      "pointradius": 5,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "expr": "sum (container_memory_working_set_bytes{image!=\"\",name=~\"^k8s_.*\",instance=~\"^$instance$\",namespace=~\"^$namespace$\"}) by (pod_name)",
          "interval": "",
          "intervalFactor": 2,
          "legendFormat": "{{ pod_name }}",
          "metric": "",
          "refId": "A",
          "step": 240
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeShift": null,
      "title": "Memory Working Set",
      "tooltip": {
        "msResolution": false,
        "shared": false,
        "sort": 2,
        "value_type": "cumulative"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "bytes",
          "label": "used",
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        },
        {
          "format": "short",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": false
        }
      ]
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": "${DS_PROMETHEUS}",
      "decimals": 2,
      "editable": true,
      "error": false,
      "fill": 1,
      "grid": {},
      "gridPos": {
        "h": 7,
        "w": 24,
        "x": 0,
        "y": 29
      },
      "id": 16,
      "isNew": true,
      "legend": {
        "alignAsTable": true,
        "avg": true,
        "current": true,
        "hideEmpty": true,
        "hideZero": true,
        "max": false,
        "min": false,
        "rightSide": true,
        "show": true,
        "sideWidth": 200,
        "sort": "avg",
        "sortDesc": true,
        "total": false,
        "values": true
      },
      "lines": true,
      "linewidth": 2,
      "links": [],
      "nullPointMode": "null",
      "percentage": false,
      "pointradius": 5,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "expr": "sum (rate (container_network_receive_bytes_total{image!=\"\",name=~\"^k8s_.*\",instance=~\"^$instance$\",namespace=~\"^$namespace$\"}[1m])) by (pod_name)",
          "interval": "",
          "intervalFactor": 2,
          "legendFormat": "{{ pod_name }} < in",
          "metric": "network",
          "refId": "A",
          "step": 240
        },
        {
          "expr": "- sum (rate (container_network_transmit_bytes_total{image!=\"\",name=~\"^k8s_.*\",instance=~\"^$instance$\",namespace=~\"^$namespace$\"}[1m])) by (pod_name)",
          "interval": "",
          "intervalFactor": 2,
          "legendFormat": "{{ pod_name }} > out",
          "metric": "network",
          "refId": "B",
          "step": 240
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeShift": null,
      "title": "Network",
      "tooltip": {
        "msResolution": false,
        "shared": false,
        "sort": 2,
        "value_type": "cumulative"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "Bps",
          "label": "transmit / receive",
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        },
        {
          "format": "short",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": false
        }
      ]
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": "${DS_PROMETHEUS}",
      "decimals": 2,
      "editable": true,
      "error": false,
      "fill": 1,
      "grid": {},
      "gridPos": {
        "h": 7,
        "w": 24,
        "x": 0,
        "y": 36
      },
      "id": 34,
      "isNew": true,
      "legend": {
        "alignAsTable": true,
        "avg": true,
        "current": true,
        "hideEmpty": true,
        "hideZero": true,
        "max": false,
        "min": false,
        "rightSide": true,
        "show": true,
        "sideWidth": 200,
        "sort": "current",
        "sortDesc": true,
        "total": false,
        "values": true
      },
      "lines": true,
      "linewidth": 2,
      "links": [],
      "nullPointMode": "null",
      "percentage": false,
      "pointradius": 5,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "expr": "sum(container_fs_usage_bytes{image!=\"\",name=~\"^k8s_.*\",instance=~\"^$instance$\",namespace=~\"^$namespace$\"}) by (pod_name)",
          "interval": "",
          "intervalFactor": 2,
          "legendFormat": "{{ pod_name }}",
          "metric": "network",
          "refId": "A",
          "step": 240
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeShift": null,
      "title": "Filesystem",
      "tooltip": {
        "msResolution": false,
        "shared": false,
        "sort": 2,
        "value_type": "cumulative"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "bytes",
          "label": "used",
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        },
        {
          "format": "short",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": false
        }
      ]
    }
  ],
  "refresh": false,
  "schemaVersion": 16,
  "style": "dark",
  "tags": [
    "kubernetes"
  ],
  "templating": {
    "list": [
      {
        "allValue": ".*",
        "current": {},
        "datasource": "${DS_PROMETHEUS}",
        "hide": 0,
        "includeAll": true,
        "label": "Instance",
        "multi": false,
        "name": "instance",
        "options": [],
        "query": "label_values(instance)",
        "refresh": 1,
        "regex": "master|node.*",
        "sort": 0,
        "tagValuesQuery": "",
        "tags": [],
        "tagsQuery": "",
        "type": "query",
        "useTags": false
      },
      {
        "allValue": null,
        "current": {},
        "datasource": "${DS_PROMETHEUS}",
        "hide": 0,
        "includeAll": true,
        "label": "Namespace",
        "multi": true,
        "name": "namespace",
        "options": [],
        "query": "label_values(namespace)",
        "refresh": 1,
        "regex": "",
        "sort": 0,
        "tagValuesQuery": "",
        "tags": [],
        "tagsQuery": "",
        "type": "query",
        "useTags": false
      }
    ]
  },
  "time": {
    "from": "now-15m",
    "to": "now"
  },
  "timepicker": {
    "refresh_intervals": [
      "5s",
      "10s",
      "30s",
      "1m",
      "5m",
      "15m",
      "30m",
      "1h",
      "2h",
      "1d"
    ],
    "time_options": [
      "5m",
      "15m",
      "1h",
      "6h",
      "12h",
      "24h",
      "2d",
      "7d",
      "30d"
    ]
  },
  "timezone": "browser",
  "title": "Kubernetes Pod Resources",
  "uid": "Tl8II5Wmz",
  "version": 19
}
```

