# K8s第一课-在Ubuntu上安装K8s集群

> 虽然Ubuntu和Centos都是Linux系统，但是安装的命令还是稍有区别；
>
> 这里给予的Ubuntu版本是20.04，对于更早的版本没有尝试，但是应该大差不差。

我们使用KubeAdm作为安装工具，这里没有过多的解释，目的就是方便快速搭建一个集群。

### 1. 禁用Swap分区

```bash
   # 注释掉swap一行
   sudo vi /etc/fstab
```

### 2. iptables设置

更改 `net.bridge.bridge-nf-call-iptables` 的值为1.（Ubuntu 20.04默认为1，可以不用做）

   ```bash
   sudo cat >>/etc/sysctl.d/k8s.conf<< EOF
   net.bridge.bridge-nf-call-ip6tables = 1
   net.bridge.bridge-nf-call-iptables = 1
   EOF
   
   sudo sysctl --system
   ```

### 3. 安装Docker

安装Docker（在安装的时候可以指定版本进行安装，和想要安装的Kuberntes版本保持一致）

```bash
sudo apt update && sudo apt -y upgrade

sudo apt install -y docker.io

sudo systemctl start docker && sudo systemctl enable docker
```

### 4. 安装Kubeadm、kubelet和kubectl

#### 4.1 首先安装依赖包

```bash
sudo apt-get update && sudo apt -y upgrade

sudo apt-get install -y ca-certificates curl software-properties-common apt-transport-https

sudo curl -s https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | sudo apt-key add -

sudo cat >>/etc/apt/sources.list.d/kubernetes.list <<EOF 
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF

sudo apt-get update && sudo apt -y upgrade
```

#### 4.2 开始安装Kubeadm、kubelet、kubectl

  ```bash
  # 查看可以安装的指定版本
  apt list kubeadm -a
  
  # 安装指定版本的Kubeadm、kubelet、kubectl
  sudo apt-get install -y kubelet=1.20.15-00 kubeadm=1.20.15-00 kubectl=1.20.15-00
  
  systemctl enable kubelet
  systemctl enable docker
  ```

### 5. 预下载k8s集群组件镜像

#### 5.1 查看 kubeadm init 时所需要的组件镜像列表

  ```bash
  # ：
  kubeadm config images list
  # 输出类似如下信息，这些代表是kubeadm要下载安装的组件;
  I1025 15:01:13.041337  340088 version.go:254] remote version is much newer: v1.25.3; falling back to: stable-1.20
  k8s.gcr.io/kube-apiserver:v1.20.15
  k8s.gcr.io/kube-controller-manager:v1.20.15
  k8s.gcr.io/kube-scheduler:v1.20.15
  k8s.gcr.io/kube-proxy:v1.20.15
  k8s.gcr.io/pause:3.2
  k8s.gcr.io/etcd:3.4.13-0
  k8s.gcr.io/coredns:1.7.0
  ```

#### 5.2 使用脚本下载并修改tag

  ```bash
  cat <<EOF > pull-k8s-images.sh
  for i in `kubeadm config images list`; do
    imageName=${i#k8s.gcr.io/}
    docker pull registry.aliyuncs.com/google_containers/$imageName
    docker tag registry.aliyuncs.com/google_containers/$imageName k8s.gcr.io/$imageName
    docker rmi registry.aliyuncs.com/google_containers/$imageName
  done;
  EOF
   
  # 执行脚本
  chmod +x pull-k8s-images.sh
  ./pull-k8s-images.sh
  ```

### 6. 安装k8s集群（kubeadm init）

  ```bash
  kubeadm init --apiserver-advertise-address=<使用自己的ip地址> --image-repository registry.aliyuncs.com/google_containers --kubernetes-version v1.21.1(修改为自己的版本) --service-cidr=10.96.0.0/12 --pod-network-cidr=10.244.0.0/16
  ```

### 7. 安装flannel插件

  ```shell
  kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
  ```

### 8. 测试集群：安装Nginx测试集群

  ```bash
  kubectl create deployment nginx --image=nginx
  
  kubectl expose deployment nginx --port=80 --type=NodePort
  
  kubectl get pod,svc
  ```

