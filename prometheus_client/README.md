# Prometheus监控实践样例程序

参考： https://github.com/giantswarm/kubernetes-prometheus

```
# 注解格式
prometheus.io/scrape，为true则会将pod作为监控目标。
prometheus.io/path，默认为/metrics
prometheus.io/port , 端口
prometheus.io/scheme 默认http，如果为了安全设置了https，此处需要改为https
```

## 监控 go 应用程序

```
Configure Prometheus data source for Grafana.
Grafana UI / Data Sources / Add data source
Name: prometheus
Type: Prometheus
Url: http://prometheus:9090
Add
```

```
Import Prometheus Stats:
Grafana UI / Dashboards / Import

Grafana.net Dashboard: https://grafana.net/dashboards/2
Load
Prometheus: prometheus
Save & Open

```
Import Kubernetes cluster monitoring:
Grafana UI / Dashboards / Import

Grafana.net Dashboard: https://grafana.net/dashboards/162
Load
Prometheus: prometheus
Save & Open
```

其他 Go程序相关 DashBoard
Go Processes https://grafana.com/dashboards/240
Go Processes for Kubernetes  https://grafana.com/dashboards/3574

## 参考：

1. [Prometheus监控实践：Kubernetes集群监控](https://blog.frognew.com/2017/12/using-prometheus-to-monitor-kubernetes.html#3kubernetes%E9%9B%86%E7%BE%A4%E4%B8%8A%E9%83%A8%E7%BD%B2%E5%BA%94%E7%94%A8%E7%9A%84%E7%9B%91%E6%8E%A7)
1. [k8s与监控--解读prometheus监控kubernetes的配置文件](https://segmentfault.com/a/1190000013230914)
1. [Kubernetes Setup for Prometheus and Grafana](https://github.com/giantswarm/kubernetes-prometheus)
1. [The Prometheus Operator: Managed Prometheus setups for Kubernetes](https://coreos.com/blog/the-prometheus-operator.html)
1. https://github.com/coreos/prometheus-operator
1. [prometheus-kubernetes.yml](https://github.com/prometheus/prometheus/blob/2bd510a63e48ac6bf4971d62199bdb1045c93f1a/documentation/examples/prometheus-kubernetes.yml)
1. [实战 | 使用Prometheus监控Kubernetes集群和应用](https://www.kancloud.cn/huyipow/kubernetes/grafana/grafana/kubernetes-apps_rev1.json)
