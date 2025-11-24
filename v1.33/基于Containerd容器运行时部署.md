# 基于containerd容器运行时部署k8s 1.33.6集群 

# 一、主机准备

## 1.1 主机操作系统说明

| 序号 | 操作系统及版本 | 备注 |
| :--: | :------------: | :--: |
|  1   |    Rocky9.6    |      |



## 1.2 主机硬件配置说明

| CPU  | 内存 | 硬盘 | 角色           | 主机名     |
| ---- | ---- | ---- | -------------- | ---------- |
| 4C   | 8G   | 50GB | master(worker) | k8s-node-1 |
| 4C   | 8G   | 50GB | master(worker) | k8s-node-2 |
| 4C   | 8G   | 50GB | master(worker) | k8s-node-3 |



## 1.3 主机配置

### 1.3.1  主机名配置

由于本次使用3台主机完成kubernetes集群部署

~~~bash
hostnamectl set-hostname k8s-node-1
hostnamectl set-hostname k8s-node-2
hostnamectl set-hostname k8s-node-3
~~~

### 1.3.2 主机免密

```bash
ssh-keygen
ssh-copy-id 192.168.88.181
ssh-copy-id 192.168.88.182
ssh-copy-id 192.168.88.183
```

### 1.3.3 主机名与IP地址解析

>  所有集群主机均需要进行配置。

~~~bash
cat>> /etc/hosts << EOF
192.168.88.181 k8s-node-1
192.168.88.182 k8s-node-2
192.168.88.183 k8s-node-3
EOF
~~~

### 1.3.4 验证mac地址uuid

> 保证各节点mac和uuid唯一，防止克隆主机出现网络异常问题

```
cat /sys/class/net/ens33/address
cat /sys/class/dmi/id/product_uuid 
```

### 1.3.5  防火墙配置

> 所有主机均需要操作。

~~~bash
systemctl disable firewalld
systemctl stop firewalld
firewall-cmd --state

# not running
~~~

### 1.3.6 SELINUX配置

> 所有主机均需要操作。修改SELinux配置需要**重启**操作系统。

~~~bash
sed -ri 's/SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config && sestatus
~~~

### 1.3.7 时间同步配置

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
server 192.168.88.181  # node-1 IP
EOF

systemctl restart chronyd
systemctl enable --now chronyd
chronyc sources
```

> 所有主机均需要操作。

```
chronyc tracking #校准硬件时间
```

### 1.3.8  配置内核转发及网桥过滤

>所有主机均需要操作。

~~~bash
# 添加网桥过滤及内核转发配置文件
cat > /etc/sysctl.d/k8s.conf << EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
vm.swappiness = 0
EOF

sysctl --system
~~~

~~~bash
# 加载br_netfilter、overlay模块
modprobe br_netfilter && modprobe overlay
~~~

~~~bash
# 查看是否加载
lsmod | grep -E "br_netfilter|overlay"

#  br_netfilter           22256  0
#  bridge                151336  1 br_netfilter
~~~

### 1.3.9 安装ipset及ipvsadm

> 所有主机均需要操作。

~~~bash
# 安装ipset及ipvsadm
dnf -y install ipset ipvsadm
~~~

~~~bash
# 配置ipvsadm模块加载方式,添加需要加载的模块
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

~~~bash
#授权、运行、检查是否加载
chmod 755 /etc/sysctl.d/ipvs.modules && bash /etc/sysctl.d/ipvs.modules && lsmod | grep -e ip_vs -e nf_conntrack
~~~

```bash
# 添加开机自动加载模块
echo "/etc/sysctl.d/ipvs.modules" >> /etc/rc.local && chmod +x /etc/rc.local
```

### 1.3.9 安装命令自动补全工具

> 所有主机均需要操作。

```bash
dnf -y install bash-completion 
source /etc/profile.d/bash_completion.sh 
```

# 二、容器运行时 Containerd准备

## 2.1 Containerd准备

### 2.1.1 Containerd部署文件获取

~~~powershell
wget https://github.com/containerd/containerd/releases/download/v2.2.0/containerd-2.2.0-linux-amd64.tar.gz
~~~

~~~bash
tar Cxzvf /usr/local containerd-2.2.0-linux-amd64.tar.gz
~~~

```bash
wget https://raw.githubusercontent.com/containerd/containerd/main/containerd.service  -P /usr/local/lib/systemd/system/
```

```bash
systemctl daemon-reload
systemctl enable --now containerd
containerd --version
```

### 2.1.2 Containerd配置文件生成并修改

~~~bash
mkdir /etc/containerd && containerd config default > /etc/containerd/config.toml
~~~

~~~bash
# 修改配置文件，修改镜像源和pause版本
vim /etc/containerd/config.toml
sandbox_image=“registry.aliyuncs.com/google_containers/pause:3.10.1"
~~~

```bash
# 配置containerd 代理
mkdir -p /etc/containerd/certs.d/docker.io  && cat > /etc/containerd/certs.d/docker.io/hosts.toml << EOF
server = "https://docker.io"
[host."https://docker.1ms.run"]
  capabilities = ["pull", "resolve"]
EOF


mkdir -p /etc/containerd/certs.d/registry.k8s.io  && cat > /etc/containerd/certs.d/registry.k8s.io/hosts.toml << EOF
server = "https://registry.k8s.io"
[host."https://swr.cn-north-4.myhuaweicloud.com/ddn-k8s/registry.k8s.io"]
  capabilities = ["pull", "resolve"]

[host."https://k8s.m.daocloud.io"]
  capabilities = ["pull", "resolve"]
  
[host."https://registry.cn-hangzhou.aliyuncs.com/google_containers"]
  capabilities = ["pull", "resolve"]
EOF
```

```bash
systemctl daemon-reload
systemctl restart containerd
systemctl status containerd
```



## 2.2 runc准备

### 2.2.2 runc安装

~~~bash
wget https://github.com/opencontainers/runc/releases/download/v1.3.3/runc.amd64
~~~

~~~powershell
install -m 755 runc.amd64 /usr/local/sbin/runc
~~~

~~~bash
# 执行runc命令，如果有命令帮助则为正常
runc
~~~



## 2.3 安装 CNI 插件

```bash
wget https://github.com/containernetworking/plugins/releases 
```

```bash
mkdir -p /opt/cni/bin && tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.1.1.tgz
```



## 2.4 安装配置nerdctl

```bash
# 下载
wget https://github.com/containerd/nerdctl/releases/download/v2.2.0/nerdctl-2.2.0-linux-amd64.tar.gz
# 解压
tar Cxzvf /usr/local/bin nerdctl-2.2.0-linux-amd64.tar.gz
# 配置 nerdctl 参数自动补齐
echo 'source <(nerdctl completion bash)' >> /etc/profile && source /etc/profile
# 验证
nerdctl -v
```



## 2.5 安装配置crictl

```bash
# 下载
wget https://github.com/kubernetes-sigs/cri-tools/releases/download/v1.33.0/crictl-v1.33.0-linux-amd64.tar.gz
# 解压
tar Czxvf /usr/local/bin crictl-v1.33.0-linux-amd64.tar.gz
# 配置
cat > /etc/crictl.yaml << EOF
runtime-endpoint: "unix:///run/containerd/containerd.sock"
image-endpoint: "unix:///run/containerd/containerd.sock"
timeout: 0
debug: false
pull-image-on-create: false
disable-pull-on-run: false
EOF
# 验证
crictl version
```



# 三、K8S集群部署

## 3.1 K8S集群软件准备

### 3.1.1 YUM源

```bash
# googleYUM源
cat > /etc/yum.repos.d/google.k8s.repo <<EOF
[google_kubernetes]
name=Google_Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg
        https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF
```

```bash
# 阿里云旧版YUM源 kubernets v1.28.0以前版本
cat > /etc/yum.repos.d/k8s.repo <<EOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```

```bash
# 阿里云新版YUM源 kubernets v1.28.0 及以后版本 https://developer.aliyun.com/mirror/kubernetes/
cat > /etc/yum.repos.d/ali.k8s.repo <<EOF
[ali_kubernetes]
name=Ali_Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes-new/core/stable/v1.33/rpm/
enabled=1
gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes-new/core/stable/v1.33/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF
```

```bash
# 官方YUM源
cat > /etc/yum.repos.d/k8s.repo <<EOF
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.33/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.33/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF
```





### 3.1.2 离线安装

```bash
RELEASE="v1.33.6"
ARCH="amd64"
DOWNLOAD_DIR="/usr/local/bin"

cd $DOWNLOAD_DIR
sudo curl -L --remote-name-all https://dl.k8s.io/release/${RELEASE}/bin/linux/${ARCH}/{kubeadm,kubelet}
sudo chmod +x {kubeadm,kubelet}

RELEASE_VERSION="v0.16.2"
curl -sSL "https://raw.githubusercontent.com/kubernetes/release/${RELEASE_VERSION}/cmd/krel/templates/latest/kubelet/kubelet.service" | sed "s:/usr/bin:${DOWNLOAD_DIR}:g" | sudo tee /usr/lib/systemd/system/kubelet.service
sudo mkdir -p /usr/lib/systemd/system/kubelet.service.d
curl -sSL "https://raw.githubusercontent.com/kubernetes/release/${RELEASE_VERSION}/cmd/krel/templates/latest/kubeadm/10-kubeadm.conf" | sed "s:/usr/bin:${DOWNLOAD_DIR}:g" | sudo tee /usr/lib/systemd/system/kubelet.service.d/10-kubeadm.conf

cd 
curl -LO "https://dl.k8s.io/release/${RELEASE}/bin/linux/${ARCH}/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

sudo systemctl enable --now kubelet
sudo systemctl status kubelet
echo "source <(kubectl completion bash)" >> ~/.bash_profile  && source ~/.bash_profile

kubeadm version
kubelet --version
kubectl version
```



## 3.2 KUBE_VIP软件安装

```bash
mkdir -p /etc/kubernetes/manifests
export VIP=192.168.88.188
export INTERFACE=ens33
KVVERSION=v1.0.2
```

```bash
alias kube-vip="ctr image pull ghcr.io/kube-vip/kube-vip:$KVVERSION; ctr run --rm --net-host ghcr.io/kube-vip/kube-vip:$KVVERSION vip /kube-vip"
```

```bash
kube-vip manifest pod \
    --interface $INTERFACE \
    --address $VIP \
    --controlplane \
    --services \
    --arp \
    --leaderElection | tee /etc/kubernetes/manifests/kube-vip.yaml
```

```bash
sed -i 's#path: /etc/kubernetes/admin.conf#path: /etc/kubernetes/super-admin.conf#' \
          /etc/kubernetes/manifests/kube-vip.yaml
```



## 3.3 K8S集群初始化

```bash
kubeadm config print init-defaults > kubeadm-conf.yaml
```

> 具体注释见文末

```yaml
apiVersion: kubeadm.k8s.io/v1beta4
kind: InitConfiguration
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: "012345.6789abcdef012345"
  ttl: "24h"
  usages:
  - signing
  - authentication
  description: "bootstrap token for worker/controlplane join"
localAPIEndpoint:
  advertiseAddress: 192.168.88.181
  bindPort: 6443
nodeRegistration:
  name: "k8s-node-1"
  criSocket: unix:///var/run/containerd/containerd.sock
  imagePullPolicy: IfNotPresent
  imagePullSerial: true
  taints: null
timeouts:
  controlPlaneComponentHealthCheck: 4m0s
  discovery: 5m0s
  etcdAPICall: 2m0s
  kubeletHealthCheck: 4m0s
  kubernetesAPICall: 1m0s
  tlsBootstrap: 5m0s
  upgradeManifests: 5m0s

---
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration
kubernetesVersion: v1.33.6                    # 指定 Kubernetes 版本（与 kubeadm 兼容）
imageRepository:  registry.aliyuncs.com/google_containers

controlPlaneEndpoint: "192.168.88.181:6443"
networking:
  dnsDomain: "cluster.local"
  serviceSubnet: "10.96.0.0/12"
  podSubnet: "10.10.0.0/16"

apiServer:
  extraArgs:
    - name: "authorization-mode"
      value: "Node,RBAC"
  certSANs:
  - 127.0.0.1
  - k8s-node-1
  - k8s-node-2
  - k8s-node-3
  - 192.168.88.181
  - 192.168.88.182
  - 192.168.88.183
  - 192.168.88.188
controllerManager: {}
scheduler: {}
etcd:
  local:
    dataDir: "/var/lib/etcd"
caCertificateValidityPeriod: 87600h0m0s
certificateValidityPeriod: 8760h0m0s
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes-test
encryptionAlgorithm: RSA-2048

---
# 指定kube-proxy基于ipvs模式
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: ipvs

---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: "systemd"
failSwapOn: true
readOnlyPort: 0
authentication:
  anonymous:
    enabled: false
  webhook:
    cacheTTL: 0s
    enabled: true
  x509:
    clientCAFile: "/etc/kubernetes/pki/ca.crt"
authorization:
  mode: "Webhook"
  webhook:
    cacheAuthorizedTTL: 0s
    cacheUnauthorizedTTL: 0s
clusterDNS:
- 10.96.0.10
```

```bash
kubeadm config images pull --config kubeadm-conf.yaml
```

```bash
# 查看 containerd images
ctr -n k8s.io images list
```

> 集群初始化, 首节点执行即可

```bash
kubeadm init --upload-certs --config=kubeadm-conf.yaml
```

>  如果配置问题导致集群初始化失败，可重置集群再次初始化：

```bash
kubeadm reset --config=kubeadm-conf.yaml
ipvsadm --clear
rm -rf $HOME/.kube/config
```





## 3.4 节点加入集群

~~~powershell
[root@k8s-node-2 ~]# kubeadm join 192.168.88.181:6443 --token hd74hg.r8l1pe4tivwyjz73 \
>         --discovery-token-ca-cert-hash sha256:29a00daed8d96dfa8e913ab4c0a8c4037f1c253a20742ca8913932dd7c8b3bd1 --cri-socket unix:///var/run/containerd/containerd.sock
~~~

~~~powershell
[root@k8s-node-3 ~]# kubeadm join 192.168.88.181:6443 --token hd74hg.r8l1pe4tivwyjz73 \
>         --discovery-token-ca-cert-hash sha256:29a00daed8d96dfa8e913ab4c0a8c4037f1c253a20742ca8913932dd7c8b3bd1 --cri-socket unix:///var/run/containerd/containerd.sock
~~~

```bash
mkdir -p $HOME/.kube 
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config 
chown $(id -u):$(id -g) $HOME/.kube/config
echo "export KUBECONFIG=/etc/kubernetes/admin.conf" >> ~/.bash_profile
source ~/.bash_profile
```

```bash
sed -i 's#path: /etc/kubernetes/super-admin.conf#path: /etc/kubernetes/admin.conf#'   \ /etc/kubernetes/manifests/kube-vip.yaml
```



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







```
# kubeadm v1.33.6 生产级 init 配置示例
# 使用 apiVersion: kubeadm.k8s.io/v1beta4（适用于 v1.33.x 的 kubeadm 配置格式）
# 如果你使用不同的 kubeadm 版本，请先确认支持的 apiVersion。
apiVersion: kubeadm.k8s.io/v1beta4
kind: InitConfiguration
# ---------------------------------------------------------------------------
# InitConfiguration 主要包含运行时相关（bootstrap token, localAPIEndpoint 等）
# ---------------------------------------------------------------------------
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: "012345.6789abcdef0123456789abcdef"   # 示例 token，生产建议使用随机并短期有效的 token，或使用 --certificate-key join
  ttl: "24h"                                  # token 有效期，加入节点之后可撤销
  usages:
  - signing
  - authentication
  description: "bootstrap token for worker/controlplane join"
localAPIEndpoint:
  advertiseAddress: 192.168.88.181
  bindPort: 6443                   # API server 监听端口（默认 6443）
nodeRegistration:
  name: "k8s-node-1"                # 节点主机名（默认取系统 hostname）
  criSocket: unix:///var/run/containerd/containerd.sock   # CRI socket
  imagePullPolicy: IfNotPresent
  imagePullSerial: true
  taints: null

timeouts:
  controlPlaneComponentHealthCheck: 4m0s
  discovery: 5m0s
  etcdAPICall: 2m0s
  kubeletHealthCheck: 4m0s
  kubernetesAPICall: 1m0s
  tlsBootstrap: 5m0s
  upgradeManifests: 5m0s

---
# ---------------------------------------------------------------------------
# ClusterConfiguration：集群级别的核心配置（网络、证书、API server args 等）
# ---------------------------------------------------------------------------
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration
kubernetesVersion: v1.33.6                    # 指定 Kubernetes 版本（与 kubeadm 兼容）
imageRepository:  registry.aliyuncs.com/google_containers
               # 可替换为你公司的镜像加速仓库（比如 my.repo.local/registry.k8s.io）
# 如需离线安装或私有仓库，请确保在所有节点把镜像拉取到本地 registry 或配置 imagePullSecrets。

controlPlaneEndpoint: "192.168.88.181:6443"   # 推荐填 LB 或 VIP（若单节点可填节点IP:6443）
# controlPlaneEndpoint 会写入 kubeadm-config ConfigMap，便于 join 时自动使用。

networking:
  dnsDomain: "cluster.local"
  serviceSubnet: "10.96.0.0/12"    # Service 网段（与 CNI 要求一致）
  podSubnet: "10.10.0.0/16"      # Pod 网段：选择与所用 CNI 插件兼容的网段（flannel 默认 10.244.0.0/16）
  # 生产建议：根据集群规模规划 podSubnet（避免与物理网段冲突）

apiServer: {}
  extraArgs:
    # 常见的生产级参数，可按需开启/调整
    authorization-mode: Node,RBAC
    audit-log-path: "/var/log/kubernetes/audit.log"
    audit-log-maxage: "30"
    audit-log-maxbackup: "10"
    audit-log-maxsize: "100"
    # 如果使用私有证书或特殊特性可在这里添加 extraArgs
  certSANs:
  - 127.0.0.1
  - k8s-node-1
  - k8s-node-2
  - k8s-node-3
  - 192.168.88.181
  - 192.168.88.182
  - 192.168.88.183
  - 192.168.88.188

controllerManager: {}
scheduler: {}

etcd:
  # 默认使用本地堆栈式 etcd（适合小到中型集群）。生产中大型建议使用独立外部 etcd 集群（参见下面 external etcd 注释）。
  local:
    # 数据目录（etcd static pod 会使用）
    dataDir: "/var/lib/etcd"


caCertificateValidityPeriod: 876000h0m0s
certificateValidityPeriod: 8760h0m0s
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes-test
encryptionAlgorithm: RSA-2048

---
# 指定kube-proxy基于ipvs模式
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: ipvs

---
# ---------------------------------------------------------------------------
# KubeletConfiguration：以 kubelet.config.k8s.io API 编写（示例常用项）
# 注意：kubeadm 会把此配置写入 /var/lib/kubelet/config.yaml（kubelet 使用）
# ---------------------------------------------------------------------------
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
# 与 kubeadm 的 nodeRegistration.kubeletExtraArgs 保持一致（优先级请参考 kubeadm 文档）
cgroupDriver: "systemd"         # production 推荐 systemd（如果 runtime 使用 systemd）
failSwapOn: true                # 生产环境请关闭 swap，kubelet 默认要求 swap off
readOnlyPort: 0
authentication:
  anonymous:
    enabled: false
  webhook:
    cacheTTL: 0s
    enabled: true
  x509:
    clientCAFile: "/etc/kubernetes/pki/ca.crt"
authorization:
  mode: "Webhook"               # 使用 RBAC/认证 webhook，生产环境常用
  webhook:
    cacheAuthorizedTTL: 0s
    cacheUnauthorizedTTL: 0s
metricsBindAddress: "127.0.0.1" # 若需要远程抓取请配置合适的地址并做访问控制
clusterDNS:
- 10.96.0.10

# ----------------------------------------------------------------------------
# Addons & post-install 提示（不是 kubeadm 字段，仅为文档说明）
# ----------------------------------------------------------------------------
# - 在 kubeadm init 完成并且 kubectl 可用后，请安装 CNI（例如 Calico、Cilium、Flannel 等）
#   确保 CNI 与上面配置的 podSubnet 兼容。
# - 建议启用网络策略（如 Calico NetworkPolicy）和集群级别监控、日志、证书轮转工具等。
# - 建议配置 PodSecurityAdmission/OPA/Gatekeeper 等策略控制。
# ----------------------------------------------------------------------------

```

