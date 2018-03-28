# 安装步骤  
## 准备工作  
* 安装必要的辅助工具，比如vim等  
* 如下操作均在root用户下，若不使用root用户，请使用具有sudo权限的用户  
* 关闭selinux  
vim /etc/sysconfig/selinux  
找到SELINUX=enforcing，改成SELINUX=disabled  
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
