# Prometheus监控实践样例程序

## 入门

下载 

```bash
$ wget https://github.com/prometheus/prometheus/releases/download/v2.3.2/prometheus-2.3.2.darwin-amd64.tar.gz
```

运行

```bash
$ ./prometheus --version
prometheus, version 2.3.2 (branch: HEAD, revision: 71af5e29e815795e9dd14742ee7725682fa14b7b)
  build user:       root@5258e0bd9cc1
  build date:       20180712-14:08:00
  go version:       go1.10.3
$ ./prometheus --config.file=prometheus.yml
$ open http://localhost:9090/metrics  # 进程指标监控
$ open http://localhost:9090/graph    # 图形界面查看


# docker 
docker run -p 9090:9090 -v ./prometheus.yml:/etc/prometheus/prometheus.yml prom/prometheus
```



Help

```bash
./prometheus -h                                               ✔  2650  14:06:37
usage: prometheus [<flags>]

The Prometheus monitoring server

Flags:
  -h, --help                     Show context-sensitive help (also try --help-long and --help-man).
      --version                  Show application version.
      --config.file="prometheus.yml"
                                 Prometheus configuration file path.
      --web.listen-address="0.0.0.0:9090"
                                 Address to listen on for UI, API, and telemetry.
      --web.read-timeout=5m      Maximum duration before timing out read of the request, and closing idle connections.
      --web.max-connections=512  Maximum number of simultaneous connections.
      --web.external-url=<URL>   The URL under which Prometheus is externally reachable (for example, if Prometheus is served via a
                                 reverse proxy). Used for generating relative and absolute links back to Prometheus itself. If the URL
                                 has a path portion, it will be used to prefix all HTTP endpoints served by Prometheus. If omitted,
                                 relevant URL components will be derived automatically.
      --web.route-prefix=<path>  Prefix for the internal routes of web endpoints. Defaults to path of --web.external-url.
      --web.user-assets=<path>   Path to static asset directory, available at /user.
      --web.enable-lifecycle     Enable shutdown and reload via HTTP request.
      --web.enable-admin-api     Enable API endpoints for admin control actions.
      --web.console.templates="consoles"
                                 Path to the console template directory, available at /consoles.
      --web.console.libraries="console_libraries"
                                 Path to the console library directory.
      --storage.tsdb.path="data/"
                                 Base path for metrics storage.
      --storage.tsdb.retention=15d
                                 How long to retain samples in storage.
      --storage.tsdb.no-lockfile
                                 Do not create lockfile in data directory.
      --storage.remote.flush-deadline=<duration>
                                 How long to wait flushing sample on shutdown or config reload.
      --alertmanager.notification-queue-capacity=10000
                                 The capacity of the queue for pending Alertmanager notifications.
      --alertmanager.timeout=10s
                                 Timeout for sending alerts to Alertmanager.
      --query.lookback-delta=5m  The delta difference allowed for retrieving metrics during expression evaluations.
      --query.timeout=2m         Maximum time a query may take before being aborted.
      --query.max-concurrency=20
                                 Maximum number of queries executed concurrently.
      --log.level=info           Only log messages with the given severity or above. One of: [debug, info, warn, error]
```



### ##  kubernets 中安装 

参考 https://github.com/giantswarm/kubernetes-prometheus/tree/master/manifests

### Namespace

```bash
$ kubectl create -f 0-namespace.yaml
```
0-namespace.yaml 
```bash
$ cat 0-namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: monitoring
```

```bash
$ ls -hl
configmap.yaml  # 创建配置 prometheus.yaml config amp
deployment.yaml # deployment 相关配置 
rbac.yaml       # 相关的 rbac 权限设置
service.yaml    # 服务相关信息
```

### RBAC

```bash
$ kubectl create -f rbac.yaml
```

```yaml
$ cat rbac.yaml
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: prometheus
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus
subjects:
- kind: ServiceAccount
  name: prometheus-k8s
  namespace: monitoring
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus
rules:
- apiGroups: [""]
  resources:
  - nodes
  - services
  - endpoints
  - pods
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources:
  - configmaps
  verbs: ["get"]
- nonResourceURLs: ["/metrics"]
  verbs: ["get"]
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus-k8s
  namespace: monitoring
```

### 配置

相关配置保存在  prometheus.yaml 文件中

```bash
$ kubectl create configmap prometheus-core --from-file prometheus.yaml -n monitoring
```

通过这种方式可以很方便的修改配置文件进行测试。



Deployment.yaml 和 service.yaml

```bash
$ kubectl proxy
# 访问当前 node 列表
$ http://127.0.0.1:8001/api/v1/nodes

# 获取对应 node 上的 metircs
$ http://127.0.0.1:8001/api/v1/nodes/docker-for-desktop/proxy/metrics

# 访问 node 上的 cadvistor 指标
$ http://127.0.0.1:8001/api/v1/nodes/docker-for-desktop/proxy/metrics/cadvisor
```



### `kubernetes_sd_config`

`<kubernetes_sd_config>`

Kubernetes SD configurations allow retrieving scrape targets from [Kubernetes'](http://kubernetes.io/) REST API and always staying synchronized with the cluster state.

One of the following `role` types can be configured to discover targets:

#### `node`

The `node` role discovers one target per cluster node with the address defaulting to the Kubelet's HTTP port. The target address defaults to the first existing address of the Kubernetes node object in the address type order of `NodeInternalIP`, `NodeExternalIP`, `NodeLegacyHostIP`, and `NodeHostName`.

Available meta labels:

- `__meta_kubernetes_node_name`: The name of the node object.
- `__meta_kubernetes_node_label_<labelname>`: Each label from the node object.
- `__meta_kubernetes_node_annotation_<annotationname>`: Each annotation from the node object.
- `__meta_kubernetes_node_address_<address_type>`: The first address for each node address type, if it exists.

In addition, the `instance` label for the node will be set to the node name as retrieved from the API server.

#### `service`

The `service` role discovers a target for each service port for each service. This is generally useful for blackbox monitoring of a service. The address will be set to the Kubernetes DNS name of the service and respective service port.

Available meta labels:

- `__meta_kubernetes_namespace`: The namespace of the service object.
- `__meta_kubernetes_service_name`: The name of the service object.
- `__meta_kubernetes_service_label_<labelname>`: The label of the service object.
- `__meta_kubernetes_service_annotation_<annotationname>`: The annotation of the service object.
- `__meta_kubernetes_service_port_name`: Name of the service port for the target.
- `__meta_kubernetes_service_port_number`: Number of the service port for the target.
- `__meta_kubernetes_service_port_protocol`: Protocol of the service port for the target.

#### `pod`

The `pod` role discovers all pods and exposes their containers as targets. For each declared port of a container, a single target is generated. If a container has no specified ports, a port-free target per container is created for manually adding a port via relabeling.

Available meta labels:

- `__meta_kubernetes_namespace`: The namespace of the pod object.
- `__meta_kubernetes_pod_name`: The name of the pod object.
- `__meta_kubernetes_pod_ip`: The pod IP of the pod object.
- `__meta_kubernetes_pod_label_<labelname>`: The label of the pod object.
- `__meta_kubernetes_pod_annotation_<annotationname>`: The annotation of the pod object.
- `__meta_kubernetes_pod_container_name`: Name of the container the target address points to.
- `__meta_kubernetes_pod_container_port_name`: Name of the container port.
- `__meta_kubernetes_pod_container_port_number`: Number of the container port.
- `__meta_kubernetes_pod_container_port_protocol`: Protocol of the container port.
- `__meta_kubernetes_pod_ready`: Set to `true` or `false` for the pod's ready state.
- `__meta_kubernetes_pod_node_name`: The name of the node the pod is scheduled onto.
- `__meta_kubernetes_pod_host_ip`: The current host IP of the pod object.
- `__meta_kubernetes_pod_uid`: The UID of the pod object.
- `__meta_kubernetes_pod_controller_kind`: Object kind of the pod controller.
- `__meta_kubernetes_pod_controller_name`: Name of the pod controller.

#### `endpoints`

The `endpoints` role discovers targets from listed endpoints of a service. For each endpoint address one target is discovered per port. If the endpoint is backed by a pod, all additional container ports of the pod, not bound to an endpoint port, are discovered as targets as well.

Available meta labels:

- `__meta_kubernetes_namespace`: The namespace of the endpoints object.
- `__meta_kubernetes_endpoints_name`: The names of the endpoints object.
- For all targets discovered directly from the endpoints list (those not additionally inferred from underlying pods), the following labels are attached:
  - `__meta_kubernetes_endpoint_ready`: Set to `true` or `false` for the endpoint's ready state.
  - `__meta_kubernetes_endpoint_port_name`: Name of the endpoint port.
  - `__meta_kubernetes_endpoint_port_protocol`: Protocol of the endpoint port.
  - `__meta_kubernetes_endpoint_address_target_kind`: Kind of the endpoint address target.
  - `__meta_kubernetes_endpoint_address_target_name`: Name of the endpoint address target.
- If the endpoints belong to a service, all labels of the `role: service` discovery are attached.
- For all targets backed by a pod, all labels of the `role: pod` discovery are attached.

#### `ingress`

The `ingress` role discovers a target for each path of each ingress. This is generally useful for blackbox monitoring of an ingress. The address will be set to the host specified in the ingress spec.

Available meta labels:

- `__meta_kubernetes_namespace`: The namespace of the ingress object.
- `__meta_kubernetes_ingress_name`: The name of the ingress object.
- `__meta_kubernetes_ingress_label_<labelname>`: The label of the ingress object.
- `__meta_kubernetes_ingress_annotation_<annotationname>`: The annotation of the ingress object.
- `__meta_kubernetes_ingress_scheme`: Protocol scheme of ingress, `https` if TLS config is set. Defaults to `http`.
- `__meta_kubernetes_ingress_path`: Path from ingress spec. Defaults to `/`.



样例文件地址： https://github.com/prometheus/prometheus/blob/release-2.3/documentation/examples/prometheus-kubernetes.yml

## `relabel_config`

Relabeling is a powerful tool to dynamically rewrite the label set of a target before it gets scraped. Multiple relabeling steps can be configured per scrape configuration. They are applied to the label set of each target in order of their appearance in the configuration file.

Initially, aside from the configured per-target labels, a target's `job` label is set to the `job_name` value of the respective scrape configuration. The `__address__` label is set to the `<host>:<port>` address of the target. After relabeling, the `instance` label is set to the value of `__address__` by default if it was not set during relabeling. The `__scheme__` and `__metrics_path__` labels are set to the scheme and metrics path of the target respectively. The `__param_<name>` label is set to the value of the first passed URL parameter called `<name>`.

Additional labels prefixed with `__meta_` may be available during the relabeling phase. They are set by the service discovery mechanism that provided the target and vary between mechanisms.

Labels starting with `__` will be removed from the label set after relabeling is completed.

If a relabeling step needs to store a label value only temporarily (as the input to a subsequent relabeling step), use the `__tmp` label name prefix. This prefix is guaranteed to never be used by Prometheus itself.

```
# The source labels select values from existing labels. Their content is concatenated
# using the configured separator and matched against the configured regular expression
# for the replace, keep, and drop actions.
[ source_labels: '[' <labelname> [, ...] ']' ]

# Separator placed between concatenated source label values.
[ separator: <string> | default = ; ]

# Label to which the resulting value is written in a replace action.
# It is mandatory for replace actions. Regex capture groups are available.
[ target_label: <labelname> ]

# Regular expression against which the extracted value is matched.
[ regex: <regex> | default = (.*) ]

# Modulus to take of the hash of the source label values.
[ modulus: <uint64> ]

# Replacement value against which a regex replace is performed if the
# regular expression matches. Regex capture groups are available.
[ replacement: <string> | default = $1 ]

# Action to perform based on regex matching.
[ action: <relabel_action> | default = replace ]
```



`<relabel_action>` determines the relabeling action to take:

- `replace`: Match `regex` against the concatenated `source_labels`. Then, set `target_label` to `replacement`, with match group references (`${1}`, `${2}`, ...) in `replacement` substituted by their value. If `regex` does not match, no replacement takes place.
- `keep`: Drop targets for which `regex` does not match the concatenated `source_labels`.
- `drop`: Drop targets for which `regex` matches the concatenated `source_labels`.
- `hashmod`: Set `target_label` to the `modulus` of a hash of the concatenated `source_labels`.
- `labelmap`: Match `regex` against all label names. Then copy the values of the matching labels to label names given by `replacement` with match group references (`${1}`, `${2}`, ...) in `replacement` substituted by their value.
- `labeldrop`: Match `regex` against all label names. Any label that matches will be removed from the set of labels.
- `labelkeep`: Match `regex` against all label names. Any label that does not match will be removed from the set of labels.

Care must be taken with `labeldrop` and `labelkeep` to ensure that metrics are still uniquely labeled once the labels are removed.



参考： https://github.com/giantswarm/kubernetes-prometheus


> 注解格式
> * prometheus.io/scrape，为true则会将pod作为监控目标。
> * prometheus.io/path，默认为/metrics
> * prometheus.io/port , 端口
> * prometheus.io/scheme 默认http，如果为了安全设置了https，此处需要改为https

Configure Prometheus data source for Grafana.
```
Grafana UI / Data Sources / Add data source
Name: prometheus
Type: Prometheus
Url: http://prometheus:9090
Add
```

Import Prometheus Stats:

```
Grafana UI / Dashboards / Import

Grafana.net Dashboard: https://grafana.net/dashboards/2
Load
Prometheus: prometheus
Save & Open
```

Import Kubernetes cluster monitoring:

```
Grafana UI / Dashboards / Import

Grafana.net Dashboard: https://grafana.net/dashboards/162
Load
Prometheus: prometheus
Save & Open
```

其他 Go 程序相关 DashBoard
Go Processes https://grafana.com/dashboards/240
Go Processes for Kubernetes  https://grafana.com/dashboards/3574



https://github.com/kubernetes/kubernetes/tree/master/cluster/addons/prometheus



配置样例：

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus
  namespace: monitoring
data:
  prometheus.yml: |-
    global:
      scrape_interval: 15s
      evaluation_interval: 15s

    scrape_configs:
    - job_name: 'prometheus'
      scrape_interval: 5s
      static_configs:
        - targets: ['127.0.0.1:9090']
    - job_name: etcd
      static_configs:
      - targets: ['172.16.63.202:2379','172.16.63.203:2379','172.16.63.204:2379','172.16.63.205:2379','172.16.63.206:2379']
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        cert_file: /var/run/secrets/pem/prometheus.pem
        key_file: /var/run/secrets/key/prometheus.key 

    - job_name: pushgateway
      static_configs:
      - targets: ['pushgateway:9091']

    - job_name: 'kubernetes-apiservers'
      kubernetes_sd_configs:
      - role: endpoints
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        insecure_skip_verify: true
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      relabel_configs:
      - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
        action: keep
        regex: default;kubernetes;https
    - job_name: 'kubernetes-nodes'
      scheme: https
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

    - job_name: 'kubernetes-cadvisor'
      scheme: https
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

    - job_name: 'kubernetes-service-endpoints'

      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
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
        action: keep
        regex: .*-metrics
      - source_labels: [__meta_kubernetes_service_name]
        action: replace
        target_label: kubernetes_name


    - job_name: 'kubernetes-services'
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
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
      - source_labels: [__meta_kubernetes_pod_container_port_name]
        action: keep
        regex: .*-metrics

```

配置参见：http://ylzheng.com/2018/01/17/prometheus-sd-and-relabel/



监控： 

* [blackbox_exporter的配置](https://blog.frognew.com/2018/02/prometheus-blackbox-exporter.html#blackbox_exporter%E7%9A%84%E9%85%8D%E7%BD%AE)  存活监控和服务监控
* [kube-state-metrics](https://github.com/kubernetes/kube-state-metrics)   整体状态监控
* [node_exporter](https://github.com/prometheus/node_exporter) 物理主机监控
* 容器监控 cAdvisor

![image-20180910171828075](/var/folders/21/fptp1vvx3td0pl10x2kff3g00000gn/T/abnerworks.Typora/image-20180910171828075.png)

## 参考：

1. [Prometheus监控实践：Kubernetes集群监控](https://blog.frognew.com/2017/12/using-prometheus-to-monitor-kubernetes.html#3kubernetes%E9%9B%86%E7%BE%A4%E4%B8%8A%E9%83%A8%E7%BD%B2%E5%BA%94%E7%94%A8%E7%9A%84%E7%9B%91%E6%8E%A7)
1. [k8s与监控--解读prometheus监控kubernetes的配置文件](https://segmentfault.com/a/1190000013230914)
1. [Kubernetes Setup for Prometheus and Grafana](https://github.com/giantswarm/kubernetes-prometheus)
1. [The Prometheus Operator: Managed Prometheus setups for Kubernetes](https://coreos.com/blog/the-prometheus-operator.html)
1. https://github.com/coreos/prometheus-operator
1. [prometheus-kubernetes.yml](https://github.com/prometheus/prometheus/blob/2bd510a63e48ac6bf4971d62199bdb1045c93f1a/documentation/examples/prometheus-kubernetes.yml)
1. [实战 | 使用Prometheus监控Kubernetes集群和应用](https://www.kancloud.cn/huyipow/kubernetes/grafana/grafana/kubernetes-apps_rev1.json)
1. [Kubernetes monitoring with Prometheus in 15 minutes](https://itnext.io/kubernetes-monitoring-with-prometheus-in-15-minutes-8e54d1de2e13)
1. [A Deep Dive into Kubernetes Metrics](https://blog.freshtracks.io/a-deep-dive-into-kubernetes-metrics-66936addedae)
