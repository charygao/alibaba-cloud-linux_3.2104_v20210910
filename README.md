# alibaba-cloud-linux_3.2104_v20210910
#### 故障描述
- 测试：在新购的阿里云ECS服务器上安装k8s单机集群，在对```Redis```单节点进行压力测试的时候，系统负载发生异常偏高情况。
- 故障：系统负载异常偏高，```sar -q 1 60```跟踪发现计数高达几十亿，严重偏离合理范围。输入命令执行卡死，随后只能强制重启，重启后依然卡死，最后只能回滚快照。
- 说明：实际上只是压测的时候能100%复现问题，其他在容器启动或者运行过程中，也会出现系统负载异常偏高，只不过相比压测的时候少点。
#### 故障猜测
- 内核版本```5.10.60-9.al8.x86_64 # v2021-09-10```：在处理k8s容器方面存在内核级bug，不能及时```ACK```导致系统负载异常
- 内核版本```5.10.23-4.al8.x86_64 # v2021-04-25```：正常
#### 系统环境

```Alibaba Cloud Linux 3```镜像发布记录：https://help.aliyun.com/document_detail/212634.htm?spm=a2c4g.11186623.0.0.17e82bf7MmbbFz#concept-2070819

准备1台ECS服务器```4核>8G```预安装```Alibaba Cloud Linux  3.2104 64位```，内核版本如下：
```bash
root@master01: ~ 22:48:56
# uname -a
Linux master01 5.10.60-9.al8.x86_64 #1 SMP Fri Apr 23 16:56:08 CST 2021 x86_64 x86_64 x86_64 GNU/Linux
```
安装阿里云主机插件
```bash
# 有的主机可以选择不预选安装，这里为了保证测试环境一致，如果没有安装就手动安装一下。
ARGUS_VERSION=3.5.3 /bin/bash -c "$(curl -s https://cms-agent-cn-hangzhou.oss-cn-hangzhou-internal.aliyuncs.com/Argus/agent_install_ecs-1.2.sh)"
```
创建```noroot```账号```k8s```（名字随便取），具体授权步骤等省略...
#### 安装 k8s 实验环境
- 安装```docker```，切换到 ```k8s```执行：
```bash
# Step.1
sudo yum install -y yum-utils

# Step.2
sudo yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

# Step.3
sudo yum install docker-ce-19.03.15-3.el8 docker-ce-cli-19.03.15-3.el8 containerd.io

# Step.4
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://ckec7rg9.mirror.aliyuncs.com"],
  "exec-opts": ["native.cgroupdriver=systemd"]
}
EOF

# Step.5
sudo systemctl daemon-reload # 加载配置文件
sudo systemctl restart docker # 重启Docker
sudo systemctl enable docker # 开机启动

# Step.6
sudo gpasswd -a $USER docker  #将当前用户添加至docker用户组
newgrp docker                 #更新docker用户组
```
- 安装```kubernetes```，切换到```root```执行：
```bash
# Step.1 关闭防火墙
systemctl stop firewalld && systemctl disable firewalld

# Step.2 将桥接的IPv4流量传递到iptables的链
cat > /etc/sysctl.d/k8s.conf << EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system  # 生效

# Step.3 修改主机名称
hostnamectl set-hostname master01
cat >> /etc/hosts << EOF
172.16.51.97 master01
EOF

# Step.4 配置镜像源
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

# Step.5 安装指定版本
yum install -y kubeadm-1.20.11 kubelet-1.20.11 kubectl-1.20.11

# Step.6 开机重启&启动
systemctl enable kubelet && systemctl start kubelet

# Step.7 创建10-kubeadm.conf文件，并编辑内容
mkdir /etc/systemd/system/kubelet.service.d
vim /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
################################################################################
[Service]
Environment="KUBELET_KUBECONFIG_ARGS=--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf"
Environment="KUBELET_CONFIG_ARGS=--config=/var/lib/kubelet/config.yaml"
# This is a file that "kubeadm init" and "kubeadm join" generates at runtime, populating the KUBELET_KUBEADM_ARGS variable dynamically
EnvironmentFile=-/var/lib/kubelet/kubeadm-flags.env
# This is a file that the user can use for overrides of the kubelet args as a last resort. Preferably, the user should use
# the .NodeRegistration.KubeletExtraArgs object in the configuration files instead. KUBELET_EXTRA_ARGS should be sourced from this file.
EnvironmentFile=-/etc/default/kubelet
ExecStart=
ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS $KUBELET_KUBEADM_ARGS $KUBELET_EXTRA_ARGS
################################################################################

# Step.8 初始化单机集群，172.16.51.97换成内网IP地址，其他不变
kubeadm init \
--apiserver-advertise-address=172.16.51.97 \
--image-repository registry.aliyuncs.com/google_containers \
--kubernetes-version v1.22.3 \
--service-cidr=10.96.0.0/12 \
--pod-network-cidr=10.244.0.0/16

# Step.9 初始化结束，执行
echo 'export KUBECONFIG=/etc/kubernetes/admin.conf' >> ~/.bash_profile
source ~/.bash_profile

# Step.10 单机部署，允许Master部署Pods
kubectl describe node master01 | grep Taints
kubectl taint nodes master01 node-role.kubernetes.io/master-

# Step.11
# 分别找到以下两个文件中的```--port=0```，注释掉即可。
vim /etc/kubernetes/manifests/kube-controller-manager.yaml
vim /etc/kubernetes/manifests/kube-scheduler.yaml

# Step.12 检查集群健康状态
kubectl get cs

# Step.13 安装 Calico:v3.21.0 网络插件
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml

# Step.14 至此，安装完毕。查看节点信息（正常节点都会显示```running```，如果没有，稍微等下。超过10分钟还是有问题，说明安装失败）
kubectl get pods --all-namespaces -o wide
```
- 安装```nfs```，切换到```root```执行：
```bash
# 检查 nfs 是否存在
rpm -qa | grep nfs

# 安装 nfs-utils
yum -y install nfs-utils

# 创建共享目录
mkdir -p /opt/nfs_data

# 修改权限
chmod -R 777 /opt/nfs_data

# 修改配置
vim /etc/exports

# 内容
/opt/nfs_data *(rw,no_root_squash,sync)

# 生效
exportfs -r
```
- 安装```nfs-subdir-external-provisioner```，切换到```root```执行：
```bash
## rbac.yml
kubectl create -f https://hy-raw.oss-cn-beijing.aliyuncs.com/k8s_docs/nfs-subdir-external-provisioner_v4.0.2/rbac.yaml

## deployment.yml
kubectl create -f https://hy-raw.oss-cn-beijing.aliyuncs.com/k8s_docs/nfs-subdir-external-provisioner_v4.0.2/deployment.yaml

## class.yml
kubectl create -f https://hy-raw.oss-cn-beijing.aliyuncs.com/k8s_docs/nfs-subdir-external-provisioner_v4.0.2/class.yaml
````
- 安装```redis-standalone```，切换到```root```执行：
```bash
# config
kubectl create -f https://hy-raw.oss-cn-beijing.aliyuncs.com/k8s_docs/redis_6.2.6/standalone/redis-standalone-config.yaml

# claim
kubectl create -f https://hy-raw.oss-cn-beijing.aliyuncs.com/k8s_docs/redis_6.2.6/standalone/redis-standalone-claim.yaml

# deployment
kubectl create -f https://hy-raw.oss-cn-beijing.aliyuncs.com/k8s_docs/redis_6.2.6/standalone/redis-standalone-deployment.yaml
```
#### 故障复现
- 使用内核版本```5.10.60-9.al8.x86_64 # v2021-09-10```

```bash
# 找到 redis节点容器，类似以 k8s_POD_redis-standalone-*** 开头的容器 name
docker ps

# 找到对应容器ID，进入容器内部
docker exec -it 775c7c9ee1e1 /bin/bash

# 执行压力测试，用100个线程发送10万个请求：
redis-benchmark -h 127.0.0.1 -p 6379 -a 123?456 -c 100 -n 100000
```
执行压力测试，几乎同时系统负载开始异常，压测结束系统负载异常依旧（服务器不操作等待几个小时以后，系统负载居高不下。只能强制重启！），系统重启后容器初始化过程也有可能发生异常（随机），这个不能100%复现，但是压力测试能100%复现。

- 回退内核版本```5.10.23-4.al8.x86_64 # v2021-04-25```

再次执行压力测试，系统没有出现负载异常。
