#CentOS7 安装 Kubernetes

## 1. 整体结构图

### 版本信息

* Kubernetes-1.2
* docker-1.9.0
* flannel-0.5.3
* etcd-2.1.1


## 2. kubernetes环境部署

1. Master:172.16.198.129
2. Slave: 172.16.198.128



两台虚拟机准备工作:

* 每台机器禁用firewalld：

###
	systemctl stop firewalld
	systemctl disable firewalld
### 

* 禁用selinux：

### 
	# vi /etc/selinux/config
	# SELINUX=enforcing  
	SELINUX=disabled
####

也可用命令
###
	sed -i '/SELINUX/s/enforcing/disabled/' /etc/selinux/config
###

## 2.1 Master机器安装与配置
###
	#yum -y install etcd kubernetes
###

###
	#vi /etc/etcd/etcd.conf
	ETCD_NAME=default
	ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
	ETCD_LISTEN_CLIENT_URLS="http://0.0.0.0:2379"
	ETCD_ADVERTISE_CLIENT_URLS="
###

###
	#vi /etc/kubernetes/apiserver
	KUBE_API_ADDRESS="--address=0.0.0.0"
	KUBE_ETCD_SERVERS="--etcd_servers=http://172.16.198.129:2379"
	KUBE_SERVICE_ADDRESSES="--service-cluster-ip-range=10.254.0.0/16"
	KUBE_ADMISSION_CONTROL="--admission_control=NamespaceLifecycle,NamespaceExists,LimitRanger,SecurityContextDeny,ServiceAccount,ResourceQuota"
	KUBE_API_ARGS=""
###

###
	#vi /etc/kubernetes/controller-manager
	KUBE_CONTROLLER_MANAGER_ARGS="--node-monitor-grace-period=10s --pod-eviction-timeout=10s"
###

###
	#vi /etc/kubernetes/config
	KUBE_LOGTOSTDERR="--logtostderr=true"
	KUBE_LOG_LEVEL="--v=0"
	KUBE_ALLOW_PRIV="--allow_privileged=false"
	KUBE_MASTER="--master=http://172.16.198.129:8080"
###
	
启动服务:
###
	for SERVICES in etcd kube-apiserver kube-controller-manager 	kube-scheduler; do
    	systemctl restart $SERVICES
   		systemctl enable $SERVICES
    	systemctl status $SERVICES 
	done
###

定义flannel网络配置到etcd,这个配置会推送到各个Slave的flannel服务上
###
	etcdctl mk /coreos.com/network/config '{"Network":"172.17.0.0/16"}'
###

## 2.2 Slave:机器的安装与配置

### 安装docker并更新重启
###
	#yum -y install docker
	#yum -y update
	#reboot
###

###
	#yum -y install kubernetes-node flannel
###

###
	# vi /etc/kubernetes/config
	KUBE_LOGTOSTDERR="--logtostderr=true"
	KUBE_LOG_LEVEL="--v=0"
	KUBE_ALLOW_PRIV="--allow_privileged=false"
	KUBE_MASTER="--master=http://172.16.198.129:8080"
###

###
	# vi /etc/kubernetes/kubelet
	KUBELET_ADDRESS="--address=127.0.0.1"
	KUBELET_HOSTNAME="--hostname_override=172.16.198.128"
	KUBELET_API_SERVER="--api_servers=http://172.16.198.129:8080"
	KUBELET_ARGS="--pod-infra-container-image=kubernetes/pause"
###

###
	# vi /etc/sysconfig/flanneld
	FLANNEL_ETCD="http://172.16.198.129:2379"
	FLANNEL_ETCD_KEY="/coreos.com/network"
###

### 启动服务
###

	for SERVICES in kube-proxy kubelet docker flanneld; do
	    systemctl restart $SERVICES
	    systemctl enable $SERVICES
	    systemctl status $SERVICES 
	done
###

登陆master，确认minions的状态：

###
	[root@master ~]# kubectl get nodes
	NAME             LABELS                                  STATUS
	172.16.198.128   kubernetes.io/hostname=172.16.198.128   Ready
###

注：如果需要多台Slave节点，只需要重复上面slava机器上的操作
