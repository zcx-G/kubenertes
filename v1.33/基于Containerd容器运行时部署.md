# 基于containerd容器运行时部署k8s 1.28集群 

# 一、主机准备

## 1.1 主机操作系统说明

| 序号 | 操作系统及版本 | 备注 |
| :--: | :------------: | :--: |
|  1   |   CentOS7u9    |      |



## 1.2 主机硬件配置说明

| CPU  | 内存 | 硬盘 | 角色           | 主机名     |
| ---- | ---- | ---- | -------------- | ---------- |
| 4C   | 8G   | 50GB | master(worker) | k8s-node-1 |
| 4C   | 8G   | 50GB | master(worker) | k8s-node-2 |
| 4C   | 8G   | 50GB | master(worker) | k8s-node-3 |



## 1.3 主机配置

### 1.3.1  主机名配置

由于本次使用3台主机完成kubernetes集群部署，其中1台为master节点,名称为k8s-node-1;其中2台为worker节点，名称分别为：k8s-node-2及k8s-node-3

~~~powershell
master节点
# hostnamectl set-hostname k8s-node-1
~~~



~~~powershell
worker01节点
# hostnamectl set-hostname k8s-node-2
~~~



~~~powershell
worker02节点
# hostnamectl set-hostname k8s-node-3
~~~



### 1.3.2 主机免密

```bash
ssh-keygen
ssh-copy-id 192.168.88.181
ssh-copy-id 192.168.88.182
ssh-copy-id 192.168.88.183
```



### 1.3.3 主机名与IP地址解析



> 所有集群主机均需要进行配置。

~~~powershell
cat>> /etc/hosts << EOF
192.168.88.181 k8s-node-1
192.168.88.182 k8s-node-2
192.168.88.183 k8s-node-3
EOF
~~~





## 验证mac地址uuid

保证各节点mac和uuid唯一，防止克隆主机出现网络异常问题

```
cat /sys/class/net/ens33/address
cat /sys/class/dmi/id/product_uuid 
```



### 1.3.4  防火墙配置



> 所有主机均需要操作。



~~~powershell
关闭现有防火墙firewalld
systemctl disable firewalld
systemctl stop firewalld
firewall-cmd --state
not running
~~~



### 1.3.5 SELINUX配置



> 所有主机均需要操作。修改SELinux配置需要重启操作系统。



~~~powershell
sed -ri 's/SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config && sestatus
~~~



~~~powershell
# sestatus
~~~



### 1.3.6 时间同步配置



>所有主机均需要操作。最小化安装系统需要安装ntpdate软件。



~~~powershell
# crontab -l
0 */1 * * * /usr/sbin/ntpdate time1.aliyun.com
~~~



node-1 服务端配置

```
cat >> /etc/chrony.conf << EOF
server ntp.aliyun.com iburst
allow 192.168.88.0/24
EOF

systemctl restart chronyd
systemctl enable --now chronyd
chronyc sources
```



node-2/3 客户端配置

```
cat >> /etc/chrony.conf << EOF
server 192.168.88.181 
EOF

systemctl restart chronyd
systemctl enable --now chronyd
chronyc sources
```



```
chronyc tracking #校准硬件时间
```





### 1.3.8  配置内核转发及网桥过滤

>所有主机均需要操作。



~~~powershell
添加网桥过滤及内核转发配置文件
cat > /etc/sysctl.d/k8s.conf << EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
vm.swappiness = 0
EOF

sysctl --system
~~~



~~~powershell
加载br_netfilter模块
modprobe br_netfilter
modprobe overlay
~~~



~~~powershell
查看是否加载
# lsmod | grep -E "br_netfilter|overlay"
br_netfilter           22256  0
bridge                151336  1 br_netfilter
~~~



### 1.3.9 安装ipset及ipvsadm

> 所有主机均需要操作。



~~~powershell
安装ipset及ipvsadm
dnf -y install ipset ipvsadm
~~~



~~~powershell
配置ipvsadm模块加载方式
添加需要加载的模块
cat > /etc/sysctl.d/ipvs.modules  <<EOF
#!/bin/bash
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack
modprobe -- br_netfilter
EOF
~~~



~~~powershell
授权、运行、检查是否加载
# chmod 755 /etc/sysctl.d/ipvs.modules && bash /etc/sysctl.d/ipvs.modules && lsmod | grep -e ip_vs -e nf_conntrack
~~~



```
# 添加开机自动加载模块
echo "/etc/sysctl.d/ipvs.modules" >> /etc/rc.local
chmod +x /etc/rc.local

```

## 安装命令自动补全工具

```
dnf -y install bash-completion 
source /etc/profile.d/bash_completion.sh 
```

# 二、容器运行时 Containerd准备

## 2.1 Containerd准备

### 2.1.1 Containerd部署文件获取

~~~powershell
# wget https://github.com/containerd/containerd/releases/download/v1.7.3/cri-containerd-1.7.3-linux-amd64.tar.gz
~~~

~~~powershell
tar Cxzvf /usr/local containerd-1.6.2-linux-amd64.tar.gz
~~~

```bash
wget https://raw.githubusercontent.com/containerd/containerd/main/containerd.service  -P /usr/local/lib/systemd/system/
```

```
systemctl daemon-reload
systemctl enable --now containerd
containerd --version
```



### 2.1.2 Containerd配置文件生成并修改



~~~powershell
mkdir /etc/containerd && containerd config default > /etc/containerd/config.toml
~~~



~~~powershell
# vim /etc/containerd/config.toml

sandbox_image=“registry.aliyuncs.com/google_containers/pause:3.9" 由3.8修改为3.9
~~~







## 2.2 runc准备

### 2.2.2 runc安装

~~~powershell
# wget https://github.com/opencontainers/runc/releases/download/v1.1.5/runc.amd64
~~~



~~~powershell
install -m 755 runc.amd64 /usr/local/sbin/runc
~~~

~~~powershell
执行runc命令，如果有命令帮助则为正常
# runc
~~~





#### 安装 CNI 插件

```
wget https://github.com/containernetworking/plugins/releases 
```

```
mkdir -p /opt/cni/bin
tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.1.1.tgz
```



# 三、K8S集群部署

## 3.1 K8S集群软件YUM源准备

### 3.1.1 google提供YUM源

~~~powershell
# cat > /etc/yum.repos.d/k8s.repo <<EOF
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg
        https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF
~~~



### 3.1.2 阿里云提供YUM源

~~~powershell
# cat > /etc/yum.repos.d/k8s.repo <<EOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
~~~





## 3.2 K8S集群软件安装

### 3.2.1 集群软件安装

> 所有节点均可安装

~~~powershell
默认安装
# yum -y install  kubeadm  kubelet kubectl
~~~



~~~powershell
查看指定版本
# yum list kubeadm.x86_64 --showduplicates | sort -r
# yum list kubelet.x86_64 --showduplicates | sort -r
# yum list kubectl.x86_64 --showduplicates | sort -r
~~~



~~~powershell
安装指定版本
# yum -y install  kubeadm-1.28.X-0  kubelet-1.28.X-0 kubectl-1.28.X-0
~~~





### 3.2.2  配置kubelet

>为了实现docker使用的cgroupdriver与kubelet使用的cgroup的一致性，建议修改如下文件内容。



~~~powershell
# vim /etc/sysconfig/kubelet
KUBELET_EXTRA_ARGS="--cgroup-driver=systemd"
~~~



~~~powershell
设置kubelet为开机自启动即可，由于没有生成配置文件，集群初始化后自动启动
# systemctl enable kubelet
~~~



## 3.3 K8S集群初始化



~~~powershell
[root@k8s-node-1 ~]# kubeadm init --kubernetes-version=v1.28.0 --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=192.168.88.181  --cri-socket unix:///var/run/containerd/containerd.sock
~~~



~~~powershell
[init] Using Kubernetes version: v1.28.0
[preflight] Running pre-flight checks
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [k8s-node-1 kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 192.168.88.181]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [k8s-node-1 localhost] and IPs [192.168.88.181 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [k8s-node-1 localhost] and IPs [192.168.88.181 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Starting the kubelet
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[apiclient] All control plane components are healthy after 20.502191 seconds
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config" in namespace kube-system with the configuration for the kubelets in the cluster
[upload-certs] Skipping phase. Please see --upload-certs
[mark-control-plane] Marking the node k8s-node-1 as control-plane by adding the labels: [node-role.kubernetes.io/control-plane node.kubernetes.io/exclude-from-external-load-balancers]
[mark-control-plane] Marking the node k8s-node-1 as control-plane by adding the taints [node-role.kubernetes.io/control-plane:NoSchedule]
[bootstrap-token] Using token: hd74hg.r8l1pe4tivwyjz73
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to get nodes
[bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] Configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] Configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[kubelet-finalize] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.88.181:6443 --token hd74hg.r8l1pe4tivwyjz73 \
        --discovery-token-ca-cert-hash sha256:29a00daed8d96dfa8e913ab4c0a8c4037f1c253a20742ca8913932dd7c8b3bd1
~~~



## 3.4 工作节点加入集群



~~~powershell
[root@k8s-node-2 ~]# kubeadm join 192.168.88.181:6443 --token hd74hg.r8l1pe4tivwyjz73 \
>         --discovery-token-ca-cert-hash sha256:29a00daed8d96dfa8e913ab4c0a8c4037f1c253a20742ca8913932dd7c8b3bd1 --cri-socket unix:///var/run/containerd/containerd.sock
~~~



~~~powershell
[root@k8s-node-3 ~]# kubeadm join 192.168.88.181:6443 --token hd74hg.r8l1pe4tivwyjz73 \
>         --discovery-token-ca-cert-hash sha256:29a00daed8d96dfa8e913ab4c0a8c4037f1c253a20742ca8913932dd7c8b3bd1 --cri-socket unix:///var/run/containerd/containerd.sock
~~~



## 3.5 验证K8S集群节点是否可用



~~~powershell
[root@k8s-node-1 ~]# kubectl get nodes
NAME           STATUS   ROLES           AGE   VERSION
k8s-node-1   Ready    control-plane   15m   v1.28.0
k8s-node-2   Ready    <none>          13m   v1.28.0
k8s-node-3   Ready    <none>          13m   v1.28.0
~~~



# 四、网络插件calico部署

> calico访问链接：https://projectcalico.docs.tigera.io/about/about-calico



![image-20230404115348450](基于Containerd容器运行时部署K8S 1.28集群.assets/image-20230404115348450.png)



![image-20230404115500987](基于Containerd容器运行时部署K8S 1.28集群.assets/image-20230404115500987.png)





~~~powershell
# kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/tigera-operator.yaml
~~~



~~~powershell
# wget https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/custom-resources.yaml
~~~



~~~powershell
# vim custom-resources.yaml


# cat custom-resources.yaml


# This section includes base Calico installation configuration.
# For more information, see: https://projectcalico.docs.tigera.io/master/reference/installation/api#operator.tigera.io/v1.Installation
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  # Configures Calico networking.
  calicoNetwork:
    # Note: The ipPools section cannot be modified post-install.
    ipPools:
    - blockSize: 26
      cidr: 10.244.0.0/16 修改此行内容为初始化时定义的pod network cidr
      encapsulation: VXLANCrossSubnet
      natOutgoing: Enabled
      nodeSelector: all()

---

# This section configures the Calico API server.
# For more information, see: https://projectcalico.docs.tigera.io/master/reference/installation/api#operator.tigera.io/v1.APIServer
apiVersion: operator.tigera.io/v1
kind: APIServer
metadata:
  name: default
spec: {}
~~~



~~~powershell
# kubectl create -f custom-resources.yaml

installation.operator.tigera.io/default created
apiserver.operator.tigera.io/default created
~~~



~~~powershell
[root@k8s-node-1 ~]# kubectl get pods -n calico-system
NAME                                       READY   STATUS    RESTARTS   AGE
calico-kube-controllers-6bb86c78b4-cnr9l   1/1     Running   0          2m26s
calico-node-86cs9                          1/1     Running   0          2m26s
calico-node-gjgcc                          1/1     Running   0          2m26s
calico-node-hlr69                          1/1     Running   0          2m26s
calico-typha-6f877c9d8f-8f5fb              1/1     Running   0          2m25s
calico-typha-6f877c9d8f-spxqf              1/1     Running   0          2m26s
csi-node-driver-9b8nd                      2/2     Running   0          2m26s
csi-node-driver-rg6dc                      2/2     Running   0          2m26s
csi-node-driver-tf82w                      2/2     Running   0          2m26s
~~~





