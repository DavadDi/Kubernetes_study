# Prometheus监控实践样例程序

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

## 参考：

1. [Prometheus监控实践：Kubernetes集群监控](https://blog.frognew.com/2017/12/using-prometheus-to-monitor-kubernetes.html#3kubernetes%E9%9B%86%E7%BE%A4%E4%B8%8A%E9%83%A8%E7%BD%B2%E5%BA%94%E7%94%A8%E7%9A%84%E7%9B%91%E6%8E%A7)
1. [k8s与监控--解读prometheus监控kubernetes的配置文件](https://segmentfault.com/a/1190000013230914)
1. [Kubernetes Setup for Prometheus and Grafana](https://github.com/giantswarm/kubernetes-prometheus)
1. [The Prometheus Operator: Managed Prometheus setups for Kubernetes](https://coreos.com/blog/the-prometheus-operator.html)
1. https://github.com/coreos/prometheus-operator
1. [prometheus-kubernetes.yml](https://github.com/prometheus/prometheus/blob/2bd510a63e48ac6bf4971d62199bdb1045c93f1a/documentation/examples/prometheus-kubernetes.yml)
1. [实战 | 使用Prometheus监控Kubernetes集群和应用](https://www.kancloud.cn/huyipow/kubernetes/grafana/grafana/kubernetes-apps_rev1.json)

```

```