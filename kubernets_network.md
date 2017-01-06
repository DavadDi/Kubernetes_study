# Docker或Kubernets的网络模型

## 阿里云部署K8S

今天在 [kubeup.com](kubeup.com) 看到通过官方的k8s部署中已经支持了aliyun，（[阿里元的部署步骤](https://tryk8s.com/tutorial/deploy-kubernetes-on-aliyun/)），其实现的方式是在开源的Flannel上添加了一个[Watcher](https://github.com/tryk8s/flannel/tree/watcher/watcher) 的功能来支持阿里的VPC, 是一个非常不错的实现方式。

如果采用阿里VPC方式创建k8s的话，Flannel的host-gw模式应该也是比较不错的选择，因为既然自建VPC当然可以保证cluster node都能够二层上互通。

此外对于Flannel中的host-gw和vxlan方式都能够达到native的性能，如果考虑通用性且服务对于网络敏感度要求不是非常高的情况下，采用vxlan的方式也是一个非常好的方案。

另外一个疑问？Flannel的对于网络后端的支持都在backend目录，该目录下包括gce/udp/vxlan/hostgw/alloc/awsvpc等各种方式，对于阿里VPC的支持应该在[本目录flannel/backend](https://github.com/tryk8s/flannel/tree/watcher/backend)加入一个新的backend type比较好？

## 相关文章

1. [Flannel for Docker Overlay Network](http://chunqi.li/2015/10/10/Flannel-for-Docker-Overlay-Network/) 重点讲了Etcd的配置运行与性能测试（UDP方式和VxLan，其中Vxlan的性能接近于native）
2. [docker网络方案简介](http://www.jingyuyun.com/article/12183.html) 分析docker中的网络方案：包括 host模式、fannel（udp/vxlan/host-gw)、 calico、桥接+ipam
3. [flannel host-gw network](http://hustcat.github.io/flannel-host-gw-network/) 文章主要分析了flannel的host-gw的安装配置和测试，host-gw的性能仅次于native，但是主机要求二层连通。
4. [一个适合 Kubernetes 的最佳网络互联方案](http://dockone.io/article/1115) 本文比较了几种 Kubernetes 联网方案，包括 Flannel（aws-vpc | host-gw | vxlan）和 IPvlan 。目前建议选择 flannel + host-gw 方案，没有特别依赖，性能也够用。一旦flannel支持IPvlan（有自动化设置工具了），且Linux内核版本比较新，就可以采用 IPvlan 方案
5. [Kubernets,Flannel,Docker网络性能深度测试](http://pangxiekr.com/kubernetsflannel-wang-luo-xing-neng-ce-shi-ji-diao-you/) 文章对于Flannel的各种模式进行了详细的测试