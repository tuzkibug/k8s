# 安装步骤  
## 准备工作  
* 安装必要的辅助工具，比如vim等
* 关闭selinux  
vim /etc/sysconfig/selinux  
找到SELINUX=enforcing，改成SELINUX=disabled  
* 检查swap  
kubernetes必须工作在无swap环境，默认安装系统时分区不要分swap分区，如果分了，请手动关闭该分区  
swapon -s检查是否有分区，假如有使用swapoff /dev/xxx来关闭分区  
