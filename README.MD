## 使用 Kubeadm 搭建单 Master 集群

使用 kubeadm 初始化 kubernetes 集群。操作系统为 CentOS 7.6.1810 X86_64, 用到的各相关程序版本如下：
- kubeadm：v1.18.4
- docker：19.03.8

### 准备环境

**1. 主机名解析**

```
~]# cat >> /etc/hosts << EOF
192.168.124.30 master master.linux.io
192.168.124.31 node1  node1.linux.io
192.168.124.32 node2  node2.linux.io
EOF
```

**2. 关闭防火墙和 selinux**

```
~]# for i in stop disable ;do systemctl $i firewalld; done
~]# setenforce 0 && sed -i "s/^SELINUX=enforcing/SELINUX=disabled/g" /etc/selinux/config
```

**3. 禁用swap分区**

```
~]# swapoff -a && sudo sed -i 's/.*swap.*/#&/' /etc/fstab
```

**4. 安装依赖**

```
~]# mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.backup
~]# curl -o /etc/yum.repos.d/CentOS-Base.repo https://mirrors.aliyun.com/repo/Centos-7.repo
~]# sed -i -e '/mirrors.cloud.aliyuncs.com/d' -e '/mirrors.aliyuncs.com/d' /etc/yum.repos.d/CentOS-Base.repo
~]# yum install -y epel-release lrzsz ntpdate wget conntrack ipvsadm ipset jq iptables curl sysstat net-tools bind-utils vim
```

**5. 调整内核参数**

```
~]# modprobe overlay
~]# modprobe br_netfilter
~]# cat > /etc/sysctl.d/99-kubernetes-cri.conf <<EOF
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
~]# sysctl  --system
```

**6. 时间同步**

```
~]# cat  > /etc/chrony.conf << EOF
server ntp.aliyun.com iburst
stratumweight 0
driftfile /var/lib/chrony/drift
rtcsync
makestep 10 3
bindcmdaddress 127.0.0.1
bindcmdaddress ::1
keyfile /etc/chrony.keys
commandkey 1
generatecommandkey
logchange 0.5
logdir /var/log/chrony
EOF

~]# for i in enable start; do systemctl $i chronyd.service; done
~]# chronyc sources
```

#### 升级内核版本(4.x)

**1. 升级内核版本为(4.4.227)**

```
~]# rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
~]# rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-3.el7.elrepo.noarch.rpm
~]# yum --enablerepo=elrepo-kernel install  kernel-lt
~]# yum update
```

**2. 设置grub2**

```
~]# awk -F\' '$1=="menuentry " {print i++ " : " $2}' /etc/grub2.cfg
~]# grub2-set-default "CentOS Linux (4.4.227-1.el7.elrepo.x86_64) 7 (Core)"
~]# reboot
```

**3. 验证**

```
~]# uname -r
4.4.227-1.el7.elrepo.x86_64
```

#### Docker 部分

**1. 安装必要的一些系统工具**

```
~]# yum install -y yum-utils device-mapper-persistent-data lvm2
```

**2. 添加软件源信息**

```
~]# yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```

**3. 安装指定版本docker**

```
~]# yum install -y  docker-ce-19.03.8
~]# systemctl  start docker && systemctl enable docker
```

**4.Docker 镜像加速**

```
~]# cat  > /etc/docker/daemon.json << EOF
{
  "registry-mirrors": ["https://o4uba187.mirror.aliyuncs.com"],
  "exec-opts": ["native.cgroupdriver=systemd"]
}
EOF
~]# systemctl  restart docker
```

### Kubeadm 部分

**1. 配置 Kubeadm 仓库**

```
~]# cat  > /etc/yum.repos.d/kubernetes.repo << EOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```

**2. 安装kubeadm tools**

```
~]# yum install -y  kubeadm-1.18.4 kubelet-1.18.4 kubectl-1.18.4
~]# systemctl enable kubelet
```

**3. kubeadm init 初始化**

```
kubeadm init --kubernetes-version=v1.18.4 \
             --apiserver-advertise-address=192.168.124.30 \
             --pod-network-cidr=10.244.0.0/16   \
             --service-cidr=10.96.0.0/12 \
             --image-repository=registry.cn-hangzhou.aliyuncs.com/google_containers \
             --ignore-preflight-errors=Swap | tee kubeadm-init.log
```

**4. kubeadm join 添加node**

```
kubeadm join 192.168.124.30:6443 --token rqr74q.8aonbtlpikc0f97c \
    --discovery-token-ca-cert-hash sha256:13864b03a6ac6204bb1c3e64a95284f01c568bd2c5d99e48744376edc4e51109
```
```
~]# kubectl  get nodes
NAME     STATUS     ROLES    AGE     VERSION
master   NotReady   master   5m56s   v1.18.4
node1    NotReady   <none>   118s    v1.18.4
node2    NotReady   <none>   78s     v1.18.4
```

**5. 部署 flannel 网络插件**

```
~]# kubectl apply -f kube-flannel.yml
```
```
~]# kubectl  get nodes
NAME     STATUS   ROLES    AGE   VERSION
master   Ready    master   29m   v1.18.4
node1    Ready    <none>   25m   v1.18.4
node2    Ready    <none>   24m   v1.18.4
```

**6. 修改成 ipvs 模式**

```
~]# vim /etc/sysconfig/modules/ipvs.modules
#!/bin/bash
ipvs_mods_dir="/usr/lib/modules/$(uname -r)/kernel/net/netfilter/ipvs"
for mod in $(ls $ipvs_mods_dir | grep -o "^[^.]*"); do
    /sbin/modinfo -F filename $mod  &> /dev/null
    if [ $? -eq 0 ]; then
        /sbin/modprobe $mod
    fi
done

~]# chmod  +x /etc/sysconfig/modules/ipvs.modules
~]# bash -x /etc/sysconfig/modules/ipvs.modules
```

```
~]# kubectl  edit cm kube-proxy -n kube-system
    mode: "ipvs" # 默认为iptables

~]# kubectl delete pods -l k8s-app=kube-proxy -n kube-system
```
