# 
- [](#)
- [部署Prometheus\&Grafnaa](#部署prometheusgrafnaa)
      - [1.1 创建账号\&证书](#11-创建账号证书)
      - [1.2 部署 prometheus](#12-部署-prometheus)
      - [1.3 部署Grafana](#13-部署grafana)
- [监控项 Exporter](#监控项-exporter)
  - [kube-state-metrics](#kube-state-metrics)
      - [1.1 集群版本对应](#11-集群版本对应)
      - [1.2 安装KMS](#12-安装kms)
      - [1.3 抓取 KSM](#13-抓取-ksm)
      - [1.4 数据可视化](#14-数据可视化)
  - [Node Exporter](#node-exporter)
      - [1.1 安装 Node Exporter](#11-安装-node-exporter)
      - [1.2 Node Exporter metrics](#12-node-exporter-metrics)
      - [1.3 抓取Node Exporter](#13-抓取node-exporter)
      - [1.4 数据可视化](#14-数据可视化-1)
      - [1.5 systemctl封装](#15-systemctl封装)
  - [Elasticsearch Exporter](#elasticsearch-exporter)
      - [1.1 安装Elasticsearch Exporter](#11-安装elasticsearch-exporter)
      - [1.2 elasticsearch exporter metrics](#12-elasticsearch-exporter-metrics)
      - [1.3 添加Prometheus配置](#13-添加prometheus配置)
      - [1.4 数据可视化](#14-数据可视化-2)
  - [dcgm-exporter](#dcgm-exporter)
      - [1.1 NVIDIA DCGM](#11-nvidia-dcgm)
      - [1.2 部署dcgm-exporter](#12-部署dcgm-exporter)
      - [1.3 dcgm-exporter metrics](#13-dcgm-exporter-metrics)
      - [1.4 添加Prometheus配置](#14-添加prometheus配置)
      - [1.5 数据可视化](#15-数据可视化)

1.生产环境

```
aiip-product-master01     172.22.120.3     
aiip-product-master02     172.22.120.4     
aiip-product-master03     172.22.120.5     
aiip-vgpu01-v100          172.21.195.40
```

Prometheus访问地址：http://172.21.195.40:31000/

2测试环境

```
dev-vgpu-master1    172.31.192.180    
dev-vgpu-master2    172.31.192.181    
dev-vgpu-master3    172.31.192.182    
dev-GPU-master4    172.31.192.183
```

Grafana访问地址：http://172.31.192.180:32000/

Prometheus访问地址：http://172.31.192.180:31000/

# 部署Prometheus&Grafnaa

#### 1.1 创建账号&证书

```
kubectl create ns monitoring
kubectl create serviceaccount monitor -n monitoring
kubectl create clusterrolebinding monitor-clusterrolebinding -n monitoring --clusterrole=cluster-admin --serviceaccount=monitoring:monitor
```

创建存储目录（共享）

> prometheus需要挂在的hostpath

注意：如果是非共享目录，需要放置在prometheus工作负载的节点。

```
mkdir -p /apps/sharedstorage/jtcmcccaiip/aiip/prometheus/172.22.120.3/data
mkdir -p /apps/sharedstorage/jtcmcccaiip/aiip/prometheus/172.22.120.3/conf/fs.d
chmod 777 /apps/sharedstorage/jtcmcccaiip/aiip/prometheus/172.22.120.3/data
chmod 777 /apps/sharedstorage/jtcmcccaiip/aiip/prometheus/172.22.120.3/conf/fs.d
```

证书导入：fs.d目录

> 1.24 版本后创建sa不回自动创建secret，需要手动去创建

创建Secret

```
apiVersion: v1
kind: Secret
type: kubernetes.io/service-account-token
metadata:
  name: token-secret
  namespace: monitoring
  annotations:
    kubernetes.io/service-account.name: monitor
```

查询生成的证书放入/fs.d中

```
cd /apps/sharedstorage/jtcmcccaiip/aiip/prometheus/172.22.120.3/conf/fs.d
# k8s.token
kubectl describe secrets -n monitoring token-secret >> k8s.token

# ca.crt 
kubectl get secrets  -n monitoring token-secret  -oyaml | grep "ca.crt:"| awk '{print $2}' | base64 -d >>ca.crt 
```



#### 1.2 部署 prometheus

加密密码：

- 用户：admin
- 密码：Gypsophila@25ai#2020

```
htpasswd -nbBC 10 admin "Gypsophila@25ai#2020"
```

会生成一个加密的密码，用于在cm中配置访问认证



创建 cm配置文件：

```
kind: ConfigMap
apiVersion: v1
metadata:
  labels:
    app: prometheus
  name: prometheus-config
  namespace: monitoring
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
      scrape_timeout: 10s
      evaluation_interval: 1m
    scrape_configs:
    - job_name: 'kubernetes-node'
      kubernetes_sd_configs:
      - role: node
      relabel_configs:
      - source_labels: [__address__]
        regex: '(.*):10250'
        replacement: '${1}:9100'
        target_label: __address__
        action: replace
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)

    - job_name: 'kubernetes-gpu'
      kubernetes_sd_configs:
      - role: pod
      relabel_configs:
      - source_labels: [__address__]
        action: keep
        regex: '(.*):9400'
      - source_labels: [__meta_kubernetes_pod_node_name]
        action: replace
        target_label: node
      - source_labels: [__meta_kubernetes_pod_host_ip]
        action: replace
        target_label: node_ip
        
    - job_name: kubernetes-pods
      bearer_token_file: /etc/prometheus/fs.d/k8s.token
      scheme: https
      tls_config:
        ca_file: /etc/prometheus/fs.d/ca.crt
        insecure_skip_verify: true
      kubernetes_sd_configs:
      - api_server: https://172.22.120.3:6443
        role: pod
        bearer_token_file: /etc/prometheus/fs.d/k8s.token
        tls_config:
          ca_file: /etc/prometheus/fs.d/ca.crt
          insecure_skip_verify: true
      relabel_configs:
      - action: keep
        regex: true
        source_labels:
        - __meta_kubernetes_pod_annotation_prometheus_io_scrape
      - regex: '([^;]+);([^;]+);([^;]+);([^;]+)'
        source_labels:
        - __meta_kubernetes_namespace
        - __meta_kubernetes_pod_name
        - __meta_kubernetes_pod_annotation_prometheus_io_port
        - __meta_kubernetes_pod_annotation_prometheus_io_path
        target_label: __metrics_path__
        replacement: '/api/v1/namespaces/${1}/pods/http:${2}:${3}/proxy${4}'
      - target_label: __address__
        replacement: '172.22.120.3:6443'
      - action: labelmap
        regex: __meta_kubernetes_pod_label_(.+)
      - action: replace
        source_labels:
        - __meta_kubernetes_namespace
        target_label: kubernetes_namespace
      - action: replace
        source_labels:
        - __meta_kubernetes_pod_name
        target_label: kubernetes_pod_name
      - action: drop
        regex: 'Pending|Succeeded|Failed'
        source_labels:
        - __meta_kubernetes_pod_phase
        
      - bearer_token_file: /etc/prometheus/fs.d/k8s.token
        job_name: kubernetes-nodes-cadvisor
        kubernetes_sd_configs:
        - api_server: https://172.22.120.3:6443
          role: node
          bearer_token_file: /etc/prometheus/fs.d/k8s.token
          tls_config:
            ca_file: /etc/prometheus/fs.d/ca.crt
            insecure_skip_verify: true
        relabel_configs:
        - action: labelmap
          regex: __meta_kubernetes_node_label_(.+)
        - replacement: 172.22.120.3:6443
          target_label: __address__
        - regex: (.+)
          replacement: /api/v1/nodes/$1/proxy/metrics/cadvisor
          source_labels:
          - __meta_kubernetes_node_name
          target_label: __metrics_path__
        scheme: https
        tls_config:
          ca_file: /etc/prometheus/fs.d/ca.crt
          insecure_skip_verify: true
        metric_relabel_configs:
          - source_labels: [ __name__ ]
            regex: 'node_netstat_Icmp_OutMsgs'
            action: drop
          - regex: 'node'
            action: labeldrop
          - source_labels: [instance]
            separator: ;
            regex: (.+)
            target_label: node
            replacement: $1
            action: replace
  config.yml: |
    basic_auth_users:
      admin: $2y$10$R.DAzYEw.M5daIa09/7T6.U3D10QZuA5IAr4qjjNZ52OCKv6lpKOa
```

创建prometheus部署文件

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus-server
  namespace: monitoring
  labels:
    app: prometheus
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus
      component: server
  template:
    metadata:
      labels:
        app: prometheus
        component: server
      annotations:
        prometheus.io/scrape: 'false'
    spec:
      serviceAccountName: monitor
      containers:
      - name: prometheus
        image: mirror.k8s.com:8989/cmri/prometheus:v2.42
        imagePullPolicy: IfNotPresent
        command:
          - prometheus
          - --config.file=/etc/prometheus/prometheus.yml
          - --storage.tsdb.path=/prometheus
          - --storage.tsdb.retention=35d
          - --web.enable-lifecycle
          - --web.config.file=/etc/prometheus/config.yml
        ports:
        - containerPort: 9090
          protocol: TCP
        volumeMounts:
        - name: prometheus-config
          mountPath: /etc/prometheus/prometheus.yml
          subPath: prometheus.yml
        - name: prometheus-config
          mountPath: /etc/prometheus/config.yml
          subPath: config.yml
        - name: prometheus-storage-volume
          mountPath: /prometheus/
        - name: prometheus-config-volume
          mountPath: /etc/prometheus/fs.d/
        - name: prometheus-time
          mountPath: /etc/localtime
      volumes:
      - name: prometheus-config
        configMap:
          name: prometheus-config
          items:
          - key: prometheus.yml
            path: prometheus.yml
            mode: 0644
          - key: config.yml
            path: config.yml
            mode: 0644
      - name: prometheus-storage-volume
        hostPath:
          path: /apps/sharedstorage/jtcmcccaiip/aiip/prometheus/172.22.120.3/data
          type: DirectoryOrCreate
      - name: prometheus-config-volume
        hostPath:
          path: /apps/sharedstorage/jtcmcccaiip/aiip/prometheus/172.22.120.3/conf/fs.d
          type: DirectoryOrCreate
      - name: prometheus-time
        hostPath:
          path: /etc/localtime
          type: ""
```

部署SVC

```
apiVersion: v1
kind: Service
metadata:
  name: prometheus
  namespace: monitoring
  labels:
    app: prometheus
spec:
  type: NodePort
  ports:
    - port: 9090
      targetPort: 9090
      protocol: TCP
      nodePort: 31000
  selector:
    app: prometheus
    component: server
```

#### 1.3 部署Grafana

部署文件

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: monitoring-grafana
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      task: monitoring
      k8s-app: grafana
  template:
    metadata:
      labels:
        task: monitoring
        k8s-app: grafana
    spec:
      nodeName: machine1
      containers:
      - name: grafana
        image: 172.31.197.130:8989/istio/grafana:9.3.2
        ports:
        - containerPort: 3000
          protocol: TCP
        volumeMounts:
        - mountPath: /var/lib/grafana
          name: grafana-storage
        env:
        - name: INFLUXDB_HOST
          value: monitoring-influxdb
        - name: GF_SERVER_HTTP_PORT
          value: "3000"
        - name: GF_AUTH_BASIC_ENABLED
          value: "true"
        - name: GF_AUTH_ANONYMOUS_ENABLED
          value: "true"
        - name: GF_AUTH_ANONYMOUS_ORG_ROLE
          value: Admin
        - name: GF_SERVER_ROOT_URL
          value: /
      volumes:
      - name: grafana-storage
        hostPath:
          path: /var/grafana
          type: Directory
```

# 监控项 Exporter

- Kube-state-metrics
- Node exporter
- Elasticsearch Exporter
- dcgm-exporter

部署 Exporter 重在举一反三

------

## kube-state-metrics

  使用 Prometheus 来监控 Kubernetes 集群的时候，`kube-state-metrics（KSM）` 基本属于一个必备组件，它通过 Watch APIServer 来生成资源对象的状态指标，它并不会关注单个 Kubernetes 组件的健康状况，而是关注各种资源对象的健康状态，比如 Deployment、Node、Pod、Ingress、Job、Service 等等，每种资源对象中包含了需要指标，我们可以在官方文档 https://github.com/kubernetes/kube-state-metrics/tree/main/docs 查看。

#### 1.1 集群版本对应

要安装 KSM 也非常简单，在安装的时候记得要和你的 K8s 集群版本对应。



#### 1.2 安装KMS

这里的集群是 v1.25 版本的，所以我先切换到该分支v2.7.0：

 网址：https://github.com/kubernetes/kube-state-metrics/tree/v2.7.0/examples



```
$ kubectl apply -f examples/standard
```

> 提前准备需要的镜像：registry.k8s.io/kube-state-metrics/kube-state-metrics:v2.7.0

该方式会以 Deployment 方式部署一个 KSM 实例：

```
$ kubectl get deploy -n kube-system kube-state-metrics
NAME                 READY   UP-TO-DATE   AVAILABLE   AGE
kube-state-metrics   1/1     1            1           2m49s
$ kubectl get pods -n kube-system -l app.kubernetes.io/name=kube-state-metrics
NAME                                  READY   STATUS    RESTARTS   AGE
kube-state-metrics-548546fc89-zgkx5   1/1     Running   0          2m51s
```

然后只需要让 Prometheus 来发现 KSM 实例就可以了

KSM 在 service 文件中明显可以看到暴露了两个 HTTP 端口：

```
apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/component: exporter
    app.kubernetes.io/name: kube-state-metrics
    app.kubernetes.io/version: 2.7.0
  name: kube-state-metrics
  namespace: kube-system
spec:
  clusterIP: None
  ports:
  - name: http-metrics
    port: 8080
    targetPort: http-metrics
  - name: telemetry
    port: 8081
    targetPort: telemetry
  selector:
    app.kubernetes.io/name: kube-state-metrics
```

可以分别去请求一下，看看返回的内容：

```
curl -s ${ip}:8080/metrics
curl -s ${ip}:8081/metrics
```

- 8080 端口返回的内容就是各类 Kubernetes 对象信息，比如 node 相关的信息
- 8081 端口，暴露的是 KSM 自身的指标，KSM 要调用 APIServer 的接口，watch 相关数据，需要度量这些动作的健康状况

#### 1.3 抓取 KSM

> 通过 prometheus agent mode 来抓取一下

```
    - job_name: 'kube-state-metrics'
      kubernetes_sd_configs:
      - role: endpoints
      scheme: http
      relabel_configs:
      - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
        action: keep
        regex: kube-system;kube-state-metrics;http-metrics

    - job_name: 'kube-state-metrics-self'
      kubernetes_sd_configs:
      - role: endpoints
      scheme: http
      relabel_configs:
      - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
        action: keep
        regex: kube-system;kube-state-metrics;telemetry
```

#### 1.4 数据可视化

- ***\*13332\****

## Node Exporter

node-exporter用于采集node的运行指标

> 二进制安装

node_exporter默认侦听HTTP端口9100

安装文档：[官方](https://prometheus.io/docs/guides/node-exporter/)

node exporter安装包: [v1.6.1-linux](https://github.com/prometheus/node_exporter/releases/download/v1.6.1/node_exporter-1.6.1.linux-amd64.tar.gz)

#### 1.1 安装 Node Exporter

https://github.com/prometheus/node_exporter/releases/download/v1.6.1/node_exporter-1.6.1.linux-amd64.tar.gz

```
# 下载包
wget https://github.com/prometheus/node_exporter/releases/download/v1.6.1/node_exporter-1.6.1.linux-amd64.tar.gz

#减压，执行可执行文件
tar xvfz node_exporter-1.6.1.linux-amd64.tar.gz
cd node_exporter-1.6.1.linux-amd64
nohup ./node_exporter &
```

#### 1.2 Node Exporter metrics

安装并运行后，可以通过 /metrics 来验证：

```
curl http://localhost:9100/metrics
```

可以看到如下输出:

```
# HELP go_gc_duration_seconds A summary of the GC invocation durations.
# TYPE go_gc_duration_seconds summary
go_gc_duration_seconds{quantile="0"} 3.8996e-05
go_gc_duration_seconds{quantile="0.25"} 4.5926e-05
go_gc_duration_seconds{quantile="0.5"} 5.846e-05
```

#### 1.3 抓取Node Exporter

prometheus.yml 配置文件将通过 localhost:9100 从 Node Exporter 进行抓取

```
  - job_name: 'node_exporter'
    static_configs:
      - targets: ['*:9100']
```

#### 1.4 数据可视化

热门模版

- 8919

#### 1.5 systemctl封装

创建 node_exporter systemd 服务单元文件

```
# 权限问题，可能需要使用vim
sudo vim /usr/lib/systemd/system/node_exporter.service

sudo cat << EOF > /usr/lib/systemd/system/node_exporter.service
[Unit]
Description=https://prometheus.io

[Service]
Restart=on-failure
ExecStart=/apps/node_exporter-1.6.0.linux-amd64/node_exporter --path.procfs=/proc --path.sysfs=/sys --path.rootfs=/ --path.udev.data=/run/udev/data --collector.cpu.info
[Install]
WantedBy=multi-user.target
EOF
```

\# 重载 systemd 并启用/启动 node_exporter 服务

```
sudo systemctl daemon-reload
sudo systemctl enable --now node_exporter
sudo systemctl enable node_exporter
sudo systemctl status node_exporter
```

## Elasticsearch Exporter

安装文档：[文档](https://github.com/prometheus-community/elasticsearch_exporter)

Elasticsearch Exporter安装包：[elasticsearch_exporter-1.6.0.linux-amd64.tar.gz](https://github.com/prometheus-community/elasticsearch_exporter/releases/download/v1.6.0/elasticsearch_exporter-1.6.0.linux-amd64.tar.gz)

#### 1.1 安装Elasticsearch Exporter

```
tar -xvf elasticsearch_exporter-1.6.0.linux-amd64.tar.gz 
cd elasticsearch_exporter-1.6.0.linux-amd64
nohup ./elasticsearch_exporter  --es.all --es.indices --es.cluster_settings --es.indices_settings --es.shards --es.snapshots  --web.listen-address=":9114" --web.telemetry-path="/metrics" --es.uri=http://elastic:Wy2nQHclKTDYmDFfIcb@10.174.107.12:9200 &
```

- es.uri：连接的Elasticsearch集群的URI

- - 需要身份验证时：[http://admin](http://admin/):pass@localhost:9200
  - 默认：[http://localhost:9200](http://localhost:9200/)

- web.listen-address：指定Elasticsearch Exporter监听的地址和端口，默认：`9114`

- --web.telemetry-path ： 指定Elasticsearch Exporter暴露指标的路径，Prometheus将从这个路径获取指标数据，默认：` /metrics`

#### 1.2 elasticsearch exporter metrics

```
curl http://elastic:Wy2nQHclKTDYmDFfIcb@10.174.107.13:9114/metrics | grep elasticsearch_cluster_health_status
```

应该可以看到如下输出:

```
elasticsearch_cluster_health_status{cluster="graylog-es",color="green"} 1
elasticsearch_cluster_health_status{cluster="graylog-es",color="red"} 0
elasticsearch_cluster_health_status{cluster="graylog-es",color="yellow"} 0
```

#### 1.3 添加Prometheus配置

```
  - job_name: 'elasticsearch'
    honor_timestamps: true
    metrics_path: '/metrics'
    static_configs:
      - targets: ['username:password@10.174.107.12:9114', 'username:password@10.174.107.13:9114', 'username:password@10.174.107.14:9114']
```

#### 1.4 数据可视化

模版

- 2322

## dcgm-exporter

> 监控Kubernetes集群的GPU资源

DCGM：NVIDIA数据中心GPU管理器

将其集成到Prometheus和Grafana等开源工具中，以实现Kubernetes的GPU监控的整体解决方案：

#### 1.1 NVIDIA DCGM

​	NVIDIA DCGM是用于管理和监控基于Linux系统的NVIDIA GPU大规模集群的一体化工具。它是一个低开销的工具，提供多种能力，包括主动健康监控、诊断、系统验证、策略、电源和时钟管理、配置管理和审计等。

#### 1.2 部署dcgm-exporter

> 使用Prometheus Operator部署Prometheus，那么推荐使用Helm

部署安装包：dcgm-exporter.2.0.13.tar.gz

```
docker load -i dcgm-exporter.2.0.13.tar.gz
docker run -d --gpus all --rm -p 9400:9400 nvidia/dcgm-exporter:2.0.13-2.1.1-ubuntu18.04
```

查看容器状态：

```
# docker ps
CONTAINER ID   IMAGE                                           COMMAND                  CREATED              STATUS              PORTS                    NAMES
198fdc1b5cff   nvidia/dcgm-exporter:2.0.13-2.1.1-ubuntu18.04   "/usr/local/dcgm/dcg…"   About a minute ago   Up About a minute   0.0.0.0:9400->9400/tcp   objective_morse
```

#### 1.3 dcgm-exporter metrics

```
curl localhost:9400/metrics
```

#### 1.4 添加Prometheus配置

```
  - job_name: 'gpu'
    static_configs:
    - targets: ['*:9400']
```

#### 1.5 数据可视化

热门模版

- 12639