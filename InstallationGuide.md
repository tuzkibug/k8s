# 安装步骤  
## 准备工作  
* 安装必要的辅助工具，比如vim等  
* 如下操作均在root用户下，若不使用root用户，请使用具有sudo权限的用户  
* 关闭selinux    
vim /etc/sysconfig/selinux  
找到SELINUX=enforcing，改成SELINUX=disabled  
* 关闭防火墙  
防火墙会组织后续的容器间通信，需要提前关闭  
systemctl stop firewalld && systemctl disabled firewalld  
* 检查swap  
kubernetes必须工作在无swap环境，默认安装系统时分区不要分swap分区，如果分了，请手动关闭该分区  
swapon -s检查是否有分区，假如有使用swapoff /dev/xxx来关闭分区  
* sshd设置为keepalive  
为方便安装，将ssh功能设置为keepalive保持连接  
echo "ClientAliveInterval 10" >> /etc/ssh/sshd_config  
echo "TCPKeepAlive yes" >> /etc/ssh/sshd_config  
systemctl restart sshd.service  
* 重启机器  
sync  
reboot  

## kubeadm安装  
* 安装docker  
yum install -y docker  
systemctl enable docker && systemctl start docker  
* 开启linux内核内部路由功能  
cat <<EOF >  /etc/sysctl.d/k8s.conf  
net.bridge.bridge-nf-call-ip6tables = 1  
net.bridge.bridge-nf-call-iptables = 1  
EOF  
sysctl --system  
* 添加repo  
由于国外的源被墙了，所以改为用国内的源，阿里的同步较快  
添加/etc/yum.repo.d/kubernetes.repo文件，内容如下：  
[kubernetes]  
name=Kubernetes  
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64  
enabled=1  
gpgcheck=0  
* 安装kube相关组件  
所有的机器均需要安装  
yum install -y kubelet kubeadm kubectl  
systemctl enable kubelet && systemctl start kubelet  
## 初始化master节点  
* 准备需要使用的镜像  
由于初始化时会创建各类型的服务容器，容器镜像是从gcr.io获取的，该网站是google旗下的镜像站，国内屏蔽。幸好有大神在同步这个容器镜像库，并传递到docker.io，方便使用，我们需要先下载下来备用，使用如下脚本即可下载镜像，同时把名称改为gcr.io的镜像名，骗过安装脚本。  
```
#!/bin/bash
images=(kube-proxy-amd64:v1.9.6 kube-scheduler-amd64:v1.9.6 kube-controller-manager-amd64:v1.9.6 kube-apiserver-amd64:v1.9.6 etcd-amd64:3.1.11 pause-amd64:3.0 kubernetes-dashboard-amd64:v1.8.3 k8s-dns-sidecar-amd64:1.14.7 k8s-dns-kube-dns-amd64:1.14.7 k8s-dns-dnsmasq-nanny-amd64:1.14.7)
for imageName in ${images[@]} ; do
  docker pull mirrorgooglecontainers/$imageName
  docker tag mirrorgooglecontainers/$imageName gcr.io/google_containers/$imageName
  docker rmi mirrorgooglecontainers/$imageName
done  
```  
* 初始化init  
最简单的初始化即直接在master节点执行kubeadm init，后面可以加各类参数。在这里我们指定一下版本和pod的IP池大小。  
kubeadm init --kubernetes-version=v1.9.0 --pod-network-cidr=10.200.0.0/16  
有其他参数需求请参考kubeadm官方文档  
过程中kubeadm执行了一系列的操作，包括一些pre-check，生成ca证书，安装etcd和其它控制组件等。  
稍等几分钟，当出现最下面的这行kubeadm join时，初始化就算结束了。kubeadm是用来让别的node加入集群的，请保存好这一行内容，这是我们之后让node加入集群的凭据，一会儿会用到。  
kubeadm join --token d08863.609af1bd1056ded5 192.168.122.139:6443 --discovery-token-ca-cert-hash sha256:9da6de47157f738a255f3288d243626a0ba5cfb1a414ce727796aec69f854b6e  
这一行只有24小时试用期，请注意时间。  
初始化过程中如果报错，或者失败，可以使用kubeadm reset重置环境。后续想重新安装也可以使用。  
* 使能kubectl  
kubectl是使用kubernetes的命令行工具，但不是默认安装完即可用，需要配置。  
export KUBECONFIG=/etc/kubernetes/admin.conf  
* 安装网络组件  
kubernetes需要安装网络插件来使能网络功能，社区有不少推荐，flanel,calico,等等，这里选择官方推荐的flannel。执行如下命令，自动安装。  
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/v0.9.1/Documentation/kube-flannel.yml    
等待几分钟，若安装顺利，则执行kubectl get pods --all-namespaces可以看到所有pods是running状态，kubectl get nodes里master是ready状态。  

