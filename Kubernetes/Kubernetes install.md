# 部署

CentOS环境。

 前置条件：

- Node节点都安装上Docker

- 集群中的所有机器的网络彼此均能相互连接(公网和内网都可以)

  ```bash
  # CentOS关闭防火墙
  systemctl stop firewalld && systemctl disable firewalld && iptables -F
  ```
  
    ```bash
  # 将 SELinux 设置为 permissive 模式（相当于将其禁用）
  sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config && setenforce 0
  
  #允许 iptables 检查桥接流量
  cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
  br_netfilter
  EOF
  
  cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
  net.bridge.bridge-nf-call-ip6tables = 1
  net.bridge.bridge-nf-call-iptables = 1
  EOF
  sudo sysctl --system
    ```
  
- 节点之中不可以有重复的主机名、MAC 地址或 product_uuid。请参见[这里](https://kubernetes.io/zh/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#verify-mac-address)了解更多详细信息。

    ```bash
    # 设置不同的hostname主机名
    hostnamectl set-hostname xxxx
    ```

- 开启机器上的某些端口。请参见[这里](https://kubernetes.io/zh/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#check-required-ports) 了解更多详细信息。

- 禁用交换分区， **必须**。

    ```bash
    swapoff -a  
    sed -ri 's/.*swap.*/#&/' /etc/fstab
    ```

## 下载

### CentOs

```bash
# 仓库
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
   http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF

sudo yum install -y kubelet-1.20.9 kubeadm-1.20.9 kubectl-1.20.9 --disableexcludes=kubernetes

# 开机自启
sudo systemctl enable --now kubelet
```

### Ubuntu

```bash
apt-get update && apt-get install -y apt-transport-https curl
apt-get install -y kubelet kubeadm kubectl --allow-unauthenticated
```

如果找不到软件包，添加阿里云仓库

```bash
echo 'deb https://mirrors.aliyun.com/kubernetes/apt kubernetes-xenial main'|tee -a /etc/apt/sources.list
```

## Master节点

下载镜像

```bash
sudo tee ./images.sh <<-'EOF'
#!/bin/bash
images=(
kube-apiserver:v1.20.9
kube-proxy:v1.20.9
kube-controller-manager:v1.20.9
kube-scheduler:v1.20.9
coredns:1.7.0
etcd:3.4.13-0
pause:3.2
)
for imageName in ${images[@]} ; do
docker pull registry.cn-hangzhou.aliyuncs.com/lfy_k8s_images/$imageName
done
EOF
   
chmod +x ./images.sh && ./images.sh
```

初始化

```bash
#所有机器添加master域名映射，以下需要修改为自己的
echo "172.31.0.4  cluster-endpoint" >> /etc/hosts

#主节点初始化
kubeadm init \
--apiserver-advertise-address=172.31.0.4 \
--control-plane-endpoint=cluster-endpoint \
--image-repository registry.cn-hangzhou.aliyuncs.com/lfy_k8s_images \
--kubernetes-version v1.20.9 \
--service-cidr=10.96.0.0/16 \
--pod-network-cidr=192.168.0.0/16
```

根据提示运行指令

设置.kube/config

安装网络组件

```bash
curl https://docs.projectcalico.org/v3.20/manifests/calico.yaml -O

kubectl apply -f calico.yaml
```

加入其它节点

在master节点上运行

```bash
kubeadm join cluster-endpoint:6443 --token x5g4uy.wpjjdbgra92s25pp \
	--discovery-token-ca-cert-hash sha256:6255797916eaee52bf9dda9429db616fcd828436708345a308f4b917d3457a22 \
--control-plane
```

在work节点上运行

```bash
kubeadm join cluster-endpoint:6443 --token x5g4uy.wpjjdbgra92s25pp \
	--discovery-token-ca-cert-hash sha256:6255797916eaee52bf9dda9429db616fcd828436708345a308f4b917d3457a22
验证集群节点状态
```

```bash
  kubectl get nodes
```



## dashboard

kubernetes官方提供的[可视化界面](https://github.com/kubernetes/dashboard)

安装

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.3.1/aio/deploy/recommended.yaml
```



配置端口

```bash
kubectl edit svc kubernetes-dashboard -n kubernetes-dashboard
```

放行

```bash
kubectl get svc -A |grep kubernetes-dashboard
## 找到端口，在安全组放行
```

创建访问账号

准备一个yaml文件； vi dash.yaml

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
```

应用

```bash
kubectl apply -f dash.yaml
```

令牌访问

todo????

```bash
#获取访问令牌
kubectl -n kubernetes-dashboard get secret $(kubectl -n kubernetes-dashboard get sa/admin-user -o jsonpath="{.secrets[0].name}") -o go-template="{{.data.token | base64decode}}"
```



## 问题

Q：CrashLoopBackOff

- 检查iptables规则是否放行

# 参考文献

