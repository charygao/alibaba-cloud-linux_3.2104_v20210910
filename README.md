# alibaba-cloud-linux_3.2104_v20210910
#### 故障描述
- 测试：在新购的阿里云ECS服务器上安装k8s单机集群，在对```Redis```单节点进行压力测试的时候，系统负载发生异常偏高情况。
- 故障：系统负载异常偏高，```sar -q 1 60```跟踪发现计数高达几十亿，严重偏离合理范围。输入命令执行卡死，随后只能强制重启，重启后依然卡死，最后只能回滚快照。
- 说明：实际上只是压测的时候能100%复现问题，其他在容器启动或者运行过程中，也会出现系统负载异常偏高，只不过相比压测的时候少点。

#### 系统环境

```Alibaba Cloud Linux 3```镜像发布记录：https://help.aliyun.com/document_detail/212634.htm?spm=a2c4g.11186623.0.0.17e82bf7MmbbFz#concept-2070819

新购服务器```4核32G```预安装```Alibaba Cloud Linux  3.2104 64位```，内核版本如下：
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
- 安装```docker```
切换到 ```k8s```，执行
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
- 安装```kubernetes```
- 安装```k8s -> caclio```
- 安装```k8s -> redis```
#### 故障复现
