---
title: "本地虚拟机搭建kubenates集群"
date: 2022-03-05T14:50:34+08:00
draft: false
---

# 概述
今天跟着[k8s easydoc](k8s.easydoc.net)在本地利用虚拟机搭建了k8s集群。
# 创建虚拟机
这里用的是centos7, 使用VMware创建了三台2核2GB的虚拟机，这里不介绍怎么安装了，可以跟着[菜鸟教程VMware安装CentOS 7教程](https://www.runoob.com/w3cnote/vmware-install-centos7.html)来做。创建一台master，其余两个node节点，可以使用VMware的克隆功能来简化创建操作。

安装完毕之后需要注意的是这里安装的`CentOS 7`，使用`ip addr`来查看虚拟机的ip。

# 配置虚拟机
输入`ip addr`命令，`inet`后面为该机器的内网地址，我的三台机器ip情况为
|ip|name|
|--|----|
|192.168.2.110|master|
|192.168.2.111|node1|
|192.168.2.112|node2|
## 为每台机器设置hostname
`hostname set-hostname master`
`hostname set-hostname node1`
`hostname set-hostname node2`
设置之后可以通过`hostname`命令查看是否设置成功。

## 设置hosts
`vi /etc/hosts`
在结尾插入
```
192.168.2.110 master
192.168.2.111 node1
192.168.2.112 node2
```
`ESC` `:wp`保存退出

## 关闭SELinux [all]
```
setenforce 0
sed -i --follow-symlinks 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux
```

## 关闭防火墙 [all]
```
systemctl stop firewalld
systemctl disable firewalld
```

## 添加安装源 [all]

### 添加 k8s 安装源
```
cat <<EOF > kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
mv kubernetes.repo /etc/yum.repos.d/
```
### 添加 docker 安装源
```
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```

## 安装所需组件 [all]
```
yum install -y kubelet kubeadm kubectl docker-ce
```

## 启动`kubelet` `docker`，并设置开机启动 [all]
```
systemctl enable kubelet
systemctl start kubelet
systemctl enable docker
systemctl start docker
```

## 修改`docker`配置 [all]
```
# kubernetes 官方推荐 docker 等使用 systemd 作为 cgroupdriver，否则 kubelet 启动不了
cat <<EOF > daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "registry-mirrors": ["https://ud6340vz.mirror.aliyuncs.com"]
}
EOF
mv daemon.json /etc/docker/

# 重启生效
systemctl daemon-reload
systemctl restart docker
```

## 使用`kubeadm`初始化集群 [master]
这里是在本地创建的需要先执行`swapoff -a`命令，不然会导致连接timeout。
如果初始化失败，需要执行`kubeadm reset`重置。
```
# 初始化集群控制台 Control plane
kubeadm init --image-repository=registry.aliyuncs.com/google_containers

# 记得把 kubeadm join xxx 保存起来
# 忘记了重新获取：kubeadm token create --print-join-command

# 复制授权文件，以便 kubectl 可以有权限访问集群
# 如果你其他节点需要访问集群，需要从主节点复制这个文件过去其他节点
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config

# 在其他机器上创建 ~/.kube/config 文件也能通过 kubectl 访问到集群
```
初始化成功之后，会输出
```
kubeadm join 192.168.2.110:6443 --token kmw4tn.glpiwvubfj00lw82 --discovery-token-ca-cert-hash sha256:5d64208226bc40d6250ffe175b8e519e6037ffd49406d6a36cf278364c845d9c
```
上面是工作节点连接主节点使用的。

## 连接主节点 [node]
```
kubeadm join 192.168.2.110:6443 --token kmw4tn.glpiwvubfj00lw82 --discovery-token-ca-cert-hash sha256:5d64208226bc40d6250ffe175b8e519e6037ffd49406d6a36cf278364c845d9c
```

## 安装插件，使`STATUS`字段正常显示 [master]
```
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

可以通过`kubectl get nodes`查看节点信息。
```shell
[root@master ~]# kubectl get nodes
NAME     STATUS   ROLES                  AGE   VERSION
master   Ready    control-plane,master   48m   v1.23.4
node1    Ready    <none>                 45m   v1.23.4
node2    Ready    <none>                 45m   v1.23.4
```